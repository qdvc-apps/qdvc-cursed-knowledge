> **Outcome (2026-11-19):** New entry "On iPad in landscape, the camera preview moves at right angles to the camera" in a new section, Swift, SwiftUI and UIKit (iOS and iPadOS).

# On iPad in landscape, an `AVCaptureVideoPreviewLayer` moves at right angles to the camera

- **Date encountered:** 2026-11-15
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** New section suggested: "Swift, SwiftUI and UIKit (iOS and iPadOS)" (the existing Swift sections are about macOS)
- **Versions:** iPadOS 26 deployment target; Xcode version and iPad model not recorded
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed
- **Related entry:** None

## Symptom

A camera preview (a QR code scanner, hosted in SwiftUI through a `UIViewControllerRepresentable`) worked in portrait. On an iPad in landscape, the picture didn't move in the same direction as the camera: panning the iPad moved the image at right angles, as if the preview were turned 90°.

## Cause

The best explanation from the code and Apple's API; not measured. An `AVCaptureVideoPreviewLayer` shows frames in the camera sensor's own orientation unless its connection is told to rotate them. Nothing in the code updated that as the interface rotated, so in landscape the preview was drawn a quarter-turn out. The older way to handle this, setting `connection.videoOrientation` from the interface orientation, is deprecated since iOS 17.

## Fix

Use `AVCaptureDevice.RotationCoordinator(device:previewLayer:)` (iOS 17 and later). Observe its `videoRotationAngleForHorizonLevelPreview` with key-value observation, and set the preview connection's `videoRotationAngle` to it, after checking `isVideoRotationAngleSupported`. Apply it once at setup and again in `viewDidLayoutSubviews` and `viewWillTransition(to:with:)`. Keep the layer's `frame` equal to the view's bounds on every layout. Not yet confirmed on an iPad.

## Watch for

- Testing only on an iPhone in portrait hides this completely: iPhone apps are often portrait-only, iPad apps never are.
- The same applies to any custom camera view (document scanning, video capture), not just QR scanners.
- Code samples that set `videoOrientation` predate the rotation coordinator. They still compile, with a deprecation warning.

## Minimal example

```swift
final class ScannerController: UIViewController {
    private let session = AVCaptureSession()
    private var preview: AVCaptureVideoPreviewLayer?
    private var rotation: AVCaptureDevice.RotationCoordinator?
    private var observation: NSKeyValueObservation?

    override func viewDidLoad() {
        super.viewDidLoad()
        guard let camera = AVCaptureDevice.default(for: .video),
              let input = try? AVCaptureDeviceInput(device: camera) else { return }
        session.addInput(input)
        let layer = AVCaptureVideoPreviewLayer(session: session)
        view.layer.addSublayer(layer)
        preview = layer
        let coordinator = AVCaptureDevice.RotationCoordinator(device: camera, previewLayer: layer)
        rotation = coordinator
        observation = coordinator.observe(\.videoRotationAngleForHorizonLevelPreview) { [weak self] _, _ in
            DispatchQueue.main.async { self?.level() }
        }
        level()
    }

    private func level() {
        guard let rotation, let connection = preview?.connection else { return }
        let angle = rotation.videoRotationAngleForHorizonLevelPreview
        if connection.isVideoRotationAngleSupported(angle) { connection.videoRotationAngle = angle }
    }

    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        preview?.frame = view.bounds
        level()
    }
}
```
