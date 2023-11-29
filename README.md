# WiX-v4-for-Flutter

When creating an [installer](https://github.com/BAA-Studios/SunnyFORM) for [CastFORM](https://github.com/BAA-Studios/CastFORM) using WiX v4, I found that documentation wasn't quite as readily available as I had hoped.

This repository will document what I've learnt from an amalgamation of the official documentation, trial and error, as well as a heck load of Googling.

Hopefully this helps add on to the publicly available documentation on WiX v4, and will go on to be of use/assistance for anyone wishing to use WiX v4
 to package their Flutter for Windows application.

## Quick Links
- [Official Docs](https://wixtoolset.org/docs/intro/)
- [Official Tutorial](https://www.firegiant.com/docs/wix/tutorial/)
- [Rob Mensching's VOD on how to install and import the extension](https://www.youtube.com/live/-1-72Py0GSM?si=gCTOtuOEt7KDDJvG)
- [Rob Mensching's VOD on how to customise the GUI](https://www.youtube.com/live/8eSS0DchoTY?si=n9Jv6eDOjCQHRmuL)
- Code examples: [`headless` branch of my installer](https://github.com/BAA-Studios/SunnyFORM/tree/headless)
- Code examples: [`main` branch of my installer](https://github.com/BAA-Studios/SunnyFORM/)

## Tech Stack

My test bench was set up with the following dependencies:  
- Visual Studio Community 2022 v17.8.1
- .NET SDK 8 (via Visual Studio)
- HeatWave for VS2022 v1.0.2 (Visual Studio extension)
  - This is the name on the Visual Studio marketplace
  - FireGiant which produces both WiX and HeatWave calls this version `HeatWave Community Edition` instead
- WiX Toolset v4 (HeatWave obtains this from nuget - no installation needed)

GUIDs were left to automatic generation; otherwise the `New-Guid` PowerShell cmdlet was used when explicit GUIDs was needed.


## How To Use HeatWave for Flutter Projects

### Minimum Viable Project - No GUI

These instructions are written for HeatWave v1.0.2, WiX v4.0.3 which are the latest versions at the time of writing.  
They assume system-wide installation, rather than user-only installation since the latter does not currently have documentation from FireGiant at the moment, and it's not required for my own workflow either.  
Following through will give you one single self-contained no-options no-GUI full-install `.msi` file as an MVP output. I have provided links to relevant documentation pertaining to GUI-customisation at the end of the section.

1. Open Visual Studio and create a new project. Set the build configuration to x64 (required for Flutter).
    - Select `WiX` under the languages list, and then the `MSI Package (WiX v4)` template
    - After clicking `Next`, set the project name and location and check the option to use directory for project and solution
      - If you don't check that last option, you'll end up with a folder structure like so: `.../solution/project/`, 
which may be suitable for large projects but unlikely to be necessary for our intents and purposes
    - After this you can hit `Create`
2. In the top toolbar go to `Project > Properties > Build`, and under `Preprocessor variables` set any user-defined variables you'd like to use
    - Refer to my `Installer.wixproj` file [here](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/Installer.wixproj) for an example
    - I use it to house the absolute paths for Fluttter build output and Visual C++ Redistributable libraries directories
3. `Package.wxs` should already be open for you. Modify the `Package` tag with the attributes you'd like to see in the installed app
    - E.g. `Name` is how it should be displayed in the Start Menu after installation
    - Version should be in the format `major.minor.build`, with the first 2 numbers being 0-255, and the build number being 0-65535
      - The automatically generated version number is in 4 parts, but you are *strongly advised* to stick to 3 part semver due to how Windows Installer handles versioning at a low level. If you'd like to be able to use a patch number, please watch [this](https://www.youtube.com/watch?v=s1ZdtkD5lZg) and [this](https://www.youtube.com/watch?v=vqiEVfeDjpw) to understand the caveats and possible headaches you might encounter.
    - Upgrade code is a GUID that lets Windows Installer identify different version of the same software
4. Nest the following line within the `Package` tags (see mine [here](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/Package.wxs) for reference): `<MediaTemplate EmbedCab="yes"/>`
    - This tells WiX to bundle the cabinet files inside of the `.msi` so that you only distribute one file as the installer
5. Create a new `Feature` tag to enclose the existing template feature; create another one in the same level as the along side `Main`. Feel free to name the IDs as you see fit (see mine [here](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/Package.wxs) for reference).
    - The feature tag should now be a tree with one parent and two children
    - I am repurposing the feature tag provided for the main application, and creating one in the same level for Visual C++ redistributables
6. Duplicate `ComponentGroupRef` tag for the other feature with a suitable ID for Visual C++ Redustributable libraries; similarly duplicate `ExampleComponents.wxs` and rename IDs to what you used in the corresponding `ComponentGroupRef` ID
    - See `Package.wxs` and `VRredist.wxs` in ours in the `headless` branch for reference
7. Use the `ComponentGroupRef` tag to create one component group reference for each folder you'd like to write files to (non-recursive)
    - See [`Package.wxs`](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/Package.wxs) and [`AppComponents.wxs`](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/AppComponents.wxs) for reference
8. Modify `INSTALLFOLDER` in `Folders.wxs` to suit your preferred install location
    - Supplying `ProgramFiles6432Folder` for `StandardDirectory` simply means to use either `C:\Program Files (x86)` or `C:\Program Files` as appropriate depending on build target
    - The template created by HeatWave concatenates company name and product name to make one folder; I split this up into a nested folder structure for my use case
9. Specify additional folders nested under `INSTALLFOLDER` according to your Flutter build
    - Refer to `build/windows/runner/Release` after running `flutter build windows` in your Flutter project
10. Flesh out application component groups - see [mine](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/AppComponents.wxs) for reference
    - To use preprocessor variables, see step 2
11. Repeat for Visual C++ Redistributable libraries
    - See my [`VRredist.wxs`](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/VCredist.wxs) for reference
12. In the top toolbar got to `Build > Build Solution`
13. Check the output and test that it installs your program correctly
    - It will not show up in the Start menu after installation
    - You can uninstall by going to `Settings > App`, then click on the ellipsis to the far right of the app name, and then `Uninstall`

The steps outlined above will generate a non-GUI `.msi` installer that you can now proceed to test. 
Once you have confirmed that you can get it working, you might want to start setting up a GUI interface for configuration. 

### GUI and Other QoL Features

There's a myriad of ways to set up a GUI interface for installation configuration options.

You will be using a WiX extension called "WixUI dialog library", which you can find the API reference for [here](https://wixtoolset.org/docs/tools/wixext/wixui/). You may also refer to either the source code in my working [installer repository](https://github.com/BAA-Studios/SunnyFORM) for examples, or, [Rob Mensching's VOD on how to install and import the extension](https://www.youtube.com/live/-1-72Py0GSM?si=gCTOtuOEt7KDDJvG) and [Rob Mensching's VOD on how to customise the GUI](https://www.youtube.com/live/8eSS0DchoTY?si=n9Jv6eDOjCQHRmuL) to see it in action.

Essentially, you pick one of the styles (i.e. how much granularity the user should have over configuration options), and override the strings and images used in the installer. If even more customisation is desired, you will need to copy-paste their `.wxs` to modify (and replace) the library components.

To have the application appear in the Start menu after installation, you'd need to create a shortcut in the `ProgramsMenuFolder` - refer to my [`AppComponents.wxs`](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/AppComponents.wxs) or watch [this](https://www.youtube.com/watch?v=U7MQCF5AZcw), [this](https://www.youtube.com/watch?v=x-E7g5H_1TA), and [this](https://www.youtube.com/watch?v=1kV4gk3tTzg) (3 VODs by Rob Mensching) for reference.

In a nutshell, you'll be using the `Shortcut` tag after specifying your `.exe` executable (i.e. entry point). This creates a shortcut to that executable in the Start menu folder in the file system, which is how Windows handles Start menu items under the hood.

You may also want a custom app icon for when your app is viewed in the `Apps` page of Windows' Settings app. For that, you may use something like this:  
```xml
<Property Id="ARPPRODUCTICON" Value="insert_icon_object_here" />
```
You can put it between the package tags in `Package.wxs` (something like [this](https://github.com/BAA-Studios/SunnyFORM/blob/headless/Installer/Package.wxs)).

---

## Common Errors and Gotchas

### ICE60 Warnings
You may come across ICE60 warnings as a result of Flutter's font files (like `MaterialIcons-Regular.otf`). This has to do with Windows Installer's behaviour at a low level:  
> ICE60 checks that files in the File table meet the following condition:
> 
> - If the file is not a font and has a version, then it must have a language.
> - ICE60 checks that no versioned files are listed in the MsiFileHash table.
> 
> \- [*Windows Installer Guide*](https://learn.microsoft.com/en-us/windows/win32/msi/ice60)

The conundrum I have is that I wish for Windows Installer to treat these as normal files, rather than install them to the Windows font folder. 
However, since these files are versioned, the first condition would require them to also have an encoding specified (which they can't).

Our solution to this is to set our Flutter-generated `.exe` executable as `keypath`, and any offending font files as companion files to it. 
This makes the font files companion references to the executable (and is upgraded every time the executable is upgraded), and thus exempt from 
the code page value requirement.

### Shortcuts - ICE43 and ICE57 Warnings

Honestly for this one... I just gave up and gave in, and set `Advertise="yes"` on the shortcut.

### Restricting Installer to 64-bit Windows 10/11 via VersionNT/VersionNT64

This oddity with VersionNT64 tripped me up for way too long. Flutter officially supports only 64-bit Windows 10/11, and one might think that querying for the VersionNT64 property would allow one to set platform checks.

This is what Microsoft's documentation have to say:  
> **VersionNT64 property**  
> The installer sets the VersionNT64 property to the version number for the operating system only if the system is running on a 64-bit computer. The property is undefined if the operating system is not 64-bit.  
> 
> The value is an integer: MajorVersion * 100 + MinorVersion.
> 
> \- https://learn.microsoft.com/en-us/windows/win32/msi/versionnt64

With this in mind, one might be led to believe that the Win32 call would result in VersionNT64 returning `1000` or more for 64-bit Windows 10+ and 64-bit Windows Server 2016+. 
However, if you are attempting this query during an `.msi` installation, VersionNT64 actually returns `603` for all Windows 8.1+ and Windows Server 2012r2+ OSes. 
According to Microsoft, this is a feature not a bug:  
> **VersionNT value for Windows 10, Windows Server 2016, and Windows Server 2019**  
> When you install a .msi installation package in Windows 10, Windows Server 2016 or Windows Server 2019, the VersionNT value is 603.  
> 
> This version numbering is by design. To maintain compatibility, the VersionNT value is 603 for Windows 10, Windows Server 2016, and Windows Server 2019.
> 
> \- https://learn.microsoft.com/en-US/troubleshoot/windows-client/application-management/versionnt-value-for-windows-10-server

My solution was to use `603`, and assume that there's no sane person using Windows 8.1 in 2023 that might want to use my application. Understandably this only barely circumvents the problem, and there might be legitimate reasons requiring you to check for an exact OS version - from the research I have done, a reliable way to do so is is via reflection from the Registry.

I note that WiX v4 documentation makes some mention of Burn's own VersionNT64 variables; I am unable to comment on how well this works and how to use it, 
since I am not using Burn bundles, given that my application is fairly small and simply structured. You may find the documentation for Burn [here](https://wixtoolset.org/docs/tools/burn/).

## Detours

This is just a description of how I ended up starting a WiX project for anyone who's curious.

Although I initially explored the possibility of using the installer creation template from the *Microsoft Visual Studio Installer Projects* extension, 
I was unable to get it working due to my relative inexperience with production C++ nor building Flutter via Visual Studio. 
It didn't help that this doesn't seem to be a particularly popular approach, and I could not find good documentation for this. 
Next, I considered using Windows Installer, which is what the Flutter team at Google recommends as second choice (their primary recommendation using the Microsoft Store for packaging). 
Microsoft's own documentation for this is an incredibly verbose set of API documentation steeped in jargon - I found them too difficult to read through. At the time, it also had minimal follow-along instructions, and the learning curve proved too steep. Searching around, I did find [this 75 page guide](https://www.itninja.com/static/c94ee3e6f937bcb62a8451cb94a0a206.pdf) written by ITNinja for an earlier version of Windows Installer - which is really great and much more digestible than Microsoft's official documentation - but unfortunately was still a bit much for me, considering my extremely small team size and product size.

In view of this, I opted for what I thought would be a more user-friendly option: the [WiX Toolset](https://github.com/wixtoolset/wix) (aka `Windows Installer XML`). WiX Toolset is an open source set of build tools for building Windows Installer packages. It's still Windows Installer (and/or MSBuild) under the hood, but with many extensions provided to automate some of the more tedious parts of Windows Installer package creation, and exposing functionality in text form via XML. While the learning curve is supposedly steeper than with [Inno Setup](https://jrsoftware.org/isinfo.php), I decided to give WiX a shot since it's able to generate `.msi` bundles (whereas Inno Setup produces `.exe` installers instead).

Taking inspiration from [this repository on WiX v3](https://github.com/zonble/flutter_wix_installer_example) by Weizhong Yang that I chanced upon during my research phase, 
I decided to continually update this README with instructions as I progress with my project, in hopes of providing more documentation for other members of the open source community whom may wish to also deploy Flutter applications using WiX.

## Disclaimer
This project serves as reference for myself and for anyone who wishes to use WiX for packaging Flutter for Windows applications.  
It is non-monetised, and provided as is. Every reasonable effort has been taken to ensure correctness and reliability of its contents. 
I will not be liable for any special, direct, indirect, or consequential damages or any damages whatsoever resulting from 
loss of use, data or profits, whether in an action if contract, negligence or other tortious action, arising out of or in connection with the use of its contents (in part or in whole).