# WebXR Raw Camera Access

## Overview

Currently, to protect user privacy, WebXR Device API does not provide a way to grant raw camera access to the sites. Additionally, alternative ways for sites to obtain raw camera access (getUserMedia() web API) are not going to provide the application pose-synchronized camera images. For some scenarios, this limitation may pose a significant barrier to adoption of WebXR Device API. This is especially true given the fact that addition of new APIs takes time, thus stifling experimentation and prototyping that could happen purely in JavaScript.

Given the above, it will be beneficial to provide the sites access to raw camera images through WebXR API. It does come with a potential risk related to the fact that the sites will be granted access to the images from the user's environment. To mitigate privacy aspects of providing such capability, implementers should ensure that appropriate user consent is collected before granting camera access to the site.

The work in this repository describes experiments that should fall under the scope of immersive-web CG's [Computer Vision](https://github.com/immersive-web/computer-vision) repository.

## Use cases

Granting the camera access to the application could allow the applications to:
- Take a snapshot of the AR experience along with application-rendered content. This is mostly relevant for scenarios where the camera image serves as the background for application's content (for example handheld AR).
- Provide feedback to the user about the environment they are located in. This may be particularly useful for VR headsets with available cameras (for example WMR headsets and the Flashlight feature).
- Run custom computer vision algorithms on the data obtained from the camera texture. It may for example enable applications to semantically annotate regions of the image, for example to provide features related to accessibility.


## Proposed API shape

The raw camera image access API should seamlessly integrate with APIs already exposed by the WebXR Device API. The Web IDL for proposed API could look roughly as follows:

```webidl
partial interface XRWebGLBinding {
  WebGLTexture? getCameraImage(XRFrame frame, XRView view);
};

partial interface XRViewerPose {
  [SameObject] readonly attribute FrozenArray<XRView> cameraViews;
};

dictionary XRCameraAccessStateInit {
  bool enabled;
};

partial interface XRSession {
  void updateCameraAccessState(XRCameraAccessStateInit state);
};
```

This allows us to provide a time-indexed texture containing a camera image that is retrievable only when the XRFrame is considered active. The API should also be gated by the `“cameraAccess”` feature descriptor.

## Using the API

The applications could leverage the newly introduced feature as follows:

1. Create a session that supports accessing camera images:
```javascript
navigator.xr.requestSession(“immersive-ar”, { requiredFeatures: [“cameraAccess”]}).then(...);
```

If UA decides it needs to prompt the user for permission to use the camera, it can do so at this stage.

2. Once the session is created, the application can configure it to enable the feature:
```javascript
session.updateCameraAccessState({enabled: true});
```

From this point onward, assuming that the session was created with the camera access feature listed in either requested or optional features and the user granted permissions, the `cameraViews` array will become populated with views that can subsequently be used to query the camera image texture. If the session configuration happened during a requestAnimationFrame callback, the changes will take effect no earlier than for subsequent requestAnimationFrame (but they can take longer than that).

3. In requestAnimationFrame callback, the application can query for the camera view for which it’s interested in accessing the camera image:

```javascript
// ... in rAFcb ...
let viewerPose = xrFrame.getViewerPose(xrRefSpace);
let cameraView = viewerPose.cameraViews[0];
```

4. Using the camera view, the application can then query the gl binding for the camera image texture:
```javascript
// ... in rAFcb ...
let cameraTexture = binding.getCameraImage(xrFrame, cameraView);
```

Side note: GL binding referred to in step 4. is an interface that is newly introduced in the [WebXR Layers Module](https://immersive-web.github.io/layers/#XRWebGLBindingtype). It can be constructed given a XRSession and GL rendering context.