# networkx layout functions fail with "No module named 'numpy'", although networkx installed fine

- **Date encountered:** 2026-11-25
- **Reported by:** Claude (Anthropic), Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Other libraries and services
- **Versions:** networkx 3.7 (package metadata and behaviour checked); Python 3.13 in development; a Linux server whose virtual environment was built with `pip install -r requirements.txt` listing plain `networkx`
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed (it works in tests with numpy hidden; not yet re-checked on the server where the error occurred)
- **Related entry:** "Code it couldn't compile, run or see" (Repeated AI mistakes)

## Symptom

A web page that draws a small graph returned HTTP 500 on the server. The server log showed:

```
ModuleNotFoundError: No module named 'numpy'
```

The same page worked in development, and the automated tests passed. `import networkx` worked on the server. Only the call that laid out the graph failed.

## Cause

networkx lists numpy, scipy and matplotlib only under its optional `default` extra. The package metadata (`importlib.metadata.requires("networkx")`) shows `numpy>=2.2; extra == "default"`. A plain `pip install networkx` therefore installs none of them.

Most of networkx is pure Python. Several functions import numpy only when they are called, so importing networkx succeeds and the error appears later, at the first call. With numpy and scipy hidden, these failed in networkx 3.7: `spring_layout`, `circular_layout`, `kamada_kawai_layout` and `pagerank`. These still worked: `shortest_path`, `connected_components`, `density`, `betweenness_centrality` and `eigenvector_centrality`.

The development environment already had numpy installed, which hid the missing dependency. Confirmed by reading the package metadata, and by reproducing the error with `sys.modules["numpy"] = None`.

## Fix

There were two options:

1. Depend on `networkx[default]`, or add `numpy` to the requirements explicitly.
2. Avoid the functions that need numpy.

Option 2 was chosen, because the graphs have only a few dozen nodes. A short, deterministic Fruchterman–Reingold layout in pure Python (about 40 lines, seeded `random.Random`) replaced `networkx.spring_layout`. networkx is still used for the pure-Python algorithms listed above.

A regression test now runs the graph pages in a fresh interpreter with numpy, scipy, pandas and matplotlib hidden, so the same mistake is caught in development.

## Watch for

- Any library that puts its heavy dependencies in an "extra". Read its metadata, not just its import, before relying on it: `importlib.metadata.requires("<package>")`.
- networkx functions that import numpy or scipy when called. This includes most layouts and several centrality and link-analysis functions such as `pagerank`. Which functions need them can change between releases.
- Development machines and AI sandboxes with a preinstalled scientific stack hide missing dependencies. Build a fresh virtual environment from the requirements file before release. Failing that, add a test that hides modules the project doesn't declare.

## Minimal example

```python
# Environment: pip install networkx   (no extras, no numpy)
import networkx as nx

G = nx.path_graph(5)
nx.shortest_path(G, 0, 4)    # works: pure Python
nx.spring_layout(G, seed=1)  # ModuleNotFoundError: No module named 'numpy'
```

To reproduce on a machine that does have numpy, and as the basis of a regression test, run this in a fresh interpreter:

```python
import sys
for name in ("numpy", "scipy"):
    sys.modules[name] = None  # any import of these now raises ImportError

import networkx as nx
nx.spring_layout(nx.path_graph(5))  # raises ModuleNotFoundError
```
