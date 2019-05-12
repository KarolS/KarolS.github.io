---
layout: post
title: "GraalVM's native-image on Windows 10 x64 – guide and unscientific benchmarks"
date: 2019-05-12 19:20
categories: [programming, java, scala, "graal-vm"]
---

Short-lived programs on JVM have a huge disadvantage – they start slowly and a huge part of their code runs interpreted before the JIT compiler kicks in. The solution is to compile the program to native code.

GraalVM is a new polyglot virtual machine, but it also comes with a decent native compiler for JVM programs, called `native-image`. My initial tests showed that it is worth the hassle for short-lived programs, like compilers and other command-line tools.

I compiled the [Millfork compiler](https://github.com/KarolS/millfork) with `native-image` and got huge speedups and the resulting executable runs on Windows out of the box. Here is how I did it.

<!-- more -->

## Prerequisites

I will assume that your program is of size similar to the Millfork compiler, which is about 11 MB. You will need the following:

* a machine running your target operating system with at least 16 GB of RAM – no, this is not an exaggeration (I used Windows 10 x64)

* a machine running Linux (it may be the same machine if you're targeting Linux, or another machine; it can be less beefy, 2 GB of RAM should be enough)

* copies of GraalVM and `native-image` on both those systems (on Windows, native-image comes with GraalVM; on Linux and macOS, you'll need to install it using the `gu` utility); get it here: https://github.com/oracle/graal/releases

* a fat jar with your program, compatible with Java 8 (your program should run with just `java -jar myprogram.jar`; GraalVM doesn't support newer Javas yet)

* on Linux or macOS: a relatively recent GCC

* on Windows: a copy of Windows 7 SDK (see instructions below)

I will assume that your GraalVM is installed in `/graalvm` on both machines and `native-image` is available.

## Installing Windows 7 SDK (Windows only)

This is the tricky part. I have only done this on Windows 10 x64, so I will only list the steps necessary on that platform. Other platforms may work differenly:

* download the SDK file [`GRMSDKX_EN_DVD.iso` from Microsoft](https://www.microsoft.com/en-us/download/details.aspx?id=8442)

* mount the ISO image, as for example E:

* uninstall Microsoft Visual C++ 2010 Redistributables – yes, this is necessary!

* run `E:\Setup\SDKSetup.exe`

* follow the instructions, install at least the headers, the compilers and the redistributables

## Analysing the program

First, you need to figure out which objects in your program are accessed via reflection and which resources in the jar are loaded. To figure this out automatically, run your program on your **Linux** machine with the following options:

```
/graalvm/bin/java -agentlib:native-image-agent=config-output-dir=/path/to/config-dir/ -jar myprogram.jar
```

where `/path/to/config-dir/` is a path to a directory where GraalVM should store data gathered during the run.
The run should cover as much code paths as possible. If necessary, you can run the program again and merge the results, using:

```
/graalvm/bin/java -agentlib:native-image-agent=config-merge-dir=/path/to/config-dir/ -jar myprogram.jar
```

(Note "merge" instead of "output" in the option name.)

The generated output will be several JSON files. Inspect them and manually add missing entries to [`jni-config.json`](https://github.com/oracle/graal/blob/master/substratevm/JNI.md), [`reflect-config.json`](https://github.com/oracle/graal/blob/master/substratevm/REFLECTION.md), [`proxy-config.json`](https://github.com/oracle/graal/blob/master/substratevm/DYNAMIC_PROXY.md) and [`resource-config.json`](https://github.com/oracle/graal/blob/master/substratevm/RESOURCES.md) (the file names are links that lead to the official documentation).

## Building the native binary

Put the config files into the `META-INF/native-image` directory in the resources of your program and rebuild the jar (this allows you to skip adding those files to the command line options for `native-image`). Files `reflect-config.json` and `resource-config.json` are obligatory, `jni-config.json` and `proxy-config.json` are needed only if they're not empty.

On your **target** machine, free as much RAM as you can. If you're using Windows, use the _Windows SDK 7.1 Command Prompt_ instead of the normal command prompt.

In the command prompt, run:

```
/graalvm/bin/native-image -jar myprogram.jar
```

This step will take a while. Go make yourself a coffee.

You may see warnings about class locality, especially if the program was written in Scala. Ignore them.

If you see a message that `native-image` decided to create a fallback executable, then it means that you failed to provide the necessary config files. Unless you're fine with shipping an entire JVM with your exe, abort the compilation, fix the config files and try again. I have not tested fallback executables, so I can't provide any guidance on how to use them.

If the compilation succeeds, then congratulations, your program is now a standalone native binary. Next to your `myprogram.jar` you should see `myprogram.exe` (if on Windows).

## Testing

You should now test that your executable works. You may have problems with class loading (which means non enough entries in `reflect-config.json`), reflection (ditto), resource loading (`resource-config.json`). If any problems occur, go back to the analysing step and try to add the missing entries, either automatically or manually.

## Results

The Millfork compiler was a perfect target for `native-image`. It's a program that runs through a set series of stages once and in a fixed order. The native binary ended up being 38 MB, which is smaller than even the smallest official JVM, and that's without counting 11 MB of the original jar.

Speed increased. Launching the compiler without any command line options (which simply yields an error message) takes less than 0.1 seconds, while doing the same with a JVM takes about 1.9 seconds. Even a moderately long compilation task is faster with the native executable. Compiling [Space Poker](https://github.com/KarolS/spacepoker) at `-O1` takes about 1.0 seconds with the native executable and about 4.8 seconds with Java (either 8 or 11), and at `-O4` it takes about 6.3 seconds and about 10–11 seconds respectively.

Overall, the experiment was a success. Future releases of Millfork will be shipped as both a cross-platform jar and a Windows executable.

