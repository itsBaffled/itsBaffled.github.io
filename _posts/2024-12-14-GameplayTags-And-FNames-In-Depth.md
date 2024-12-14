---
layout: post
title:  "Unreal Engine: GameplayTags and FNames In Depth"
date:   2024-12-14 23:02:00 +1100
permalink: /posts/UE/GameplayTags-And-FNames-In-Depth
---


## Introduction

The purpose of this blog post is to provide not only a slightly more in depth look at FNames and GameplayTags but to also provide the information together in a single place 
so that it is easier to grasp the entire context of each system and how they interact with each other. <br>

I'm going to begin with a look at FNames and how they work and then move onto GameplayTags and how they are built on top of FNames, I will also provide some tips and 
tricks for using them in your projects. <br>

Feel free to jump to the section you are interested in, I will provide a TLDR at the end of each section for those who are in a hurry or just want a quick recap, 
but if you have limited knowledge, have not read this before or want to make sure there isn't anything missed then I suggest reading it first the whole way through as it was intended. <br>
<br>

### FNames
```cpp
UPROPERTY(EditAnywhere)
FName MyName;
```

[FNames](https://dev.epicgames.com/documentation/en-us/unreal-engine/fname-in-unreal-engine) are unreals solution to cheap and efficient comparisions 
of [immutable](https://www.wordnik.com/words/immutable) strings in a way that can be compared and passed around quickly due to being hashed. <br>

They work by first taking a string of characters and then hashing its contents using a CityHash algorithm to create a unique identifier for that string,
these strings are then stored in a global pool called `FNamePool` and can be accessed by their hashed value or Index. <br>



There is a globally defined macro called `WITH_CASE_PRESERVING_NAME` that allows for case preservation in FNames, this is because by default FNames are non case 
preserving **unless** you are building with the editor.
<img src="/assets/img/UE/case_preserving.png" alt="Case Preserving engine macro" style="width: 100%;">

Any builds that contains the editor will have `WITH_CASE_PRESERVING_NAME` enabled by default, this is because the editor wants to visibly record and show the spelling of `Foo` and `foo` but in shipping builds this is not needed and can/should be disabled to save on memory, 
the only case where you would want this is when you to convert back from an FName to a string in a lossless manor.

This is NOT supposed to help differentiate between `"Foo"` and `"foo"` in a comparision as the `DisplayIndex` (see below) is not taken into account there. <br>

>Note: This is a snippet from the `FNamePool` showing how the display names are stored if enabled, as you can see it adds overhead to not only the pool but every FName that exists. <br>

```cpp
// inside of FNamePool
#if WITH_CASE_PRESERVING_NAME
FNamePoolShard<ENameCase::CaseSensitive> DisplayShards[FNamePoolShards]; 
#endif


// Inside of FName
#if WITH_CASE_PRESERVING_NAME
/** Index into the Names array (used to find String portion of the string/number pair used for display) */
FNameEntryId DisplayIndex; // 4 bytes in size, FNameEntryId is a wrapper for a uint32 with some utility functions
#endif
```

<br>

An `FName` consists of 2/3 uint32's:
- `ComparisonIndex` - This is the index of our hashed string result which is stored in the global FName pool.
- `Number` - This is a unique int that is associated with the string, more on this below but it allows us to reuse FNames without them being equal to each other, IE `Foo_1` and `Foo_2` are not equal to eachother but both have the same `ComparisonIndex`.
- `DisplayIndex` - This is the optionally compiled out value that is responsible for tracking our case sensitive names index in the global FName Pool (only if `WITH_CASE_PRESERVING_NAME` is enabled).

Lets take an example of constructing some FNames and see what would happen, note that I wont be discussing `DisplayIndex` anymore as it's only relevant for going back to a string with `ToString()` in most cases:
```cpp
FName Name1 {"Hello"};
FName Name2 {"Hello_1"};
```
This results in `Name1` having a hashed `ComparisonIndex` value of `"Hello"` and a `Number` value of `0`, this might seem pretty obvious but how about `Name2`? 

`Name2` is going to contain the same hashed `ComparisonIndex` value as `Name1` since they both have the exact same string up until `_` from there unreal will 
parse the string and as long as no invalid characters are found and it's less than or equal to 10 total max digits, then the `Number` value will now store the number given **plus** one, 
otherwise unreal would hash the string with the `_` inside of it, for example `"Hello_032"` is stored as `"Hello_032"` with a `Number` value of 0 because `_032` is an invalid start to a `Number`, more on 
the invalid characters later but here is the parsing code from epic, we see how it only counts digits that start with 0 if its the **only** digit. <br>

##### Parsing code
<img src="/assets/img/UE/fname_digits_parse.png" alt="" style="width: 75%;">


So I said a lot, but essentially it means that if we supply a valid number in the name then it's going to store the number we gave plus one, so `Name2` will have a `Number` value of `2`, 
this may seem unintuitive but it is due to unreal having `0` represent not used, so if you were to supply `"Hello_0"` for example unreal would make it contain a `1` internally to represent it is in fact used. <br>

Here is a screenshot of rider showing me the inline values, the inline debugger is smart enough to know it should display the `Value` minus one for the `Number` of 
the inline displayed value but we can see by calling `GetNumber()` it returns the actual internally stored value.
<img src="/assets/img/UE/fname_comparison_inline.png" alt="" style="width: 100%;">


Here shows a watch window the values expanded showing the important information, we can see they all share the same hashed value indexes but have different `Number` values.
<img src="/assets/img/UE/fname_compairison_watch_window.png" alt="" style="width: 100%;">


if we were to take our FName and convert it back to a string, as long as our `Value` int has a non 0 value then our resultant string will contain the `_` character and the number, otherwise it will not be included in the string.
```cpp
FName Name1 {"Hello_1"}; // Stored internally as "Hello" and 2
FName Name2 {"Hello"}; // Stored internally as "Hello" and 0
FString Name1Str = Name1.ToString(); // Results in "Hello_1"
FString Name2Str = Name2.ToString(); // Results in "Hello"
```
Unreal takes **great** advantage of the `Number` for example: UObject instances, it is why in the debugger you often always see the `MyObjectName_1` or `MyObjectName_2` etc, this is because unreal is taking advantage 
of the `Number` to give you a unique name for each object. <br>

#### Construction of an FName
Lets follow the construction of an FName:

- 1) Start by creating a basic FName like so `FName MyName {"FooBar"};`
    - Side Note: This constructor takes a second parameter which is an `EFindName` enum that can be either `FNAME_Find` or `FNAME_Add`. 
        - `FNAME_Add` will add the name to the table if it is not found, this is the default behaviour.
        - `FNAME_Find` will return a 0 or `NAME_None` if the name is not found.
```cpp
FName::FName(const ANSICHAR* Name, EFindName FindType) 
	: FName(FNameHelper::MakeDetectNumber(MakeUnconvertedView(Name), FindType))
{}
```

- 2) Here we can see a call to `FNameHelper::MakeDetectNumber` (at least with this constructor, FName has a lot, I recommend checking them out) which is what actually handles making the FName and parsing for that `_` character.

