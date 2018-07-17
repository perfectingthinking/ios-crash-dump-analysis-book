# Symbolification

This chapter explains crash dump symbolification.
We use the `icdab_planets` sample app to demonstrate a crash. @icdabgithub

When dealing with real world crashes, a number of different entities are involved.  These can be the end user device, the settings allowing the crash report to be sent back to Apple, the symbols held by Apple and your local development environment setup to mirror such a configuration.

In order to understand how things all fit together it is best to start from first principles and do the data conversion tasks yourself so if you have to diagnose symbolification issues, you have some experience with the technologies at hand.

## Build Process

Normally when you develop an app, you are deploying the Debug version of your app onto your device.  When you are deploying your app for testers, app review, or app store release, you are deploying the Release version of your app.

In both scenarios debug information is placed into the binary being generated.
This is called DWARF debugging information.

For Release builds, that information is then stripped out and placed into a DSYM file.
For Debug builds, it is left in.

The debugger can use debugging information in the binary when it sees a crash to
help us understand where the program has gone wrong.

When a user sees your program crash, there is no debugger.  Instead, a crash
report is generated.  This comprises the machine addresses where the problem was
seen.  A later phase, called symbolification, can convert the addresses
into meaningful source code references so long as an appropriate DSYM file exists.

Xcode is by default setup so that only DSYM files are generated for Release
builds, and not for Debug builds.

The reason why Debug builds just use the application binary with all the debug information built in is that the information is always available and consistent with the rest of the binary.  However it makes the binary much larger and allows reverse engineers to peek into your binary quite easily as if you had published the source code together with the program.

### Build Settings

From Xcode, in your build settings, searching for "Debug Information Format" we see the following settings:

Setting|Meaning|Usually set for target
--|--|--
DWARF|Debugging information built into the binary itself|Debug
DWARF with dSYM File|We get an extra file also generated with symbols|Release

In the default setup, if you run your debug binary on your device, launching it from the app icon itself then if it were to crash you would not have any symbols in the crash report.  This confuses many people.

Whilst you may have all the source code for your program, and DWARF data in the crashed binary, ReportCrash crash reporter only looks for DSYM files on your Mac in order to perform symbolification.

To avoid this problem, the sample app `icdab_planets` has been configured to have `DWARF with dSYM File` set for both debug and release targets.

## Observing a local crash

The `icdab_planets` program is designed to crash upon launch due to an assertion.

If the `DWARF with dSYM File` setting had not been made, we would get a partially symbolicated crash.

The crash report, seen from Windows->Devices and Simulators->View Device Logs,
would look like this (truncated for ease of demonstration)

```
Thread 0 Crashed:
0   libsystem_kernel.dylib        	0x0000000183a012ec __pthread_kill + 8
1   libsystem_pthread.dylib       	0x0000000183ba2288 pthread_kill$VARIANT$mp + 376
2   libsystem_c.dylib             	0x000000018396fd0c abort + 140
3   libsystem_c.dylib             	0x0000000183944000 basename_r + 0
4   icdab_planets                 	0x00000001008e45bc 0x1008e0000 + 17852
5   UIKit                         	0x000000018db56ee0 -[UIViewController loadViewIfRequired] + 1020

Binary Images:
0x1008e0000 - 0x1008ebfff icdab_planets arm64
  <9ff56cfacd66354ea85ff5973137f011>
   /var/containers/Bundle/Application/BEF249D9-1520-40F7-93F4-8B99D913A4AC/
   icdab_planets.app/icdab_planets
```

However with the setting in place, a crash would instead be reported as:

```
Thread 0 Crashed:
0   libsystem_kernel.dylib        	0x0000000183a012ec __pthread_kill + 8
1   libsystem_pthread.dylib       	0x0000000183ba2288 pthread_kill$VARIANT$mp + 376
2   libsystem_c.dylib             	0x000000018396fd0c abort + 140
3   libsystem_c.dylib             	0x0000000183944000 basename_r + 0
4   icdab_planets                 	0x0000000104e145bc -[PlanetViewController viewDidLoad] + 17852 (PlanetViewController.mm:33)
5   UIKit                         	0x000000018db56ee0 -[UIViewController loadViewIfRequired] + 1020
```

Lines 0, 1, 2, 5 are the same in both cases because our developer environment will
have the symbols for the iOS release under test.  In the second case Xcode will
look up the DSYM file to clarify line 4.  It tells us this is line 33 in file
PlanetViewController.mm.  This is:

```
assert(pluto_volume != 0.0);
```

## DSYM structure

The DSYM file is strictly speaking a directory hierarchy:
```
icdab_planets.app.dSYM
icdab_planets.app.dSYM/Contents
icdab_planets.app.dSYM/Contents/Resources
icdab_planets.app.dSYM/Contents/Resources/DWARF
icdab_planets.app.dSYM/Contents/Resources/DWARF/icdab_planets
icdab_planets.app.dSYM/Contents/Info.plist
```

It is just the DWARF data normally put into the debug binary but copied into a separate file.

From looking at your build log you can see how the DSYM was generated.
It is effectively just `dsymutil path_to_app_binary -o output_symbols_dir.dSYM`

## Manual Symbolification

In order to help us get comfortable with crash dump reports, we can demonstrate
how the symbolification actually works.  In the first crash dump we want to understand:

```
4   icdab_planets                 	0x00000001008e45bc 0x1008e0000 + 17852
```

If we knew accurately the version of our code at the time of the crash we can
recompile our program but with the DSYM setting switched on and then get a
DSYM file after the original crash.  It should line up almost exactly.

The crash dump program tells us where the program was loaded in memory at the
time of the problem.  This is important because it is a master base offset from
which all other address (TEXT) locations are relative to.  At the bottom of the crash
dump we have line `0x1008e0000 - 0x1008ebfff icdab_planets`

So the icdab_planets binary starts at location `0x1008e0000`

Running the lookup command `atos` symbolicates the line of interest:
```
# atos -arch arm64 -o ./icdab_planets.app.dSYM/Contents/Resources/DWARF/
icdab_planets -l 0x1008e0000 0x00000001008e45bc
-[PlanetViewController viewDidLoad] (in icdab_planets)
 (PlanetViewController.mm:33)
```

The crash reporter tool fundamentally just uses `atos` to symbolicate the
crash report, as well as providing other system related information.

Symbolification is described further by an Apple Technote in case you want to get into it in more detail. @tn2123