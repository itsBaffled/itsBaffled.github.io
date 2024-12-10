---
layout: post
title:  "Unreal Engine: Creating A Plugin Or Module"
date:   2024-12-09 20:10:00 +1100
permalink: /posts/UE/Creating-A-Plugin-Or-Module
---

## Introduction
When working in Unreal Engine, sometimes you need to create a plugin or module to extend the engine or add functionality. This can be for a number of reasons like keeping code/assets isolated, sharing code/assets between projects, or even just to keep things organized.


There are PLENTY of resources on how to create a plugin or module for Unreal Engine but I wanted to create a simple guide that I can link to if need be so I can save time in other posts.
<br>


### Module vs Plugin
At a high level there isn't really a "Module vs Plugin", a plugin is simply a collection of modules and assets (there are also content only plugins that supply no code and just .uasset files but we won't be covering that here as it's simple if you already understand regular plugins).<br>

So, when it comes to choosing between creating a brand new plugin or adding a module to an existing one, it comes down to what you need. If you are creating a module that is very specific to your project and has no real use outside of it then it makes sense to add it to your project as a module, if you are creating a module that could be useful in other projects or is a standalone feature then it makes sense to create a plugin for it. <br>

Let's imagine we were making a runtime building/crafting system, I could see great benefit to this being it's own plugin for future reusability and ease of sharing between projects so I would definitely opt to create a plugin for this, where as lets say I was creating a specific enemy type for my game, I might opt to still create a separate module for all enemy logic just to keep it out of the core of the game code but I wouldn't see a need to create a plugin for it.
<br>
<br>



