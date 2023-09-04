---
layout:     post
title:      Introduction to USD and Omniverse
summary:    An introduction to USD and Omniverse
categories: omniverse
tags:
 - usd
 - omniverse
---

To understand Omniverse, first we need to understand what USD is (since Omniverse is based on USD).

## What is USD exactly

USD stands for "Universal Scene Description" and at this point is an umbrella term that includes an open standard ([https://openusd.org/](https://openusd.org/)) to collaboratively edit and create 3D scenes and assets, apply physically correct properties to the 3D assets, describe the materials to apply, define the hierarchies (e.g. which asset goes under which other asset to create the 3D model of a car), define how properties vary with time (e.g. for an animation of a 3D model), etc.

It also describes a file format (.usd files) which defines how to save that information to a single or multiple files that can be shared with other users locally or across a network.

If a 3D content creator program like Blender or 3D Studio Max or Maya have a _USD connector_ available, it means the program is able to export its content (be it a 3D model, a texture, or even just plain text data) to a USD file. And that file can be later read and imported by another program which has a USD connector. Plus there are tools to visualize those USD files, to edit them, to compose them and so on.

USD is a very descriptive, complex and powerful 3D representation format, some of the key points:

* It was initially developed by Pixar and later open-sourced (free to use under [Apache 2.0](https://github.com/PixarAnimationStudios/USD/blob/release/LICENSE.txt)), so it's rather battle-tested for large-scale usage

* It is often addressed as the 'Photoshop of 3D graphics' since it uses layers to compose a final component of a scene (e.g. the color of an asset can be overridden in a stronger layer, but the other colors are retained as well - one can always go back to it or compose them together) and it allows for multiple people editing the same 3D scene collaboratively and/or in real-time (if the underlying software allows it). E.g. see this image where `layer2` (a USD file itself) which gives the cube a red color with a _stronger opinion_ is muted and the renderer transitions the cube to a blue color (given by `layer1` which has a _weaker_ opinion on the color given by the layers' hierarchy)

![Thumper](/assets/images/muting_layer.gif)

* It is **extensible**. USD allows for new _schemas_ to be defined: a schema is a textual document of rules. E.g. there could be a 'light schema' which defines light properties like the light's color, intensity and direction. The schema should afterwards be parsed and interpreted by a software (like Omniverse) which can later load a USD file with primitives (for now just think of 'primitives' as 3D assets) which have that schema applied to them (much like fulfilling an interface contract in programming languages or inheriting from a pure virtual class) and have the properties defined by that schema _defined_ and _instantiated_ into them. So to make it simple a software can read a 'LightBulb' prim (3D asset) which has the 'Light' schema applied on it, and therefore it _must_ have `color`, `intensity` and `direction` defined in order for the renderer to actually render it on the scene. This is the [real light schema](https://github.com/PixarAnimationStudios/USD/blob/release/pxr/usd/usdLux/schema.usda) in usda format (USD format in ASCII - i.e. human readable, not binary nor compressed), as you can see it is pretty complex but there is no need to grasp that complexity right now.

    Schemas are a powerful mechanism which allows the USD format to expand, evolve and represent complex 3D objects properties (e.g. **physics properties!**).


Pixar has also open-sourced with the same license a reference implementation of the USD standard, i.e. they provide a rather large C++ library hosted at [github.com/PixarAnimationStudios/USD](https://github.com/PixarAnimationStudios/USD) that the users can download, compile, contribute to, etc.. the library is written in C++ but python bindings are also available in that same repository to call USD APIs from Python programs.

It has to be noted that this library doesn't provide any renderer in itself, i.e. even if you clone the repository, build it, generate the python bindings and write code to generate a cube

{% highlight python %}
from pxr import Usd, UsdGeom

# Create a new stage
stage = Usd.Stage.CreateNew('helloWorld.usd')

# Define a new Cube primitive
cube = UsdGeom.Cube.Define(stage, '/HelloWorldCube')

# Set the size of the cube
cube.GetExtentAttr().Set([(1.0, 1.0, 1.0)])

# Save the stage
stage.GetRootLayer().Save()
{% endhighlight %}

you will have the opportunity to save this stage as a USD file, but you will not have any viewer to visually inspect and render that cube. USD defines another rather complex specification for a rendering architecture called _Hydra_ that renderer programmers can abide by to have their own renderer integrate with USD scenes. There is however a small tool called [usdView](https://docs.omniverse.nvidia.com/app_usdview/app_usdview/overview.html) in the same official USD repo based on PyQt that allows you to quickly render USD stages (mostly for debugging purposes and to understand how USD works).

## What is Omniverse

Omniverse is a platform and a series of technologies developed by NVIDIA around the USD standard (although they're rapidly evolving in other directions as well).

Omniverse comprises technologies and applications to work with 3D graphics, collaborating on creating 3D assets and scenes, using AI to create stunning visual effects or improve the process of creating 3D contents, adding real-time and physically correct physics behaviors to 3D contents, rendering in a physically-correct way with ray tracing or path tracing in real-time, etc.

**NVIDIA doesn’t impose any workflow or dictate how Omniverse tools and technologies should be used** (they can be used to create photorealistic render images that you later use commercially, they can be used to let multiple 3D artists work on a 3D scene simultaneously without interfering with each other’s modifications, they can be used to ‘predict’ the mechanical ‘wear’ in a ‘digital twin’ 3D representation of a mechanical part in 3D with accurate physics after many simulation steps, they can be used to create a server-side web service which renders something complex and streams the result as a video back to the user’s browser, etc.).

The foundation of many Omniverse applications is called _Kit_ and it's an extensible framework and SDK developed by NVIDIA which provides a series of libraries and APIs to let users write _extensions_ (i.e. kit-based libraries written in Python, C++, both or in other languages as well) so that they can use NVIDIA's best-in-breed technologies (e.g. RTX raytracers, PhysX, AI integrations, etc.) to do useful graphical work for them. [Official docs for Kit can be found here](https://docs.omniverse.nvidia.com/prod_kit/prod_kit/kit-architecture.html).

An example: a user can write a Kit extension ([here are lots of extensions used in Omniverse](https://docs.omniverse.nvidia.com/extensions/latest/index.html)) which acts as a web service: whenever a HTTP request is received, the extension can fire up a complex 3D scene, generate the assets or the scene requested via the [HTTP request with chatGPT](https://www.youtube.com/watch?v=mFazJsjUUSo), render it and send the rendered result back to the user.

## Composer & Presenter

Two of the most famous applications (and some of the very first ones a newcomer might try out) in Omniverse are [USD Composer (formerly Create)](https://www.nvidia.com/en-us/omniverse/apps/create/) and [USD Presenter (formerly View)](https://www.nvidia.com/en-us/omniverse/apps/view/).

The former is a 3D authoring program which allows users to compose complex scenes from 3D assets, applying physical properties to them, simulating and rendering, applying photorealistic materials and much more

[![Thumper](/assets/images/create_joints_authoring.png)](https://www.youtube.com/watch?v=3QjFjpUooXI)

Composer/Create is usually not equipped with 3D creation tools to model single 3D assets (think Blender) but rather orchestrates composition of a USD scene from external assets (although it could even become a modeling tool with the right extensions).

Presenter/View instead focuses on visualizing already composed environments and inspecting USD scenes (it doesn't feature advanced authoring tools as Composer).

There is some overlap in the UI elements of the two applications, as there is in some of the core extensions used for a simple reason: both Composer and Presenter are _just a bunch of highly complex Kit based extensions_. Kit is both a SDK that programmers can leverage to build their own extensions and also **a portable cross-platform executable** that can 'bootstrap' their extensions with a bare minimum skeleton environment. Composer and Presenter are both instances of the Kit platform executable but they load different extensions. These extensions can also be altered (and dependencies broken, if the user so desires..), changed or custom ones loaded. This is the beauty of Omniverse: you can compose it in any way you want. You're not satisfied with a particular workflow? Create your own extension and fire it up from a Kit CLI command or from the Composer UI panel.

## Nucleus

Omniverse isn't just a collection of Kit extensions though but much more. For instance: to foster collaboration between lots of 3D artists working together on a complex 3D scene (think Pixar..), together with the USD specification and file format (which features "photoshop layer-like behavior" for 3D contents) Omniverse also provides Nucleus which is a distributed object storage optimized for graphical assets.

With nucleus users can reference `omniverse://asset_paths` from external repositories or internal organization repos, use or modify those assets and seamlessly have them streamed back to other users which might be using them. Dynamically.

## Pricing and requirements

Before we take a look at some rather cool Omniverse possible uses in other posts (and again: just _some_ , **you** decide how Omniverse should be used and/or program an app or an extension for Omniverse yourself and let it do whatever you want for you), let’s first talk about two things newcomers usually care a lot about: pricing and requirements.

**Omniverse is free to use for individuals, but a license must be purchased for team use**: [Omniverse Licensing](https://www.nvidia.com/en-us/omniverse/download/).

More in detail (from the official discord):

> Omniverse users are welcome to sell their extensions for whatever they please. The end users must have a license of Omniverse to use their extensions with, but that can be the free Omniverse Individual version.

So programmers are free to write and sell their Omniverse extensions. Users can buy and use those extensions as long as they do it abiding by the Omniverse license (i.e. if they're working as a team of 20 people with Omniverse, an enterprise license must be purchased). If they're working as individuals (or teams of 2 people), no license is necessary and Omniverse is totally free.

What about 3D content I create with e.g. Omniverse Composer? Can I sell a rendered video of a Physical simulation made with Omniverse?

> Content and or code\extensions\apps created using OVI (Omniverse Individual license, i.e. abiding by the 2-users-tops requirement) for small teams, using desktops or cloud resources is allowed and can be used for commercial purposes.

So yes: you can create a video using Omniverse and you can sell it for whatever you want.

Can I use Omniverse in my own private cloud?
> For the free version, you are allowed to put Omniverse in the cloud for your own purposes. For example, you are allowed to put OV apps on Azure or AWS VM, create 3D projects and render out those projects using Omniverse Farm which can also be on an Azure VM for free.
> The EULA is designed that once you scale the number of users working together and you need support, you should get the enterprise license.
> Other licensing example, You can also use the OVI version to create, build, sell your own extensions and or apps for free. The user leveraging that extension or app just needs to follow the same EULA.

The only other restriction pertains to letting users use your "abiding OVI individual license" Omniverse apps as cloud services:

> Lets say for example, you put USD Composer in the cloud and allow anyone to use it for free as a streamed application. This would not be allowed, using OVI as a service to users outside your company.

For any other question or clarification please read the final paragraph of this post and get in contact with NVIDIA sales for a special license tailored to your needs: [omniverse-license-questions@nvidia.com](mailto:omniverse-license-questions@nvidia.com) will get you in touch with a developer relations manager that can work with you.

Regarding Omniverse requirements, each application that works on the Omniverse platform might have different system requirements. The suggested way to get up-to-date information is to browse the NVIDIA website for the app you’re specifically interested in, e.g. [the USD Composer/Create page](https://www.nvidia.com/en-us/omniverse/apps/create/) lists requirements like a RTX class card as minimum viable hardware.


## Support, learning, official resources

Official channels to learn more about Omniverse, post questions regarding its official applications and main extensions (e.g. related to omni.physx) and get in touch with the great NVIDIA Omniverse community (friendly and available, NVIDIA is doing its best to foster a good community) are the [Omniverse discord server](https://forums.developer.nvidia.com/t/omniverse-discord-server-is-live/178422), the [YouTube Omniverse channel](https://www.youtube.com/c/nvidiaomniverse), the [developer blog articles](https://developer.nvidia.com/blog/tag/omniverse/) and, for critical bugs/issues, the [official Omniverse forum](https://forums.developer.nvidia.com/c/omniverse/) (less chatty, more support-y).

Any non-Omniverse related question should not be asked in the above channels but rather in the [NVIDIA customer support forum](https://www.nvidia.com/en-us/support/).

Happy Omnivers'ing!