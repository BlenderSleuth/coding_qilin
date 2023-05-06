---
title: "Quick Tip: How to save default Unreal Editor Preferences "
date: 2020-07-23
description: "Usually, Unreal Editor settings are not persistent between projects - every experimental test project, every new game jam will reset back to default settings. How can we make our favourite settings the default?"
draft: true
---

Once you become a bit more comfortable working in Unreal Engine, you might find that the default Unreal Editor settings are not to your liking. They're easy enough to change in the editor preferences menu{{< super >}}[1]({{< relref "saving_unreal_editor_prefs#Ref1">}}){{< /super >}}:

![AccessPrefs](/saving_unreal_editor_prefs/AccessPrefs.png)

However, what you will quickly notice is that these settings do not translate between projects. Every experimental test project, every new game jam will reset your settings back to the default. That isn’t really a big issue if it’s only one or two settings, but once you are consistently changing the same ten settings spread out across the interface it can become tedious.

Hang on, what does that ‘Set as Default’ button do then?

![SaveAsDefault](/saving_unreal_editor_prefs/SaveAsDefault.png)

If you look at your project `Config` folder, you'll notice that when you click that button a new file appears:

![PerProjectSettings](/saving_unreal_editor_prefs/PerProjectSettings.png)

When the engine loads a new project, it will first load the _Engine_ version of the file, located at:
```
UE_4.XX\Engine\Config\BaseEditorPerProjectUserSettings.ini
```
This is located in `C:\Program Files\Epic Games\` or `/Applications/Epic Games/` by default.
It then loads the custom _Project_ version of the file, modifying entries to reflect your preferences. 
There are a few problems with this:

* It only saves defaults for one section, for example the `General - Appearance` section.
* It only saves defaults for the current project, not for the entire engine installation.
* This is stored in the project folder, so it will overwrite other developers' settings if that file is checked into source control (there is a warning about this when you click the button).

There is also the 'Export' and 'Import' buttons which have similar problems.

What can be done about this? We can edit the engine config file `BaseEditorPerProjectUserSettings.ini` with our prefered settings. It is a little bit finicky but bear with me.

1. From a clean project, set up all your default settings, taking note of the different sections.
2. Click 'Set as Default' for each of the sections you changed.
3. Open the newly created _project_ file `Config\DefaultEditorPerProjectUserSettings.ini`.
4. (optional) To make it a bit cleaner, delete all the irrelevant settings. These are the default settings loaded from the engine version of the file. Depending on the number of sections you edited this could be quite a few. For example, if I only change the AssetEditorOpenLocation in `General - Appearance`, my `.ini` file looks like:
```ini
[/Script/EditorStyle.EditorStyleSettings]
...
LogFontSize=9
LogTimestampMode=None
bPromoteOutputLogWarningsDuringPIE=False
AssetEditorOpenLocation=MainWindow
...
```
I can edit this to simply
```ini
[/Script/EditorStyle.EditorStyleSettings]
AssetEditorOpenLocation=MainWindow
```
The hardest part here is matching up the `ini` identifier with the setting in the editor.

5. Go to your `Engine\Config` folder, and make a copy of `BaseEditorPerProjectUserSettings.ini` as a backup.
6. In `BaseEditorPerProjectUserSettings.ini` paste the contents of your project file `DefaultEditorPerProjectUserSettings.ini`. 

In the example above, all I would do is scroll to the very bottom of the file (So it's applied last and we can find it later) and paste:
```ini
...

; Custom default user settings:

[/Script/EditorStyle.EditorStyleSettings]
AssetEditorOpenLocation=MainWindow
```

7. Rename the `DefaultEditorPerProjectUserSettings.ini` to test if the engine settings are taking effect. If we leave it the Editor will overwrite our newly created default settings.

With all this completed, your settings should be saved the next time you create a project. If you've already saved defaults for a project they will take precedence, but otherwise you should be good to go!


Notes:
1. {{< anchor "Ref1">}} I use the amazing [UE4 Minimal](https://github.com/Sythenz/UE4Minimal) theme to make the Editor a bit easier on the eyes, so my screenshots might look a bit different.
