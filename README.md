# Unreal Game as a Shared Library (UELibrary) for Unreal 5.2

I've spent a lot of time figuring our how to set up UELibrary for my custom project. The [official documentation](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/BuildAsALibrary/) is helpful but unfortunately incomplete. I've compiled all of my findings in one post + added an example on how you can extend the base API.

## UELibrary Setup

Our goal is to build a game as a shared .dll library that we can use in an external Win32 application.

Follow the following steps:

1. Clone the [Unreal Engine source code](https://www.unrealengine.com/en-US/ue-on-github).
1. Find this line of code in _SlateApplication.cpp_ and remove it:

    ```C++
    // If we are embedded inside another app then we never need to be "active"
    bAppIsActive = !GUELibraryOverrideSettings.bIsEmbedded;
    ```

    This will fix [the mouse bug](https://forums.unrealengine.com/t/ueaslibrary-mouse-input-not-working-as-app-is-never-active/261483).

1. (Optional) Find this line of code in _DesktopPlatformBase.cpp_ and replace it:

    ```C++
    // Append any other options
    Arguments += " -Progress -NoEngineChanges -NoHotReloadFromIDE";
    ```

    with this line:

    ``` C++
    // Append any other options
    Arguments += " -Progress";
    ```

    This will solve the ["Engine modules are out of date"](https://forums.unrealengine.com/t/how-to-solve-engine-modules-are-out-of-date/564119) error when you open your projects in a custom built UnrealEditor.

1. Compile the Unreal Engine source code. Run Setup.bat, GenerateProjectFiles.bat, open UE5.sln. Select _Development Editor_ configuration, right click on UE5 and press Build.
1. Open _Engine\Binaries\Win64\UnrealEditor.exe_. Create a new Blank C++ project (no starter content required). Let's name it _UELibExtended_.
1. Navigate to your project location and open _UELibExtended.sln_. Find _UELibExtended\Source\UELibExtended.Target.cs_ and modify it:

    ```C#
    // Copyright Epic Games, Inc. All Rights Reserved.
    using UnrealBuildTool;
    using System.Collections.Generic;

    public class UELibExtendedTarget : TargetRules
    {
      public UELibExtendedTarget( TargetInfo Target) : base(Target)
      {
        Type = TargetType.Game;
        DefaultBuildSettings = BuildSettingsVersion.V2;
        IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_1;

        // new lines of code
        GlobalDefinitions.Add("UE_LIBRARY_ENABLED=1");
        LinkType = TargetLinkType.Monolithic;
        bShouldCompileAsDLL = true;
        ExtraModuleNames.AddRange(new string [] { "UELibExtended", "UELibrary"});
      }
    }
    ```

   This will build your game as a monolith .dll file and expose `UELibrary_` (e.g. `UELibrary_WndProc`) functions.

1. Select __Development Editor__ configuration, right click on _UELibExtended_ project and build it. This will generate the necessary modules for the UE5 Editor.
1. Open _UELibExtended.uproject_ and cook the content (Platforms -> Windows -> Cook Content). This will transform raw assets into platform specific packages.
1. Select __Development__ configuration and compile  _UELibExtended_ project. This will generate `UELibExtended\Binaries\Win64\UELibExtended.dll` library. and corresponding `UELibExtended.lib` file
1. Open _Developer Command Prompt for VS 2022_ from the Windows Start menu. Run these commands:

    ```batch
    cd path\to\UELibExtended.dll
    dumpbin /exports UELibExtended.dll
    ```

    Look for `UELibrary_InitA`, `UELibrary_InitW`, `UELibrary_Shutdown`, `UELibrary_Tick` and  `UELibrary_WndProc` symbols. If they are missing make sure that you've saved the _UELibExtended.Target.cs_ file and compiled `UELibExtended` project in _Development_, __not__ _Development Editor_ configuration.

1. Clone [UELibApp]() project. This is a Win32 app that will embed the UELibExtended game.

    Inside of this repo you will find the _UELib_ folder:

    ```txt
    UELibApp\
      UELib\
        include\
        lib\
    ```

    I have configured `UELibApp` project to search for header files in the `include` folder and look for `UELibExtended.lib` file in the `lib` folder.

1. Copy `UnrealEngine\Engine\Source\Runtime\UELibrary\Public\UELibraryAPI.h` to `UELibApp\UELib\include`.

1. Modify the path to your `UELibExtended.uproject` file in the `UELibApp.cpp`:

    ```C++
    // replace ..\\..\\..\\UELibExtended\\UELibExtended.uproject with the full path to your .uproject
    int Error = UELibrary_Init(hInstance, hWnd, L"..\\..\\..\\UELibExtended\\UELibExtended.uproject");
    ```

1. Copy `UELibExtended\Binaries\Win64\UELibExtended.lib` to `UELibApp\UELib\lib`.

1. Open _UELibApp.sln_, change the configuration to _Release_ and compile it.

1. Copy __all__ _.dll_ files from `UELibExtended\Binaries\Win64` to `UELibApp\x64\Release` and run the project.

You need to repeat the last 3 steps every time you recompile _UELibExtended_ game. You can automate this with PostBuild steps. I'm too lazy to learn it lol.

## Extending the base UELibrary API (per game)

You can expose more functions to the external application. The official documentation is very vague on what to do so we will craft our own example. Let's expose a `UELibExtended_CountActors` function that will return a number of actors in the current level.

1. Create blank `UELibExtended\Source\UELibExtended\UELibExtendedApi.h` and `UELibExtended\Source\UELibExtended\UELibExtendedApi.cpp` files.
1. Generate project files by right clicking on `UELibExtended.uproject` and selecting the _Generate Visual Studio project files_ option.
1. Modify header file `UELibExtendedApi.h`:

    ```C++
    // Copyright Epic Games, Inc. All Rights Reserved.
    #pragma once

    #ifdef UELIBEXTENDED_LIBRARY_DLL_EXPORT
    #define UELIBEXTENDEDLIBRARYAPI __declspec(dllexport)
    #else
    #define UELIBEXTENDEDLIBRARYAPI __declspec(dllimport)
    #endif

    extern "C"
    {
        UELIBEXTENDEDLIBRARYAPI int UELibExtended_CountActors();
    }
    ```

1. Modify source file `UELibExtendedApi.cpp`:

    ```C++
    // Copyright Epic Games, Inc. All Rights Reserved.
    #define UELIBEXTENDED_LIBRARY_DLL_EXPORT
    #include "UELibExtendedApi.h"
    #undef UELIBEXTENDED_LIBRARY_DLL_EXPORT

    #include "EngineMinimal.h"

    UELIBEXTENDEDLIBRARYAPI int UELibExtended_CountActors()
    {
        if (!GEngine) {
            return 0;
        }

        for (const FWorldContext& Context : GEngine->GetWorldContexts())
        {
            if (Context.WorldType == EWorldType::Game) {
                return Context.World()->GetActorCount();
            }
        }
        return 0;
    }
    ```

1. Compile `UELibExtended` project in __Development__ configuration. Examine _UELibExtended.dll_ with the _dumpbin_ command and make sure `UELibExtended_CountActors` is exposed.
1. Copy `UELibExtended\Source\UELibExtended\UELibExtendedApi.h` to `UELibApp\UELib\include`
1. Copy `UELibExtended\Binaries\Win64\UELibExtended.lib` to  `UELibApp\UELib\lib`.
1. Copy __all__ _.dll_ files from `UELibExtended\Binaries\Win64` to `UELibApp\x64\Release` and run the project.
1. Compile `UELibApp` project and run it. If there are some errors make sure that you've copied `UELibExtendedApi.h`, `UELibExtended.lib` and `UELibExtended.dll` files as described above.
1. Modify the `UELibApp.cpp` to include the new `UELibExtendedApi.h` header. Modify Windows Message Processing callback to show the actor count on right mouse button click:

    ```C++
    // include new header
    #include "UELibExtendedApi.h"

    // a lot of code here... 

    // modify WM_RBUTTONUP in WndProc function
    case WM_RBUTTONUP:
    {
        OutputDebugStringVar(L"Actor count :%d", UELibExtended_CountActors());
        return UELibrary_WndProc(hWnd, message, wParam, lParam);
    }
    break;
    ```

1. Compile `UELibApp` project and run it. Click the right mouse button to spawn a dialog box.
