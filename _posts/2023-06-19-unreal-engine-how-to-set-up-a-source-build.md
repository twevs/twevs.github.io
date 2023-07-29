# Unreal Engine: how to set up a native source build

[Epic's instructions](https://docs.unrealengine.com/5.2/en-US/building-unreal-engine-from-source/) for building Unreal Engine from source unfortunately have the common effect of leading users to build much more of the engine than they need to, resulting in a waste of time, disk space, and sanity, especially when people then wonder why some modules (which were never meant to be built by the end user) are yielding build errors. Below are correct instructions for a mean and lean source build, compiling only those parts of the engine that your project depends on, based on Epic's own internal practices as revealed by [the repository for the cancelled _Unreal Tournament_ game](https://github.com/EpicGames/UnrealTournament) as well as the expectations of various tools such as [UnrealGameSync](https://docs.unrealengine.com/4.26/en-US/ProductionPipelines/DeployingTheEngine/UnrealGameSync/); I and many others are indebted to [Stephen Swires](https://swires.me/) for having tirelessly repeated the proper folder layout for such a build on the Unreal Slackers Discord server. Such a setup, as the title suggests, is known as a "native project", as opposed to a "foreign project".

Skip to step 5 if you have already cloned the engine's source code from Epic's Github repository.

1. If your project is Blueprint-only, turn it into a C++ project by adding an empty C++ class via the editor menu: `Tools -> New C++ Class... -> None -> Create Class`.

2. Follow [these instructions](https://www.unrealengine.com/en-US/ue-on-github) to gain access to Epic's Github repository.

3. Install [Git for Windows](https://gitforwindows.org/).

4. Open the folder into which you wish to install the engine in Windows Explorer (it is **highly** recommended that it should be on an SSD, not a hard drive), click on the address bar so as to select the path, replace it with `cmd`, then press Enter; a command-line window should appear, into which you then enter `git clone --filter=blob:none https://github.com/EpicGames/UnrealEngine.git` to clone the repository.

5. Once the repository is cloned, enter `cd UnrealEngine` and then the following command, removing exclusions of platforms that you intend to target; for example, if you intend to build for Android, remove `-exclude=Android`. Note that if you do not have a Launcher build from which to copy over the GoogleTest library files, you will probably also need to remove `-exclude=GoogleTest`. You can always rerun this script later with fewer exclusions if you need to.

	`Setup.bat -exclude=WinRT -exclude=Mac -exclude=MacOS -exclude=MacOSX -exclude=osx -exclude=osx64 -exclude=osx32 -exclude=Android -exclude=IOS -exclude=TVOS -exclude=Linux -exclude=Linux32 -exclude=Linux64 -exclude=linux_x64 -exclude=HTML5 -exclude=PS4 -exclude=XboxOne -exclude=Switch -exclude=Dingo -exclude=Win32 -exclude=GoogleVR -exclude=GoogleTest -exclude=LeapMotion -exclude=HoloLens`

6. Once the prerequisities have been installed, copy and paste your project folder (the one containing your project's `.uproject` file) into the folder into which you cloned the engine; given for example the project folder `MyGame`, you should typically see something along the following lines, ie your project folder should be side-by-side with the `Engine` folder:

	- `Engine/`
	- `FeaturePacks/`
	- `MyGame/`
	- `Samples/`
	- `Templates/`
	- `GenerateProjectFiles.bat`
	- `GenerateProjectFiles.command`
	- `GenerateProjectFiles.sh`
	- etc.
	
7. If you copied over a project whose binaries are source-controlled using Perforce, you will need to disable `Read-only` status on them to allow the build to overwrite them.

8. Enter `GenerateProjectFiles.bat` into the command-line window to generate `UE5.sln`.

9. Open `UE5.sln`, then make your game the startup project. Assuming you are using Visual Studio, this would be done in the Solution Explorer by expanding `Games`, right-clicking on your project name, then clicking on `Set as Startup Project`.

10. Whatever you do, and once again assuming you are using Visual Studio, do **not** click on `Build -> Build Solution` or press F7, as this will perform the aforementioned whole-engine build; instead, in the `Development Editor` + `Win64` build configuration (use the dropdown menus in the toolbar if you see different settings), click on `Build -> Build MyGame` or Control + F5 to launch the compilation and, ultimately, the editor. (Of course, if you know what you are doing, you can choose another build configuration such as `DebugGame Editor` and debug as usual.) Even compiling only those parts of the engine that your project needs can take hours on a mid-range PC, so do be patient.