#### Linking Issues
Linking symbols between modules is one of the biggest issues I have seen newer UE devs face, this may be due to lack of experience or just the fact that dll export/imports are abstracted into your `MYMODULENAME_API` macro.
<br> 
If you are unfamiliar with how normal DLLs and static libraries work then I suggest you take the time to quickly get an overview because it will help a LOT when working with C++ and Unreal Engine, [Nice overview](https://learn.microsoft.com/en-us/cpp/build/dlls-in-visual-cpp?view=msvc-170).

The TLDR for linking symbols across modules is that the symbol must be explicitly exported from their source module with the `MYMODULENAME_API` macro or the symbols must be a header only implementation. <br>
If a class as a whole is exported then its functions are implicitly also exported. <br>
Unreal also has specifiers such as MinimalAPI or NoExport that can be used with `UCLASS`s but I find it be less explicit. Read more [here](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules), [here](https://dev.epicgames.com/documentation/en-us/unreal-engine/module-api-specifiers-in-unreal-engine) and [here](https://benui.ca/unreal/uclass/#minimalapi). <br>

If an engine class is NOT explicitly exported and you try to inherit from it then you will get linking errors, there is no real work around for this and I noticed epics push for removing whole class exports and relying on manually exporting each symbol has been the cause for a fair amount of unneeded headaches. In those cases you either need to copy the plugin to your projects plugin folder and edit the copied plugin there or copy the entire class if applicable and just specialize it for your own cases (though this is rarely needed).

Another cause for linking errors is when having **multiple definitions** of the same symbol, this is usually due to including a header file in multiple places that defines the same symbol in a global way or having a circular dependency / Missing Pragma once.


## Overview
### Plugin Overview
Plugins are a way to extend the functionality of the engine, they can contain code, content, or both. Plugins can be enabled or disabled on a per project basis and can be shared between projects. Plugins can also be distributed via the marketplace or other means.<br>

If a plugin is not inside of your projects `/Plugins` folder then you also must explicitly add it as a dependency in your `.uproject` file (unreal handles this when you click the tick in the editor to enable a plugin).
<br>


> You can copy plugins from the engine to your projects plugins folder and modify them without affecting the source engine. This can be a godsend for non custom engine builds where you can't modify the engine source code directly. <br>
{: .prompt-info }

A non code Plugin must contain the following: <br>
- A `.uplugin` file, this allows us to specify the modules within our plugin, when they are loaded, and other metadata about the plugin. It also allows us to specify other plugins that our plugin depends on.
- A `Source` directory containing the directories of modules within the plugin or a `Content` directory for content only plugins.
- Typically the plugin will contain a default module of the same name as the plugin.




<br>
### Module Overview
Each module can choose its type and loading phase which impacts when the module is loaded and what it can do. 
<br>
The Most Common Module Types Are ( [Extensive List](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Projects/EHostType__Type#values) ):
- **Runtime**: What you should use for most modules, this is for your gameplay code.
- **UncookedOnly**: This is for modules that should only be loaded in uncooked circumstances, useful for K2 nodes or other editor only code.
- **DeveloperTool**: Not needed often but gives an advantage over UncookedOnly in that it can in **any** configuration as long as `bBuildDeveloperTools` is true in your `.Build.cs` file, great for being able to debug or use something in a shipping build but the end goal isn't to ship with it.

The Most Common Loading Phases Are ( [Extensive List](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Projects/ELoadingPhase__Type) ):
- **Default**: This is the most common and is loaded after the engine has initialized.
- **PreDefault**: This is loaded before the Default phase, this is useful for modules that need to be loaded before other modules are loaded that depend on it, K2 nodes are a good example, sometimes if the loading order is weird you will have your node be disconnected upon start up because it loads after the module that uses it.

A module must contain the following: <br>
- A `.Build.cs` file, this file is used to describe how the module is built and what other modules it depends on.
- `.h` and `.cpp` files that contain the module class and the `IMPLEMENT_MODULE` macro.




## Creation

<details>
<summary>Creating A Plugin</summary>
Unreal has plugin wizards that can be invoked from the editor and will aid in creating a new plugin, this is one of the easiest way to create a plugin and is recommended for most cases if are not using Rider (Rider is a Jetbrains IDE that tries to specialize in game development and is used with engines like unity, it is a massive productivity boost compared to visual studio and esspecially VS code with unreal), if you are using rider then its a simple right click menu from your IDE.<br>


<details>
<summary>Wizard Approach</summary>
Start by launching the editor and navigating to the menu bar and locating <code>Edit->Plugins->Add</code>.<br>

Here we are presented with a list of templates to choose from, for most cases you will want to choose "Blank Plugin" but there are other templates like "Blueprint Function Library" or "Editor Toolbar Button" that can be useful if you are creating a plugin for a specific purpose (That toolbar one is actually super useful for a quick start for editor customization).<br>

<img src="/assets/img/UE/plugin_templates.png" alt="Plugin Templates"><br>
</details>


<details>
<summary>Manual Approach</summary>
Create a new directory in your projects <code>Plugins</code> directory, this directory should be named the same as your plugin.<br>
The file/folder structure should look like this:<br>
{% highlight text %}
Plugins/
    MyAwesomePlugin/
        MyAwesomePlugin.uplugin
        Source/
            MyAwesomePlugin/
                MyAwesomePlugin.Build.cs
                MyAwesomePlugin.cpp
                MyAwesomePlugin.h
{% endhighlight %}



<details>
<summary>Copy Pasta</summary>

<h3>.uplugin</h3>
{% highlight json %}
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "MyAwesomePlugin",
	"Description": "",
	"Category": "Other",
	"CreatedBy": "My Name",
	"CreatedByURL": "",
	"DocsURL": "",
	"MarketplaceURL": "",
	"CanContainContent": true,
	"IsBetaVersion": false,
	"IsExperimentalVersion": false,
	"Installed": false,
	"Modules": [
		{
			"Name": "MyAwesomePlugin",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	]
}
{% endhighlight %}


<h3>MyAwesomePlugin.Build.cs</h3>
{% highlight c# %}
using UnrealBuildTool;

public class MyAwesomePlugin : ModuleRules
{
	public MyAwesomePlugin(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;
			
		
		PublicDependencyModuleNames.AddRange(
			new string[]
			{
				"Core",
			}
			);
			
		
		PrivateDependencyModuleNames.AddRange(
			new string[]
			{
				"CoreUObject",
				"Engine",
				"Slate",
				"SlateCore",
			}
			);
	}
}
{% endhighlight %}


<h3>MyAwesomePlugin.h/cpp</h3>
{% highlight c++ %}
// Header File
#pragma once
#include "Modules/ModuleManager.h"

class FMyAwesomePluginModule : public IModuleInterface
{
public:

	/** IModuleInterface implementation */
	virtual void StartupModule() override;
	virtual void ShutdownModule() override;
};


// Cpp File
#include "MyAwesomePlugin.h"

#define LOCTEXT_NAMESPACE "FMyAwesomePluginModule"
void FMyAwesomePluginModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file per-module
}

void FMyAwesomePluginModule::ShutdownModule()
{
	// This function may be called during shutdown to clean up your module.  For modules that support dynamic reloading,
	// we call this function before unloading the module.
}
#undef LOCTEXT_NAMESPACE
	
IMPLEMENT_MODULE(FMyAwesomePluginModule, MyAwesomePlugin)

{% endhighlight %}


</details>
</details>

</details>

<br>

<details>
<summary>Creating A Module</summary>
Creating a module is way simpler and is often just a matter of creating a new directory in your `Source` directory named what the module is intended to be called and adding a `.Build.cs`, `.h` and `.cpp` file that contains the module class and the `IMPLEMENT_MODULE` macro. 
<br>
<br>

The folder struct of the new module should be located within a source folder of your project or plugin you wish to house the module and should look like this:<br> 
{% highlight text %}
Source/
    MyNewModule/
        MyNewModule.Build.cs
        MyNewModule.cpp
        MyNewModule.h
{% endhighlight %}

After creating the files you will need to add the module to your projects `.uproject` file or your plugins `.uplugin` file and specify its type and loading phase <a href="#module-overview">discussed here</a>. 
<br>

<details>
<summary>Copy Pasta</summary>
<h3>MyNewModule.Build.cs</h3>
{% highlight c# %}
using UnrealBuildTool;

public class MyNewModule : ModuleRules
{
	public MyNewModule(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;
			
		
		PublicDependencyModuleNames.AddRange(
			new string[]
			{
				"Core",
			}
			);
			
		
		PrivateDependencyModuleNames.AddRange(
			new string[]
			{
				"CoreUObject",
				"Engine",
				"Slate",
				"SlateCore",
			}
			);
	}
}
{% endhighlight %}


<h3>MyNewModule.h/cpp</h3>
{% highlight c++ %}
// Header File
#pragma once
#include "Modules/ModuleManager.h"

class FMyNewModuleModule : public IModuleInterface
{
public:

	/** IModuleInterface implementation */
	virtual void StartupModule() override;
	virtual void ShutdownModule() override;
};


// Cpp File
#include "MyNewModule.h"

#define LOCTEXT_NAMESPACE "FMyNewModuleModule"
void FMyNewModuleModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file per-module
}

void FMyNewModuleModule::ShutdownModule()
{
	// This function may be called during shutdown to clean up your module.  For modules that support dynamic reloading,
	// we call this function before unloading the module.
}
#undef LOCTEXT_NAMESPACE
	
IMPLEMENT_MODULE(FMyNewModuleModule, MyNewModule)

{% endhighlight %}

</details>
</details>



