Here is a general rundown of the inner workings of URF specifically. (I will use URF from now on to refer to Unified Race Framework) This manual should act as a guide to modders and developers who want to either extend or use the base functionality of URF for their own purposes. There will be a few sections which outline both the functionality and usage, as well as a quick tutorial on how to make your own new race, how neat!

## Inheritance, Overriding and Mod Structure
Crusader Kings 3 has a slightly complex file structure and overwriting system. However that is being used to it's full effect in URF so it's necessary that you know how to utilize it for your own extensions. Here are some basic facts:
* CK3's load order is a top down hierarchy, so mods are loaded from top to bottom. Ergo, if you want to overwrite the contents of another mod, you must place that mod underneath it in the hierarchy. For this reason, URF must **always** be placed as high in the load order to avoid it overwriting the mods it's supposed to be a library for.
* In the CK3 mod manager this can be helped by placing your lib library names with alphabetically first names (e.g using a ! or 0 will lead to it being placed at the top)
* In the file system, when CK3 is loading mods, it will go through all the folders alphabetically and load the given files. This also means that another way to ensure that some files are loaded before others is to name the files alphabetically to align the hierarchy to the top (you will notice all URF files start with the 0lib_ prefix)
* CK3 features a sort of inline override. That means there are two ways to override vanilla functionality.
	* 1. By naming the file identical to a vanilla or mod file, you effectively overwrite all the contents of that file. (e.g game_start.txt in the on_actions folder) However this is a universal overwrite and can break things, especially between versions. It is unrecommended unless you have to.
	* 2. By overwriting inline values. Effectively on runtime, paradox compiles all the individual files in a given folder (on_actions) into one record. This means if you add a new file to the folder then it will be compiled as if it was a new record in that folder. This also means you can overwrite existing mod or vanilla values by making sure your file is loaded after the vanilla ones. A great example is this:
		* urf_game_start.txt is loaded after game_start.txt the vanilla file
		* the URF file has `on_game_start` which will intercept the on_action in the game_start.txt file.
		* after that we can add a new line to the `effect = {}` clause called `urf_game_start_init = yes`
		* this effectively allows URF to intercept and insert the needed `urf_game_start_init = yes` code into the game_start.txt file without breaking existing code. This means that no matter what vanilla code is changed by paradox in the game_start code, URF will continue to function as normal.

Effectively, the inheritance and overriding functionality here works across all modding in ck3. However, it is especially useful for libraries where you need to maintain compatibility as much as possible.

## A Rundown of URF
Here is a brief written down segment on the basic concept of how URF works. This is in no means exhaustive, or is it accurate in every area. As always, delving into the code is the best way to get an idea on how things work, but because of the abstraction used in the library it may be difficult for those unfamiliar with more complex scripting in PDX to grasp on first glance. Thus this rundown.

In short URF works not by assigning race traits in the normal sense. CK3 traits are very versatile, but we as modders simply don't have complete control over how they work and are inherited by the game. While some of this can be controlled by on_birth on actions and various running event cycles, it turns out to be a very arduous process to both setup and debug in the event of errors. Because of this, I have moved to a system that relies on persistent character variables. But first a moment to talk about scripted variables.

> Scripted variables work as the manifestation of character flags in the backend. Character flags are essentially expirable flags we can set on characters. Normally these are set with the basic `add_character_flag` function and are empty, only having their name. But due to the magic of programming, we can add values to the flags using the `set_variable` function! This allows the variable both have a name and a value, which is super useful for storing information about a character for later use. Such as race or it's genetic makeup

All of these changes to the variables act as stand ins for traits, this allows a complex percentage based inheritance of the races. These are split into two major pieces phenotype and genotype. The genotype is collected from the character's parents and put into a percentage (e.g half elf is 50% human 50% elf). Whichever percentage wins the threshold get's to be the phenotype which determines the visible "race" of the character. This is then displayed through the GUI in a separated part of the trait window.

Because of this, each race can also have modifiers applied manually through the code as well. This acts like the modifiers part of the trait window. The GUI also grabs this and shows on hover over the race button which allows the player to see their bonuses. Essentially every part of URF is built to look and feel like vanilla functionality, but in the end it is much more complicated and customizable.

