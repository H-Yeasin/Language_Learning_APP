# Building an Interactive 3D Avatar Companion in Flutter: A Deep Dive into Hybrid Integration

In modern language learning applications, engagement is key. Text and audio are standard, but having a visual, interactive companion can significantly boost user immersion. In our latest project, we implemented a 3D avatar system that not only displays a character but also lip-syncs to speech and performs animations like waving. 

This article walks through how we integrated 3D avatars into our Flutter app using a **hybrid approach**—leveraging **Native Swift (SceneKit)** for iOS and **WebView (model-viewer)** for Android—to ensure optimal performance and compatibility.

## The Challenge

We needed a solution that could:
1.  Render high-quality 3D models (**USDZ for iOS**, **GLB for Android**).
2.  Support animations (Idle, Wave, Talk).
3.  Perform real-time lip-syncing based on text-to-speech output.
4.  Work seamlessly across different operating systems without sacrificing performance.

## The Solution: A Hybrid Rendering Architecture

To achieve the best of both worlds, we adopted a platform-specific rendering strategy:

### 1. iOS: Native SceneKit Implementation
On iOS, we bypassed Flutter's standard rendering and went straight to the metal using **SceneKit**. This ensures the avatar looks crisp, responds instantly, and integrates perfectly with the Apple ecosystem using `.usdz` models.

**Key Native Components (`ios/Runner`):**
*   **`AppDelegate.swift`**: Registers the `AvatarView` plugin on app launch.
*   **`AvatarViewFactory.swift`**: Acts as the builder that Flutter calls to create the native view instance.
*   **`AvatarPlatformView.swift`**: The brain of the operation. It uses an `SCNView` to render the 3D scene, manages the camera to focus on the avatar's face, and handles `MethodChannel` calls from Flutter to trigger animations.

### 2. Android: Web-Based Implementation
For Android (and potentially Web), we utilize Google's powerful `<model-viewer>` web component inside a `WebView`. This allows us to easily render `.glb` (glTF binary) models with built-in support for animations and lighting.

**Key Components:**
*   **flutter_assets**: Stores the `.glb` models.
*   **WebView**: Loads a local HTML file that initializes the `model-viewer` script.
*   **JavaScript Bridge**: Exposes functions like `playAnimation()` and `setMorphTarget()` that our Dart code can call.

---

## Implementation Details

### The `AvatarView` Widget (Dart)

The core usage in Flutter is simple. The `AvatarView` widget checks the current platform and builds the appropriate underlying view.

```dart
class AvatarView extends StatefulWidget {
  final AvatarController? controller;
  final String? avatarName;
  
  @override
  Widget build(BuildContext context) {
    // ✅ iOS - Uses Native UiKitView (SceneKit)
    if (defaultTargetPlatform == TargetPlatform.iOS) {
      return UiKitView(
        viewType: 'AvatarView', // Matches the ID in AppDelegate.swift
        onPlatformViewCreated: _onPlatformViewCreated,
        creationParams: {
          'avatarName': widget.avatarName, // e.g., "Karl" or "Clara"
        },
        creationParamsCodec: const StandardMessageCodec(),
      );
    }

    // ✅ Android - Uses WebView with <model-viewer>
    if (defaultTargetPlatform == TargetPlatform.android) {
      return Container(
        child: WebViewWidget(controller: _webViewController),
      );
    }
    
    return const SizedBox();
  }
}
```

### The `AvatarController`: Bridging Dart, Swift, and JS

To control the avatar (make it wave or talk), we created a unified `AvatarController`. It abstracts the underlying platform complexity so the rest of the app doesn't need to know *how* the avatar works, just that it *does*.

```dart
class AvatarController {
  // ...
  
  Future<void> triggerHandWave({double duration = 0.25}) async {
    if (_channel != null) {
      // 🍎 iOS: Calls Native Method in AvatarPlatformView.swift
      // This triggers a CAAnimation on the SCNNode
      await _channel?.invokeMethod('triggerHandWave', {'duration': duration});
    } else {
      // 🤖 Android: Calls JavaScript in the WebView
      // This triggers the model-viewer's animation logic
      _webViewController?.runJavaScript("window.playAnimation('Wave', $duration);");
    }
  }
}
```

## Real-Time Lip Syncing

One of the most complex features is lip-syncing. We align the avatar's mouth movements with the Text-to-Speech (TTS) engine by mapping phonemes (speech sounds) to "visemes" (facial morph targets).

*   **On Android**: We pass the viseme data to JavaScript, which updates the `morph-target-influences` of the 3D model frame-by-frame.
*   **On iOS**: The `AvatarPlatformView.swift` receives the viseme list and uses a `VisemeScheduler` (linked to `CADisplayLink`) to smoothly interpolate the facial geometry using SceneKit's morphers.

## Conclusion

By combining the flexibility of Web technologies for Android with the raw performance of Native Swift for iOS, we created a fluid, interactive avatar system. This hybrid approach allows us to:
1.  **Optimize Performance**: Native iOS rendering is far more efficient than loading a heavy web engine on older iPhones.
2.  **Ensure Compatibility**: Android's fragmentation is handled gracefully by the robust `model-viewer` component.
3.  **Deliver Delight**: The result is a learning partner that feels alive, actively listening and responding to the user.