- 3) `FNameHelper::MakeDetectNumber` will then call `ParseNumber()` which returns the number plus one like discussed [earlier](#parsing-code) but if it's not used or can't be used it returns 0. 

- 4) Finally we eventually end up at `MakeInternal()` which is going to first find or create a new entry in the global FName pool for our string and then is going to construct a return value FName with the `Number` 
set to the parsed number and the `ComparisonIndex` set to the hashed index of the string.


#### Caveats
An FNames hashed `ComparisonIndex` can change between launches of the engine even if no new names are added, for example, here I launch the engine 5 times with making no modifications at all, 
just running and quitting and we will see how the `ComparisionIndex` changes, here is the variable I will be inspecting every launch <br>
- `FName Name1 {"MyNonPreExistingTag"};`<br>
 and here is the results
```cpp
1579582 // Run 1
1579540 // Run 2
1579553 // Run 3
1579564 // Run 4
1579593 // Run 5
```
This means that we cannot reliably network the **index** between clients (unreal sends them as strings) or serialize the FName directly, the order in which strings are added into the pool has a direct effect on their `ComparisionIndex` with one exception:
`EName`, ENames are an engine hardcoded mapping to constants and they can reliably be networked between clients but you cannot go extending them without engine modifications and this isn't usually required since
Gameplay Tags are the solution to cheap networked FName like tags. <br>