Finally we get to the inner workings of URF as an outline. This is more so oriented to the modder who wants to make changes or extensions, so pay attention if you want to add more races etc etc.
## The Inner Workings of URF
URF is pretty segmented between three separate areas. These are listed below
1. Common
2. GUI/GFX
3. Localization
Because of this, here is a breakdown of each section and how they work so you can better understand the library.

### Common
This area is roughly broken down into a few areas itself
* Innovations
* Customizable Localization
* Game Concepts
* Game Rules
* Modifiers
* On Actions
* Character Templates
* Scripted Effects
* Scripted Triggers
#### Innovations
This folder is pretty simple, though complex. It will make more sense as we move along. It in effect creates a new innovation for each race icon in the GUI. This is needed because of a weird problem in the GUI which doesn't allow trait icon lookup. This is a workaround and should not effect gameplay. However **this is needed to add new races**
#### Customizable Localization
This is the localization constants for new races that are needed. Specifically `GetRacialDescription` needs to be overwritten and added to add new races. You will need a new phenotype trigger as well as a localization key for your racial description.
#### Game Concepts
This adds the new game concepts associated with sapience, and races. This file normally doesn't need to be overwritten if you intend to not change URF's default functionality. However you will need to add new concepts for the races you do add. Here is an example:
```
human = {
    texture = "gfx/interface/icons/race/urf_human.dds"
    alias = { man humans men mankind humankind }
    parent = race
}
```
#### Game Rules
Pretty simple, only really meant for usage with sapience. It allows you to disable sapience in your own game.
#### Modifiers
This doesn't need to be overwritten, but does hold the template for the race modifiers that you need. To add new ones you must have the format here `urf_phenotype$YOURRACE$_modifier`
#### On Actions
This isn't necessary to change. However, this handles the handing out of the racial and sapience variables using the `on_game_start` on action and the `on_child_birth` look at the scripted effects stuff for more indication on what is being done here.
#### Character Templates
While this isn't necessary to change. It essentially adds the new effect `urf_template_base_race_init = yes` to all character templates. this ensures that all characters generated by the game (not on game start and not born) receive the racial and sapience variables. Without this, the pool of variables will be tainted and you will start to see strange gameplay and race calculations.
#### Scripted Effects
Oh the doozy. The majority of the magic and calculations happen here. While already explained in the rough rundown, this should list what needs to be changed or added for new races. **This is the most important folder to pay attention to.**
* In **0lib_urf_race_init_effects.txt** you need to overwrite the ``urf_for_all_phenotypes`` by adding new `$APPLY$ = { RACE = human }` lines for each race you want to add. This uses inverse logic to apply each calculation to every race listed.
* In **0lib_urf_on_birth_init_effects.txt** you need to overwrite the `urf_on_birth_genotype_assignment` with the two lines `urf_on_birth_potential_assignment = { CHILD = $CHILD$ FATHER = $FATHER$ MOTHER = $MOTHER$ RACE = human }` and `urf_genotype_phenotype_threshold = { RACE = human }` for each race you want to add. This set's the threshold and potential assignment on birth for each race. Note that this will get more complicated if you want to have specific rules for the threshold, but it can be done.
#### Scripted Triggers
Importantly, the overwrites needed for the Scripted Triggers are compiled here:
* `urf_for_all_phenotypes_trigger` needs to be overwritten with the ``$APPLY$ = { RACE = human }`` line in an `OR = {}` statement for each race
* `urf_for_all_phenotypes_comparator_trigger` also needs to be overwritten in an `OR = {}` statement for each race needed `$APPLY$ = { RACE = human COMPARATOR = $COMPARATOR$ }`

### GUI/GFX
While the changes in the GUI are sweeping, this will not focus on those changes. A rundown could be written but would take far longer. The changes need to add new races in the gfx is simple.
* The files need to match this path **gfx/interface/icons/race** and need to match this format `urf_$YOURRACE$.dds` note that the urf is needed for the GUI to match the files properly. Also make sure that your addition in the innovations section matches this file as well.
* The name of the race in the gfx files has to match that of your race name in scripted effects. (e.g gfx file = urf_human.dds and scripted effects file = human)

### Localization
Effectively what you need for localization is this:
* A entry for every race, without the urf_ prefix. (e.g human/game_concept_human)
* Note that the race concept has to match the race name added in all of your scripted effects files. Otherwise this will not match

With this guide in mind, you should now have a decent understanding of the basic workings of URF and the steps needed to extend the library.
