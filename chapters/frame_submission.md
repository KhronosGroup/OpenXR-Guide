<!--
Copyright 2021-2022 The Khronos Group, Inc.
SPDX-License-Identifier: CC-BY-4.0
-->

# Frame Submission

OpenXR frame timing and frame submission is implemented through three functions: `xrWaitFrame`, `xrBeginFrame` and `xrEndFrame`.

`xrWaitFrame` does two operations in one call:
1. The function blocks until the XR runtime determines the best time to continue to produce the next frame.
2. The function returns some information about the next frame to be submitted. The most important is the `predictedDisplayTime` timestamp which is the time the frame is expected to be shown on the display.

`xrBeginFrame` marks the start of the GPU rendering work. GPU work that contributes to rendering that is submitted to the OpenXR runtime should happen after this call.

`xrEndFrame` submits the rendered layers to the runtime.

Unlike many other VR/XR APIs, there is no explicit frame identifier concept. While many of the legacy or proprietary APIs may use a frame id, frame index, or frame object to make frame operations explicit, OpenXR uses a different approach, consisting of implicit frame(s) in flight and display timestamps.

## Call order restrictions

Because there is no explicit frame identifier, OpenXR instead uses a concept of implicit frames, of which there can be up to two in progress with some restrictions. Support for more than one frame in flight is useful for 3D engines which are pipelined (meaning they are working on multiple frames in parallel). This will be discussed more later in the "Common frame submission patterns" section. A frame can be in one of two states: pre-rendering and rendering. After `xrWaitFrame` returns, there is a new implicit frame that is active in the pre-rendering stage. 3D engines often do their logic "tick" during this period (e.g., game logic, physics calculations, etc.). After `xrBeginFrame`, the frame is moved from pre-rendering state to a rendering state. 3D engines are expected to queue their GPU work when in this state. A key restriction is that there can only be one frame active in the pre-rendering and rendering state. This is required for the app and runtime to have a common understanding of which frame is being submitted since there is no explicit frame identifier used.

NOTE: `predictedDisplayTime` may look like it could double as a frame identifier. After all, it is guaranteed to be a monotonically increasing integer. However, it is not guaranteed to be used by the application in any other function directly. For example, the application use `predictedDisplayTime + predictedDisplayPeriod` to work on one extra frame in advance and submit this on `xrEndFrame`.

In order for the runtime to track which frame is being submitted, the OpenXR runtime will block in `xrWaitFrame` if there is a frame active in the pre-rendering state (`xrBeginFrame` is called). See `xrWaitFrame` deadlock) in the Pitfalls section to see where this often leads to trouble.

## Threading

All three frame functions can be called from any thread, however the app must ensure the following:
1. `xrWaitFrame` calls are not overlapping. This should not be a problem because engines typically call `xrWaitFrame` from the same thread each time.
2. `xrBeginFrame` and `xrEndFrame` calls are not overlapping. Again, this typically is not a problem because it is common to call `xrBeginFrame` and `xrEndFrame` on the same thread.
3. `xrBeginFrame` and `xrEndFrame` calls are properly protected from concurrent graphics API queue access for graphics APIs without queue thread-safety like Vulkan. This means the app must not be queuing work on the same queue as the one provided to the runtime while `xrBeginFrame` and `xrEndFrame` are being called if Vulkan is being used.

## Common frame submission patterns

The frame submission APIs are flexible enough to support many different patterns with various trade-offs.

NOTE: Error checking is not included in the code snippets but is always encouraged.

### Single-Threaded 1

This is the simplest frame loop where all operations run on the same thread. For simple applications where `MyGameLogic` and `MyGameRendering` are brief, this single-threaded approach is reasonable.

Pros:
* Simplicity

Cons:
* The amount of time spent in `MyAppLogic` adds to the latency of the frame in flight.
* For the application to maintain full framerate, it must keep the total time spent working on a frame (from `xrWaitFrame` returning to calling `xrEndFrame`) under the native frame duration. For example, a device that renders at 60fps allows the application at most 1/60 seconds (16.66ms) to submit each frame.

