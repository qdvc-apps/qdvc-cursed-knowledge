> **Outcome (2026-11-19):** New entry "Werkzeug over TLS logs a client hanging up as `ssl.SSLError`" in Other libraries (section renamed to include Werkzeug). The entry links back here for the code.

# Werkzeug logs a client hanging up over TLS as `ssl.SSLError: [SYS] unknown error`

- **Date encountered:** 2026-11-15
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Other libraries (Werkzeug, the development server behind Flask)
- **Versions:** Python 3.9 (a Linux distribution's own, on the server), Flask 3.x with Werkzeug 3.x (exact versions not recorded), the distribution's OpenSSL 3; the fix was checked on the sandbox's newer Python
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed in a local reproduction; not confirmed on the original server
- **Related entry:** None

## Symptom

The Flask app is served over HTTPS by Werkzeug's own server (`run_simple` with an SSL context). While a large file was being sent, the server log showed this as an "Error on request" traceback:

```
Traceback (most recent call last):
  File ".venv/lib64/python3.9/site-packages/werkzeug/serving.py", line 374, in run_wsgi
  execute(self.server.app)
  File ".venv/lib64/python3.9/site-packages/werkzeug/serving.py", line 338, in execute
  write(data)
  File ".venv/lib64/python3.9/site-packages/werkzeug/serving.py", line 313, in write
  self.wfile.write(data)
  File "/usr/lib64/python3.9/socketserver.py", line 826, in write
  self._sock.sendall(b)
  File "/usr/lib64/python3.9/ssl.py", line 1240, in sendall
  v = self.send(byte_view[count:])
  File "/usr/lib64/python3.9/ssl.py", line 1209, in send
  return self._sslobj.write(data)
ssl.SSLError: [SYS] unknown error (_ssl.c:2501)
```

It looked like a server crash, and it was a red herring: the real problem was elsewhere.

## Cause

Confirmed by reproduction: a real TLS server sending a large file to a client that hung up part-way. Browsers routinely abandon a large response part-way: they read the start of a file, then hang up and ask for a different byte range (when seeking, say). Over plain HTTP that raises a `ConnectionError`, which Werkzeug's request handler treats as a dropped connection and doesn't log. Over TLS the same hang-up can surface as a bare `ssl.SSLError` instead ("[SYS] unknown error", "bad length", and so on, depending on the Python and OpenSSL versions). `ssl.SSLError` is an `OSError` but not a `ConnectionError`, so Werkzeug logs a full traceback. After a failed write, Werkzeug also drains the request body, and over TLS that read raises a second SSL error ("unexpected eof while reading"). In the reproduction, Werkzeug's own handler logged 4 tracebacks for 4 hang-ups.

## Fix

A `WSGIRequestHandler` subclass wraps the connection's `rfile` and `wfile` and turns any `ssl.SSLError` raised while reading or writing into a `ConnectionResetError`, so Werkzeug handles it like any other dropped connection. Both directions have to be wrapped, because of the drain after a failed write. In the reproduction that logged no tracebacks for the same 4 hang-ups, and ordinary range requests and form posts still worked. Pass the class to `run_simple(..., request_handler=…)`.

## Watch for

- Don't chase the traceback: on a TLS server it usually just means a client went away. Look for whatever else went wrong.
- It shows up with large responses, and with clients that make range requests.
- Werkzeug's development server isn't meant for production; if an app does use it over TLS, this fix keeps its log readable.

## Minimal example

```python
import ssl
from werkzeug.serving import WSGIRequestHandler, run_simple

class _HangUpSafe:
    def __init__(self, raw): self._raw = raw
    def _call(self, name, *args):
        try:
            return getattr(self._raw, name)(*args)
        except ssl.SSLError as exc:
            raise ConnectionResetError(f"client hung up ({exc})") from exc
    def write(self, data): return self._call("write", data)
    def flush(self): return self._call("flush")
    def read(self, *a): return self._call("read", *a)
    def read1(self, *a): return self._call("read1", *a)
    def readinto(self, b): return self._call("readinto", b)
    def readline(self, *a): return self._call("readline", *a)
    def __iter__(self): return iter(self.readline, b"")
    def __getattr__(self, name): return getattr(self._raw, name)

class HangUpSafeHandler(WSGIRequestHandler):
    def setup(self):
        super().setup()
        self.rfile = _HangUpSafe(self.rfile)
        self.wfile = _HangUpSafe(self.wfile)

# run_simple(host, port, app, ssl_context=(cert, key), request_handler=HangUpSafeHandler)
```
