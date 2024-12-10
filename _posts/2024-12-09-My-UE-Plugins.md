---
layout: post
title:  "Unreal Engine: My Plugins"
date:   2024-12-09 23:22:00 +1100
permalink: /posts/UE/My-UE-Plugins
---

I have created a few plugins for Unreal Engine and intend to create more in the future, as time goes on I will list them here with a brief description and link to their respective repositories.




### [Object Pooling Plugin](https://github.com/itsBaffled/BFObjectPooling)
Object pooling is all about reusing spawned objects/actors and not destroying and spawning them over and over again. There are many times where pooling objects is preferred over spawning them, for example in a bullet hell game where you have hundreds of bullets on screen at once, it's much more efficient to pool them and reuse them (Although these would probably be niagara meshes), another example is enemies, sounds, vfx, decals, etc. <br>
The plugin has a quite extensive list of built in poolable actor types which can be found [here](https://github.com/itsBaffled/BFObjectPooling/tree/main/BFObjectPooling/Source/BFObjectPooling/GameplayActors) but just to name a few
- Sound
- Niagara VFX
- Decals
- Static / Skeletal Meshes
- Widgets
More

The pool does **NOT** require a subsystem which was a massive goal of mine when creating it, I wanted it to be as reusable and easy to integrate as possible, It was important that I could allow (*almost*) any object to own the pool and manage it, nothing stops you from putting a pool on a subsystem though if desired. <br>


### [Squirrel Noise Plugin](https://github.com/itsBaffled/BFSquirrel)
This plugin is not only a wrapper for Squirrel noise "http://eiserloh.net/noise/SquirrelNoise5.hpp" (which if you didn't know is a very fast and efficient pseudo random number generator that uses noise and deterministic seeds and indexes, [this](https://www.youtube.com/watch?v=LWFzPP8ZbdU) is definitely worth a watch if you have the time) but also comes with an integration with Unreal and a custom K2 node implementation that makes the editor UI for using the noise super clean (See the repo for more on this). <br>

There are many reasons to prefer this over unreals base `FMath::Rand` and `FRandomStream` if you really care about randomness. <br>













