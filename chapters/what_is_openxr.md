<!--
Copyright 2021-2022 The Khronos Group, Inc.
SPDX-License-Identifier: CC-BY-4.0
-->

# What is OpenXR?

OpenXR is a widely adopted open standard for XR: augmented reality, virtual
reality, and everything in between. It specifies the interface between XR
applications and runtimes which may handle frame composition, peripheral
management, and more. The Khronos Group is a member-driven consortium that
created and maintains OpenXR and other standards like Vulkan, OpenGL, glTF, etc.

## What does the standard provide?

The core of OpenXR is its [_API specification_][openxr-spec]. It describes in
detail the procedures and structures an OpenXR _runtime_ should expose or,
similarly, the ones an OpenXR _application_ could use. Here _should_ and _could_
being just natural language words with no precise definition, as opposed to
[their usage][spec-verbs] in the spec.

[openxr-spec]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html
[spec-verbs]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html#introduction-terminology

OpenXR does not include a runtime implementation. Particular runtimes tend to be
provided by third parties like, for example, device manufacturers. For a list of
publicly available runtimes see the OpenXR [landing page][openxr-runtimes].

[openxr-runtimes]: https://www.khronos.org/openxr#openxr_runtimes

Other resources provided by the standard are listed [below](#openxr-resources).

## What is in an OpenXR runtime?

A runtime is a system that exposes the OpenXR API and satisfies its
specification. It is called a runtime and not a _driver_ because it has a bigger
scope than regular drivers. The latter usually provides low-level abstractions
to interface directly with the hardware. Runtimes, on the other hand, often
implement functionality beyond that. Examples would be things like a home
environment to launch XR applications from, a compositor that allows multiple
applications to be run simultaneously or even overlaid on top of each other, and
visualization of tracking boundaries, among others.

## What functionality does the API expose?

In broad terms, it provides standardized access to the resources of the XR
hardware. Next is a list of examples of the kind of input-output you can expect
when interacting with the OpenXR API.

- Receiving the position and orientation (i.e., the [_pose_][`XrPosef`]) of
  controllers and other tracked entities ([`XrSpace`]).
- Receiving pose, field of view, and projection and view matrices for each
  display ([`XrView`]).
- Receiving information about the state of buttons, triggers, touchpads,
  thumbsticks, etc ([`XrAction`]).
- Sending textures to present to the displays.
- Sending haptic feedback to controllers.

It is a good idea to read the [fundamentals] chapter of the spec to better
understand the concepts behind the API.

[`XrView`]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html#view-projections
[`XrSpace`]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html#XrSpace
[`XrAction`]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html#XrAction
[`XrPosef`]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html#XrPosef
[fundamentals]: https://www.khronos.org/registry/OpenXR/specs/1.0-khr/html/xrspec.html#fundamentals

## Why OpenXR?

Before OpenXR, vendors designed and implemented their own APIs. XR applications
that wanted to run on multiple XR devices needed to support each vendor API
separately. In many cases, you could develop on top of a game engine, but then
it was up to it to support each interface. With OpenXR, vendors expose
functionality to a common interface, the [OpenXR API][openxr-headers] which is
then consumed by the XR application or game engine. In this way, the same
application codebase can run on multiple OpenXR-capable devices. Furthermore,
the standard allows vendors to implement
[_extensions_][openxr-extension-proposal] for supporting special features that
are not covered in the base specification.

[openxr-headers]: https://github.com/KhronosGroup/OpenXR-SDK/tree/main/include/openxr
[openxr-extension-proposal]: https://www.khronos.org/registry/OpenXR/specs/1.0/styleguide.html

## The OpenXR registry and other resources {#openxr-resources}

A good starting point for the standard is the [registry][openxr-registry]. This
is an index containing both explanations and links to the different parts that
constitute OpenXR. It includes links to the [specification][openxr-spec], the
[API reference pages][openxr-apiref], the handy [reference
guide][openxr-refguide] which has a nice overview of the OpenXR lifecycle, the
[_loader_ design document][openxr-loader] which is in charge of selecting
runtimes, and others.

[openxr-apiref]: https://www.khronos.org/registry/OpenXR/specs/1.0/man/html/openxr.html
[openxr-registry]: https://www.khronos.org/registry/OpenXR/
[openxr-refguide]: https://www.khronos.org/registry/OpenXR/specs/1.0/refguide/openxr-10-reference-guide.pdf
[openxr-loader]: https://www.khronos.org/registry/OpenXR/specs/1.0/loader.html

The registry also contains references to the OpenXR repositories published by
Khronos on [GitHub][khronos-openxr-github]. You can find the API headers on the
[OpenXR-SDK] repository. The alternative [OpenXR-SDK-Source] repository
additionally includes source code for the loader or the [`hello_xr`] sample
application. You might also want to check out the [OpenXR-Hpp] bindings for
using OpenXR through idiomatic C++.

[khronos-openxr-github]: https://github.com/orgs/KhronosGroup/repositories?q=openxr
[OpenXR-SDK]: https://github.com/KhronosGroup/OpenXR-SDK
[OpenXR-SDK-Source]: https://github.com/KhronosGroup/OpenXR-SDK-Source
[OpenXR-Hpp]: https://github.com/KhronosGroup/OpenXR-Hpp
[`hello_xr`]: https://github.com/KhronosGroup/OpenXR-SDK-Source/tree/main/src/tests/hello_xr

You can join the [KhronosDevs Slack][khronos-slack] workspace and the [community
forums][khronos-discourse] if you need help or just want to discuss Khronos and
OpenXR-related topics.

[khronos-slack]: https://khr.io/slack
[khronos-discourse]: https://community.khronos.org/
