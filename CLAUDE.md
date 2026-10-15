# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
flutter pub get

# Analyze / lint
flutter analyze

# Run the example app
cd example && flutter run

# Run tests (no tests currently exist)
flutter test
```

## Architecture

This is a Flutter package (`cropme`) that provides an Apple Photos-style image cropper widget with spring physics, rotation, and free-form resize handles.

### Public API (`lib/cropme.dart`)

Exports four things:
- `ImageCropper` — the main widget / full-screen dialog entry point
- `CropSettings` / `CropResult` — serializable state and result models
- `CropHandle` — enum for which resize handle is active
- `CropperThemeData` — full visual customization

### Core widget (`lib/src/image_cropper.dart`)

`_ImageCropperState` owns all physics and gesture logic. Key design points:

- **Gesture stack**: A `Listener` (pointer count) wraps an `InteractiveViewer` (pan/zoom). `onInteractionUpdate` intercepts scale events to extract two-finger rotation delta on top of the built-in pan/zoom.
- **Four spring configs** are defined as module-level constants (`_kAppleSpringDescription`, `_kBounceBackSpring`, `_kLightSpringDescription`, `_kCenteringSpring`). Each `AnimationController.unbounded` is driven by a `SpringSimulation` from `package:flutter/physics.dart`.
- **Boundary math** works in rotated (u, v) coordinates. `_calculateBoundaryViolationForMatrix` projects the image center offset onto the crop rect's local axes, computes safe margins accounting for rotation, and returns a world-space correction vector. This is how the image stays fully within the crop rect even when rotated.
- **Minimum scale** is computed by `_calculateMinScaleForRotation`, which finds the smallest scale at which the image covers the crop rect at a given angle. A 0.5% safety margin is added.
- **Crop output**: `_cropImage` uses `PictureRecorder` → `Canvas`, applies the transformation matrix and rotation, and produces PNG `Uint8List`. Output is capped at 3000 px on either side.
- **HDR/HEIF normalization**: On non-web platforms, `FlutterImageCompress.compressWithList` converts the source bytes to JPEG at 95% quality before decoding, so iOS 10-bit HDR images load correctly.
- **`CropSettings` restore**: When `initialSettings` is provided, `_fitImageToRect` reconstructs the exact `Matrix4` (scale + translation) and crop rect, bypassing the default fit-to-rect calculation.

### Supporting widgets (`lib/src/`)

| File | Responsibility |
|---|---|
| `cropper_resize_handles.dart` | 8 drag handles (4 corners + 4 edges) with rubber-band resistance near screen edges; haptic feedback on drag start/end |
| `cropper_rotation_trackbar.dart` | Horizontal drag trackbar drawn with `CustomPainter`; haptic feedback at 0°, ±15°, ±30°, ±45° |
| `cropper_overlay_painter.dart` | Draws rule-of-thirds grid lines inside the crop rect |
| `cropper_inverted_clipper.dart` | `CustomClipper` that cuts a rounded-rect hole for the crop area (used by the blur backdrop) |
| `cropper_app_bar.dart` | Top bar with close icon and Reset button |
| `cropper_bottom_controls.dart` | Rotation trackbar, zoom-in button, and Crop & Save button |
| `cropper_glass_button.dart` | Reusable frosted-glass-style button |
| `crop_settings.dart` | `CropSettings` (serializable via `toMap`/`fromMap`) and `CropResult` |
| `cropper_theme.dart` | `CropperThemeData` with `const` defaults for a dark-themed cropper |

### Layout constants (hardcoded in multiple files)

`topPadding = 100.0`, `bottomPadding = 210.0`, `horizontalPadding = 15.0` define the safe area for the crop rect. These appear in `_calculateCropRect`, `_animateCropRectToCenter`, and `CropperResizeHandles._applyResistanceToRect` — keep them in sync if changed.