```c
void SimpleRenderLoop() {
  while (!quitting) {
    XrFrameState frameState{XR_TYPE_FRAME_STATE};
    XrFrameWaitInfo waitInfo{XR_TYPE_FRAME_WAIT_INFO};
    xrWaitFrame(session, &waitInfo, &frameState);
    
    // Handle input, physics, and other logic.
    MyAppLogic();
    
    XrFrameBeginInfo beginInfo{XR_TYPE_FRAME_BEGIN_INFO};
    xrBeginFrame(session, &beginInfo);
    
    if (frameState.ShouldRender) {
      MyAppRendering();
    }
    
    XrFrameEndInfo endInfo{XR_TYPE_FRAME_END_INFO};
    endInfo.displayTime = frameState.predictedDisplayTime;
    // Layer(s) and other data populated in endFrame excluded for brevity.
    xrEndFrame(session, &endInfo);
  }
}
```

### Multithreading

The benefit of multithreaded frame submission is that two frames can be worked on simultaneously so that more total time can be spent on a frame while maintaining full frame rate. For example, if a device runs at 60fps then the app only has 16.66ms to submit a frame. If game logic takes 12ms and rendering takes another 8ms, then a single-threaded app will not be able to make full frame rate because it will be spending 20ms total. However, if MyGameLogic and MyGameRendering run concurrently then the app will not have a problem. This is called pipelined rendering and is common in most 3D engines.

## Time

`predictedDisplayTime` returned by `xrWaitFrame` is the time the frame is predicted it will be displayed. Sometimes this is called photon time or mid-photon time as it should be the middle of the period that the display is illuminating the frame. This is called a prediction because runtimes may determine at a later time a different photon time, or the app may end up being too slow submitting the frame that it ends up being displayed at a different time. When this happens, it can result in judder and so runtimes will try to avoid it. `predictedDisplayPeriod` is also returned by xrWaitFrame. This is the amount of time predicted until the next frame (the next `predictedDisplayTime` returned by `xrWaitFrame`) will be presented. This is useful for some pipelined engines when they need a time to run a simulation of the next frame while simultaneously rendering the current frame. A common request is to know the frame rate, but this may change depending on a multitude of factors, and `predictedDisplayPeriod` is the closest equivalent. If the `predictedDisplayPeriod` is 11.11ms then the frame rate is 1/0.0111 or 90 FPS.

### Adjusting the simulation clock

For each frame, the application may need to progress a simulation forward and there are multiple ways this can be done. One naive method is to track the time of the previous frame simulation and simulate for the delta for the new frame. This will not give terrible results either. However, this excludes any changes in timing for the current frame that may occur. A better method is to advance a simulation based on the delta of the current `predictedDisplayTime` and the previous `predictedDisplayTime`. This can make an important difference when the runtime decides to postpone the next frame and will ensure animations remain smooth.

## Poses

`xrLocateViews` and `xrLocateSpace` are used to locate things in the world, often for rendering. Most importantly is the head (`VIEW`) pose, but this also includes hands, eyes and more. A pose is always located at a specific time, and it is ideal to use the time that the frame will be displayed for. This ensures the runtime is providing the right amount of extrapolation into the future to best predict the pose that the space will be in at the time it is shown. There are rare cases where a timestamp other than the display time should be used, but these are the exception to the rule.

Another important note is that locating a pose at a timestamp will result in the best guess at that point in time. Locating a pose at the same timestamp in the future may result in a different pose with potentially less latency which results in higher accuracy. More accurate poses will provide a more immersive feeling and for the head pose will result in a reduced chance for motion sickness. This is also why `xrEndFrame` requires the poses for each composition projection layer view--because the runtime has no way to know which pose was actually used to render with.

## Spaces