Let us just go over a couple more examples to make sure we understand how they work and see some other constructors that are available to us:
```cpp
// Returns an FName of "He" with Value of 0 
// because the first argument supplies the length of the string to account for
FName LimitedName {2, "Hello_1234567890"}; 

FName Name1 {"Foo_1"}; // Returns an FName of "Foo" and Value of 2
FName Name2 {"Foo_0"}; // Returns an FName of "Foo" and Value of 1
FName Name3 {"Foo"}; // Returns an FName of "Foo" and Value of 0


// Returns an FName of "Foo" and Value of 1 
// (this allows us to be explicit with the number and it wont be auto incremented by unreal)
FName Name4 {"Foo", 1}; 
FName CheapConstruction { Name4 }; // Cheap to copy around and construct, only around 8 bytes in normal builds

// Both result in an invalid NONE index because we cannot have an empty string as an FName
FName Name5 {"", 4};
FName Name6 {"_4"};
```



<br>

#### Quick TLDR/Recap and some extra notes about FNames:
- FNames consist of 2-3 parts; `ComparisonIndex`, `Number` and `DisplayIndex` (`DisplayIndex` is stripped if not using `WITH_CASE_PRESERVING_NAME`).
- FNames are case insensitive, meaning `"Foo"` and `"foo"` will both be equal to each-other, even with case sensitivity preserved.
- FNames equality operator `==` is defined to compare via `ToUnstableInt()` which treats its own memory as an uint64 `FMemory::Memcpy(&Out, this, sizeof(uint64));` this means no matter if we are preserving case sensitivity or not the comparison will be the same and not take the `DisplayIndex` into account.
- When you compare two FNames you are comparing the hashed index of the strings and not the string itself (just two ints - `ComparisionIndex` and `Number`, extremely cheap!) .
- `FName A {"Foo"};` is a lot more expensive to construct than `FName B { A };` because we are member wise copying A's int values as opposed to A needing to hash itself on the string to first get it's values.
- FNames are stored in a global name pool called `FNamePool`, if you have a lot of unique FNames you will be using a more memory but you can mitigate this by taking advantage of the `Number` int associated with every `FName`.
- FNames are extremely cheap to copy and pass around, it's only `8` to `12` bytes depending on if case sensitive preservation are enabled.
- The first entry of a string in the pool is how the string will be stored, subsequent entries that match will not be stored (unless preserving is enabled but even then it doesnt affect the `ComparisonIndex` but just it's `DisplayIndex`), this means that if `"FoOOo"` is registered before `"Foooo"` then when the second FName attempts to register it will be told it already exists in the pool it will continue to use that hashed index, this means later if you tried to convert back to a string and did not have case sensitivity enabled you would get `"FoOOo"` back no matter what you supplied. Basically, the first instance to be registered in the table will continue to be used so be weary of this.
- You can enable case sensitivity in your project by adding `GlobalDefinitions.Add("WITH_CASE_PRESERVING_NAME=1");` to your build.cs file but requires a custom engine build.
- FNames are hashed using `CityHash` which is a hashing algorithm that is designed to be quick and deterministic.
- FNames are not reliable to network between clients or serialize directly, the `ComparisonIndex` can change between launches of the engine even if no new names are added, the only exception to this are ENames.
- The max size for an FName is defined as `1024` characters.
- The max digits in an `FName` is limited to `10` and must be a value less than uint32's max, meaning `"Foo_1234567890"` contains the maximum length of an FNames `Number` value and will be stored as `"Foo"` and 1234567890.
    - If you were to go over that like so `"Foo_12345678901"` then it would become just the string of `"Foo_12345678901"` and the `Number` value would be the default `0` to signify it's not used.
- `EName` is an enum class for a `uint32` and is used by unreal to define hardcoded FNames along with their respective index, this allows for the engine to reliably use these names and 
have a constant mapping to them, even in a network context. Cannot be extended **without** explicit engine modifications, see `UnrealNames.inl` for the full list of hardcoded names, some examples of accessing these global variables are anything 
that has the common `NAME_` naming convention, for example `NAME_None` or `NAME_Cylinder`.

##### Invalid characters:
```cpp
/** These are the characters that cannot be used in general FNames */
#define INVALID_NAME_CHARACTERS			TEXT("\"' ,\n\r\t")

/** These characters cannot be used in object names */
#define INVALID_OBJECTNAME_CHARACTERS	TEXT("\"' ,/.:|&!~\n\r\t@#(){}[]=;^%$`")

/** These characters cannot be used in ObjectPaths, which includes both the package path and part after the first . */
#define INVALID_OBJECTPATH_CHARACTERS	TEXT("\"' ,|&!~\n\r\t@#(){}[]=;^%$`")

/** These characters cannot be used in long package names */
#define INVALID_LONGPACKAGE_CHARACTERS	TEXT("\\:*?\"<>|' ,.&!~\n\r\t@#")

/** These characters can be used in relative directory names (lowercase versions as well) */
#define VALID_SAVEDDIRSUFFIX_CHARACTERS	TEXT("_0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz")
```


<br>

---

### GameplayTags

Example
```cpp
UPROPERTY(EditAnywhere, meta = (Categories = "Weapon.AR,Weapon.SMG") )
FGameplayTag WeaponTag;
```

Gameplay Tags are a framework that is built ontop of FNames, they allow you to structure and organise your FNames into groups of hierarchical tags that are seperated by a `.` character, for example:
- `Item.Equippable.Weapon`
- `Item.Consumable.Potion`

They allow for a more structured and organised way to group and categorize your various tags.

All Gameplay Tags are stored in a central tag dictionary upon startup and cannot be modified in gameplay.



Gameplay Tags are used to represent names or identifiers, contextual state information, attributes, markers and many many other things. 
Gameplay Tags by themselves are useless and represent nothing but an ID, it's what you build ontop of them and how you interpret them that gives them meaning.

They can also be used to enforce a naming convention or remove any chance of a typo which is not possible with FNames alone (there are nice [meta tags](https://benui.ca/unreal/uproperty/#getoptions) which can aid with this though.). <br>


GAS (Gameplay Ability System) is a system that heavily relies on Gameplay Tags to define and categorize abilities and effects. They make common use of `FGameplayTag` being used as a kind of "boolean" in which stops other abilities if specific tags are already found active on the player. GAS is however, a massive topic that I won't go into here and really has nothing to do with Gameplay Tags specifically, you can completely never touch GAS and make extensive use of Gameplay Tags. <br>

Another example of epic making use of Gameplay Tag is with their `UGameplayMessageSubsystem` plugin found in lyra where Gameplay Tags are used to categorize and filter messages that are sent to the player or other listeners.

<div style="display: flex; align-items: center;">
    <div style="flex: 1;">
        <p>An example using Gameplay Tags for a weapon tagging system could be <code>Weapon.Type.Name</code>, for example <code>Weapon.AR.AK47</code> where <code>Weapon</code>
        is the root tag, <code>AR</code> is the parent tag and <code>AK47</code> is the child tag, here we start to see the hierarchical power of using Gameplay Tags, this allows for a more structured and organised way to group and categorize your various tags and now we could check if the players weapon tag has a <code>Weapon.AR</code> tag without caring what type of <code>AR</code> weapon it is, but we also could compare exactly to see if it has the full desired tag of <code>Weapon.AR.AK47</code>.</p>
    </div>
    <div style="flex: 1;">
        <img src="/assets/img/UE/simple_gameplaytags_hierarchy.png" alt="" style="width: 100%;">
    </div>
</div>





#### Related Classes

The primary types you will interface with is `FGameplayTag` and `FGameplayTagContainer`.

`FGameplayTag` consists of:
- `FName TagName;` - Single tag that can be queried and compared against other tags, it is the hashed FName of the actual tag, so `"Weapon.AR.AK47"` be the string that our FName has a hashed index of.

`FGameplayTagContainer` (Should be prefered over you using a `TArray<FGameplayTag>` yourself) consists of:
- `TArray<FGameplayTag> GameplayTags;` - Contains each individual tag that is stored in the container, this is not to say it looks something like `Weapon` and `AR` and `AK47` but rather it `Weapon.AR.AK47` is stored as a single tag, it's parent tags are then stored in the `ParentTags` array.
- `TArray<FGameplayTag> ParentTags;` - Contains the parent most tags of the tags in the Gameplay Tags array, for example if we have `Weapon.AR.AK47` then `ParentTags` would contain `Weapon` and `Weapon.AR` or `Hello.World` would contain just `Hello`.

Unreal provides a generic interface called `IGameplayTagAssetInterface` which can be used to streamline the API between your classes but it is not required in way.




#### How to add them to your project
Gameplay tags can be added a project in a few different ways such as:
- Adding them to your Config folders `DefaultGameplayTags.ini` file or any other `.ini` file that resides in `Config/Tags` for example `Config/Tags/MySecondaryTags.ini` 

```cpp
# Can be added via appending to the UGameplayTagsSettings class array like so
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagList=(Tag="MyTag",DevComment="")

# or can be added directly to a GameplayTagsList like so
[/Script/GameplayTags.GameplayTagsList]
GameplayTagList=(Tag="MyTag",DevComment="")
GameplayTagList=(Tag="MyTag.MySubtag",DevComment="")
```
- You can modify and edit tags within the editors GameplayTag Manager window from `ProjectSettings -> GameplayTags`, here it can add and remove tags from an `ini` file without needing to actually for you to go in the `ini` file.
- Using the `UE_DECLARE_GAMEPLAY_TAG_EXTERN` and `UE_DEFINE_GAMEPLAY_TAG` [macros](#native-gameplay-tags) in your code to define new or existing tags without needing to query the `UGameplayTagManager::Get().RequestGameplayTag()`.
- Importing them via Data table assets that extend from `FGameplayTagTableRow`.
    - This approach allows for managing the tags via the editors Gameplay Tags Manager window as well modifiying the table asset itself. 
    - To import tags from a Data Table, do the following in `ProjectSettings -> GameplayTags`:
        - Click the Add Element (+) button next to Gameplay Tag Table List.
        - Click the dropdown for the new index and select your Data Table.



#### Usage examples

Gameplay tags have the ability to query the hierarchy of other gameplay tags, this allows you to check for an exact match or a partial match, for example if you have a tag `Weapon.AR.AK47` and you want to check if the weapon is of
an `AR` type, you could do so by checking if the `Weapon.AR` tag is contained within the `Weapon.AR.AK47` tag.

<img src="/assets/img/UE/tag_functionality.png" alt="" style="width: 100%;">

The demonstrated examples also exist in `FGameplayTagContainer` form to query multiple tags at once.




As your project grows you will notice that your Gameplay Tag drop downs can become quite cluttered, this can be significantly mitigated by filtering tags using the meta specifier like so:
- `UPROPERTY(meta = (Categories = "Tag1.Tag2.Tag3") )` 

and now only tags that are children of `Tag3` and `Tag3` itself will be shown in the dropdown, you can also add multiple categories to a filter for like so:
-  `UPROPERTY(meta = (Categories = "Weapon.AR,Weapon.SMG") )`

 and now only themself and tags that are children of `Weapon.AR` or `Weapon.SMG` will be shown in the dropdown.

- Do's and Don'ts
    - `Item.Apple.Heal` or `Attachment.AK47.Magazine` leads to broken hierarchies and can be difficult to work with, instead use `Item.Heal.Apple` or `Attachment.Magazine.AK47`, structure your tags in ways that make it easy to add sub tags to that aren't going to break the hierarchy.
    - Be consistent with naming and plan out hierarchies before you start adding tags.
    - Have tags only be as specific as they need to, global reusable tags are the key. For example imagine an FPS where you may need to constantly differentiate if something is for the first or third person model, take animation montages
    for example, you could go with a naive approach of `Anim.Montage.Player.FirstPerson.Attack` and `Anim.Montage.Player.ThirdPerson.Attack` but now what about when we have other things like a view model skin `Cosmetic.Skin.FirstPerson.Jerry` and `Cosmetic.Skin.ThirdPerson.Jerry`, we can see that we are starting to get a lot of duplication and what if we add a different kind of view like `Spectator` or `Vehicle`, this just becomes unmaintainable and grows out of control, instead you should be using something like `Anim.Montage.Player.Attack` or `Cosmetic.Skin.Jerry` with a secondary tag on the objects that also specify some other contextually relevant information like `FirstPerson` or `ThirdPerson` or `Spectator` or `Vehicle`, this applies to many cases with Gameplay Tags so don't be so quick to add a new tag, think about how you can reuse and structure your tags in a way that is maintainable and scalable (this could also be a boolean or literal enum too, it was just to get the point across, it is easy to abuse Gameplay Tags but I personally don't think Gameplay Tag abuse is the end of the world, just be aware.).






#### Native Gameplay Tags
Any c++ code that uses Gameplay Tags needs to add the "GameplayTags" module to their modules build.cs file like so:
```c#
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "EnhancedInput", "GameplayTags"});
```


```cpp
// Should be declared in a header to share with other files and is used 
// in conjunction with a DEFINE_GAMEPLAY_TAG macro to define a new GameplayTag
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TagName);

