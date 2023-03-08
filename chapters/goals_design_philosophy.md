<!--
Copyright 2023 The Khronos Group, Inc.
SPDX-License-Identifier: CC-BY-4.0
-->

## Working group goals for Application Developers

1. We want you to be able to customize how your software works on the devices
   you are focused on and testing most, and we want it to work the way you
   expect/intend on those devices by default.
2. We want your software to work (without modification) on hardware/software you
   have not tested, as well as hardware that has not been invented yet.
3. We want your software to depend only on the specified, documented behavior,
   so that goal 2 is possible, and so that compatibility with your software does
   not constrain the design space of future XR systems.
4. We want to avoid writing compatibility code in runtimes to work around a bug
   in your software (or alternately, having to choose to simply let your
   software be unusable).
5. As a consequence of 4, we want to avoid encouraging or relying on untested
   code paths in your software: hardware you have not tested should use the same
   code paths that you use for hardware you did test.
6. As a consequence of 4 and corollary of 5, we want to avoid "surprising" your
   software with behavior permitted by the spec but not tested in your
   development process.
7. We want the common tasks to be as easy as possible without constraining
   future potential, so the natural choice is the right one or the best practice
   in most cases. (Pit of success)
8. We want complicated and uncommon things to be possible, but not at the
   expense of making the common ones more error-prone. If it is possible for a
   given rare, complicated use to be implemented with the specification as-is,
   we are unlikely to add to the spec to make it easier if it might pose a trap
   for the more common use cases.

## Why interaction profiles?

Goal number 1 is the reason that interaction profiles (that correspond to real
hardware) exist. Suggested bindings for interaction profiles allow you to
describe how interaction should work on a controller you have actually tested.
Ideally, most of the conditional logic that depends on the controller in use
would actually be described (statically/declaratively) in the suggested
bindings.

## Why is it the current interaction profile may not match the actual hardware?

Because of 5 and 6, we do not want to return unexpected data, so the only
interaction profiles you will ever get back from the runtime as being active,
are the ones for which you have suggested bindings. By your software suggesting
bindings for an interaction profile, you are declaring to runtimes, "I know
about this controller/device and I test my software regularly with it." If the
spec allowed runtimes to return any interaction profile as current, your
software might receive a value you had never tested with, and the code handling
that would more likely to contain bugs than the code you did test.

If a controller/device is in use that you did not suggest bindings for, the
runtime will make a best effort to map what it actually has, to one of the
bindings you suggested, and report that as the active interaction profile
instead. This keeps your software running in the tested code paths, and it
allows such compatibility mappings to work across multiple applications, instead
of needing to be added as per-app compatibility exceptions to normal behavior.

## How do I find out which controller is in use?

So if you want to know what controller is in use, check which interaction
profile is active. **If knowing about a specific controller matters to you, test
with it and suggest bindings for it.** If there is not an interaction profile
defined for it, reach out to the vendor so they can register one. Other
techniques for detecting hardware/controllers are likely to be very fragile and
make errors and consequently make everyone's job harder by requiring explicit,
specific app compatibility code in every runtime if they want to be able to run
your software.

### What if I want to show the controller?

If you want to provide a rendered version of the controller: use the current
interaction profile to pick which of your assets to use. If you do not need a
customized model, you may consider using the extensions for controller models
provided by some runtimes. We are working on a more standardized and uniform way
of accessing them in the future.

### For some other reason?

If you want to know what controller/hardware is in use for a different reason,
please contact the OpenXR Working Group by filing an issue at
<https://github.com/KhronosGroup/OpenXR-Docs/> or posting in the forums at
<https://community.khronos.org/c/openxr/25>. We can help you figure out the best
way to meet the underlying need in a way that keeps your software robust, and
can help us learn if there is a use case we did not cover with existing spec and
extensions. You should not need a big "switch" statement or complicated hardware
detection code: if you have that, it will probably be brittle and hard to
maintain, and there is probably a better way of achieving your objective.
Remember goals 7 and 8: if you are having to do something strange and obscure,
you might be going about things in an unexpected or discouraged way.

## How can I determine what hardware my app is running on? (Why don't you make this part easy?)

You should try to avoid using any hardware-detection, vendor ID, or system
name/ID features you may find, if at all possible. They increase the likelihood
of following an untested code path, they are brittle (being unspecified or only
weakly specified), and they make it less likely that your software will run
reliably across different systems, and more likely that your software will
either fail to run or require specific compatibility workarounds in every single
XR runtime.

We are trying to work around the core observation in
[Hyrum's Law](https://www.hyrumslaw.com/) - in brief, that any observable
behavior of a system will be depended upon - in service of goal 2, while
balancing it with goal 1. The interaction profile and action system is the key
of our solution here. We are trying to limit the observable behavior to things
that are clearly described and specified by OpenXR, and that are tested by you
prior to shipping your software. This is why there is very little open-ended
"get system information" functionality in OpenXR. It increases the likelihood of
untested code paths (what happens when a function you've seen return 1 or 2
someday returns 3?). It also ties your software more closely to a single OpenXR
runtime implementation, constraining current and future compatibility, as well
as constraining the design space of future XR hardware and software. At some
point, for compatibility reasons, at least one runtime would need to "lie" in
such a "system information" API, so we worked (and continue to work) hard to
design an interface where your software describes the environments it was tested
with and the high-level details of its interaction, and where the runtime can
make the best possible match and mapping of the current environment to the
tested ones.

## But how can I change the interaction mappings?

We intended that all runtimes, or at least general-purpose runtimes not
specifically constrained, would expose a remapping/rebinding UI to users, or at
least a file-based system for customizing bindings so that third-party UIs could
exist. Such an interface would allow users of your software to customize this
interaction mapping not only when they use controllers you did not provide
suggestions for, but also for alternate usage preference or access needs on
devices you did test. This is why the call refers to "suggested" bindings: the
user is intended to have final control over how they interact.