XrSpaces are the core primitive used in OpenXR to represent a location in the world. Some spaces may represent a relatively fixed place in the world (i.e., world-locked) such as `LOCAL` space, `STAGE` space or an anchor. Other spaces represent dynamic places in the world such as `VIEW` space, eyes, and hands. When composition layers are submitted to the runtime, all poses are relative to some XrSpace. A common pattern is to locate the `VIEW`s relative to `LOCAL` space, render for those poses, and then submit the composition projection layers to the runtime using the same poses relative to the `LOCAL` space. This space is the space the runtime will reproject (stabilize) relative to. This means for content to be stable in the world, the projection layer needs to be provided relative to a world-locked space like `LOCAL` space.

One not uncommon mistake that is made is that applications will provide their projection layer relative to a dynamic space like `VIEW` space. This can be tempting because an application can locate the VIEWs relative to `VIEW` space (this effectively gives the eye offsets) and then submit the rendered layer using these poses relative to `VIEW` space. The effect though is that the runtime should reproject/stabilize the rendering relative to `VIEW` space. This is almost always undesirable except in the case that the application wants content to be "head locked". An example of where this might be desirable is if the application wants to provide a layer that is head-locked, like for a heads-up display projection or quad layer.

## Reducing latency

As mentioned in the [Poses](#poses) section, reducing latency results in more accurate pose predictions which improves the user experience. There are two common methods used to do this:

### Late Update

Late update is when an application will locate the head pose a second time right before issuing the draw calls. Applications will use an early pose during simulation which can be used for interaction or culling purposes, but then "late update" the head pose right before rendering.

### Late Latch

Late latch takes the late update idea even further. The application will locate the head pose a second time right before queueing the render work. This removes the additional latency of the draw calls.

## Pitfalls

### xrWaitFrame deadlock

Earlier it was discussed that `xrWaitFrame` will block until `xrBeginFrame` is called for the previous frame. This is a very strict requirement because the call order is necessary to track which frame is being presented. A consequence is that if `xrWaitFrame` is called but there is no corresponding `xrBeginFrame` call, the application will become locked on the next `xrWaitFrame` call indefinitely. This may happen due to app logic deciding to skip rendering a frame or because an error was encountered. In more complex engines, the XR component is often a plugin that gets called by the core engine, and there may be subtle conditions that cause a call into the XR plugin to be skipped. So, care must be taken here.

The rule is very simple though: For every call to `xrWaitFrame`, the app MUST make a matching call to `xrBeginFrame`.

Note though that `xrEndFrame` is NOT critical to keeping the frame loop running and does not play a part in this common pitfall. This is because if `xrBeginFrame` is called without a previous call to `xrEndFrame`, it is an implicit discard of the previous frame.

### Rendering for the wrong pose

An application should use `xrLocateViews`, sometimes in conjunction with `xrLocateSpace`, to get the eye poses to render for. And when the app submits the rendered frame, it must give the same poses that it used when rendering back to the runtime along with projection layer details. This allows the XR system to reproject the content in the same location in the world. A mistake here can sometimes be subtle, especially in VR where the physical world is not available as a reference point, but would still contribute to motion sickness. This will manifest as "swimming" where the world-locked content appears to move in the world, proportional to the amount of head movement. A mistake here is obvious with a see-through AR solution though.

The most common reason that this bug happens is the application is pipelined and it accidentally uses the wrong pose when submitting the frame. For example, it may render using a pose for frame N-1 but submit the rendered frame using the pose for frame N. The inverse problem can also happen, where the app renders for the wrong pose but submits the rendering using the correct pose.

NOTE: Some runtimes even provide a stressing mechanism for developers to test for this kind of application bug by jittering the head poses randomly. If the app is rendering for the same pose that it submits with `xrEndFrame`, the XR system will be able to compensate by reprojecting the content in the correct place in the world with just black pulled in on the sides. But if there is a bug, it will result in very chaotic and unstable reprojection because the app is instructing the XR system to reproject the content to the wrong pose.