// Used to define a new GameplayTag with a name and optionally a comment
UE_DEFINE_GAMEPLAY_TAG(TagName, Tag);
UE_DEFINE_GAMEPLAY_TAG_COMMENT(TagName, Tag, Comment);

// Used for defining tags in a cpp file that is not shared with other files
UE_DEFINE_GAMEPLAY_TAG_STATIC(TagName, Tag);
```

These macros create a `FNativeGameplayTag` variable which is a wrapper for a `FGameplayTag` that is able to register with the global `UGameplayTagsManager` and then store the tag internally so you can access it whenever it is needed without
having to go through the `UGameplayTagsManager::Get().RequestGameplayTag()` every time which can be slow and unneeded since unreal provides these `FNativeGameplayTags` in a convenient macro. <br>


<details>
<summary>👈🏻 My Personal Preference of how I store Gameplay Tags from c++</summary>

My recommended way to store and use Gameplay Tags from c++ that <b>aren't</b> already set/populated from the editor is to store them inside of some global namespace that can be shared and used between multiple files like so.

I use this as a way to reach into tags that I already define via my `DefaultGameplayTags.ini` file but there is nothing requiring you to define them inside of your ini file first but I 
think it's good practice to be explicit about what tags exist and do not from a single place. <br>

{% highlight cpp %}
// GlobalTags.h
namespace ProjectShortName::Tags
{
    namespace Stats
    {
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_STAT);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_STAT_HEALTH);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_STAT_MANA);
    }

    namespace UI
    {
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_UI);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_UI_MENU);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_UI_INTERACTION);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_UI_INTERACTION_BUTTON);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_UI_INTERACTION_SLIDER);
    }

    namespace Weapon
    {
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_WEAPON);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_WEAPON_AR);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_WEAPON_AR_AK47);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_WEAPON_SMG);
        UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_WEAPON_LMG);
    }
    // Etc
}

// The cpp file
namespace ProjectShortName::Tags
{
    namespace Stats
    {
        UE_DEFINE_GAMEPLAY_TAG(TAG_STAT, "Stat");
        UE_DEFINE_GAMEPLAY_TAG(TAG_STAT_HEALTH, "Stat.Health");
        UE_DEFINE_GAMEPLAY_TAG(TAG_STAT_MANA, "Stat.Mana");
    }

    namespace UI
    {
        UE_DEFINE_GAMEPLAY_TAG(TAG_UI, "UI");
        UE_DEFINE_GAMEPLAY_TAG(TAG_UI_MENU, "UI.Menu");
        UE_DEFINE_GAMEPLAY_TAG(TAG_UI_INTERACTION, "UI.Interaction");
        UE_DEFINE_GAMEPLAY_TAG(TAG_UI_INTERACTION_BUTTON, "UI.Interaction.Button");
        UE_DEFINE_GAMEPLAY_TAG(TAG_UI_INTERACTION_SLIDER, "UI.Interaction.Slider");
    }

    namespace Weapon
    {
        UE_DEFINE_GAMEPLAY_TAG(TAG_WEAPON, "Weapon");
        UE_DEFINE_GAMEPLAY_TAG(TAG_WEAPON_AR, "Weapon.AR");
        UE_DEFINE_GAMEPLAY_TAG(TAG_WEAPON_AR_AK47, "Weapon.AR.AK47");
        UE_DEFINE_GAMEPLAY_TAG(TAG_WEAPON_SMG, "Weapon.SMG");
        UE_DEFINE_GAMEPLAY_TAG(TAG_WEAPON_LMG, "Weapon.LMG");
    }
    // Etc
}

{% endhighlight %}
</details>



#### Replication Information
Gameplay Tags also replicate a lot more efficiently over the network compared to FNames (which are replicate via their entire string),

Gameplay Tags must be statically/editor time defined (meaning not at runtime) or at least added into the `UGameplayTagsManager` before the engine has finished initializing, this can play a vital role in ensuring that the tags are mapped and replicated correctly over the network and we do not have any out of sync tags between clients.


Unreal provides a few configuration options found it `ProjectSettings -> GameplayTags` to help out with replication and efficiency of tags, these are:
- `Fast Replication` - If enabled cannot use `Dynamic Replication` and means that tags are replicated via a global index and all clients and server must agree on the mapping of tag to index, this is the most efficient way to replicate tags but also the most restrictive.
- `Dynamic Replication` - If enabled is supposed to able replicate tags between clients and server even if locally they have different tag dictonaries, the client and server come to agreement of a cheaper index to use for the tag rather than it's string representation, this is not as efficient as `Fast Replication` but allows for more flexibility, it also requires [IRIS](https://dev.epicgames.com/documentation/en-us/unreal-engine/iris-replication-system-in-unreal-engine) to be enabled in your project and from what I can tell is still experimental, so I don't see this as an option as of right now.
- `CommonlyReplicatedTags` - An array of FNames that represent our Gameplay Tags that are commonly replicated between clients, this is then converted to an array of FGameplayTags upon startup and the comment for that array reads as `"These tags are given lower indices to ensure they replicate in the first bit segment."` which just ensures the most common tags are the cheapest to send, more information on bit segments below.
    - `GameplayTags.PrintReplicationFrequencyReport` is a super useful console command that can aid in letting you know which tags you replicate the most and if they should be added into your `CommonlyReplicatedTags` array, it will give you 
    a text print out in the console to copy paste in your `ini` file, it's super convenient. **DO NOT OVERLOOK THIS**.
- `NumBitsForContainerSize` - The number of bits to use for the container size, this should be based on how large your containers tend to be, it's a simple `2^n -1` calculation where n is your bit count, the default is 6 bits which allows for a maximum of `(2^6) -1 = 63` tags to be stored in the container.
- `NetIndexFirstBitSegment` - Needs a little more explaining in my opinion. This is asking us how many bits we should minimally **ALWAYS** send, and if we need to replicate an index that is above that then we need to pay the remaining cost of our maximum possible tag count. So a simple example would be imagine we have 255 tags in our game as a whole, this number can be represented in `8 bits`, but if we were to 95% of the time only use the lower 32 Gameplay Tag indicies registered in our global tag map then we would be paying an extra `3 bits` every single time for little to no reason, so we could instead set our `NetIndexFirstBitSegment` to `5` and then we would only pay `5 bits` as a minimum every time and that would cover the first 32 tags (Unreal defaults to 16) and then if we needed to go above that we would pay the remaining cost of the end/remaining bit segment which is a hard set value of `MaxNumberOfBitsNeededToRepresentOurHighestIndex - NetIndexFirstBitSegment`, so in our case we have a max size of 255 which means `8 bits` and we are sending a `NetIndexFirstBitSegment` of `5` so we would pay `3 bits` for the remaining 223 tags, this is a slightly simplified example but I hope it helps to understand the concept a little better.

```cpp
- CommonTag    (index 23)     -> Binary: 10111|0        (5 bits for tag + 1 flag bit = 6 bits)
- NonCommonTag (index 245)    -> Binary: 11110|1|101    (5 bits first segment + 1 flag bit + 3 remaining bits = 9 bits)
```




<br>


#### TLDR/Recap
- Hierarchical tags that are seperated by a `.` character.
- Is just internally a wrapper for an FName but is backed by a much larger system that allows for more structured and organised way to group and categorize your various tags.
- Comparing Gameplay Tags via the equality operator `==` will compare their FNames which are super cheap.
    - Doing a `MatchesTag` or any of the other similar functions come with a slightly higher cost but are still very cheap compared to raw string comparisions.
- Can be used to reprsent names or identifiers, contextual state information, attributes of something and many many other things.
- Used all of the engine by many plugins and systems.
- Does NOT require GAS to be used.
- Can be used to enforce a naming convention or remove any chance of a typo.
- Can filter tags using the `UPROPERTY(meta = (Categories = "Tag1.Tag2.Tag3") )` to only show tags that are children of `Tag3` and `Tag3` itself in the dropdown.
- Can be added to your project via `DefaultGameplayTags.ini` or the editor.
- C++ should use the `UE_DECLARE_GAMEPLAY_TAG_EXTERN` and `UE_DEFINE_GAMEPLAY_TAG` [macros](#native-gameplay-tags) to define new tags.
- Replicates more efficiently over the network compared to FNames.
- Max limit of 65'535 tags in a project.
- Lot's of possible optimizations and configurations in `ProjectSettings -> GameplayTags` to help with replication and efficiency of tags.
- `GameplayTags.PrintReplicationFrequencyReport` is a super useful console command that can aid in letting you know which tags you replicate the most and if they should be added into your `CommonlyReplicatedTags` array.
- `NetIndexFirstBitSegment` is asking us how many bits we should minimally **ALWAYS** send, and if we need to replicate an index that is above that, then we need to pay the remaining cost of our maximum possible tag count.

---

### Conclusion

Hopefully this has been a helpful and informative look at FNames and Gameplay Tags, they are both incredibly useful and I couldn't imagine working without them.

Now get out there and start using them!











