---
title:  "8kSec BorderDroid Writeup"
date:   2025-09-12T16:34:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for BorderDroid challenge from 8kSec Android Labs. 

```
Goal:
Goal is to find a way to bypass BorderDroid's kiosk security mechanisms and deactivate it without usb interaction and hardcoded credentials.
```

## Static Analysis

![](/assets/8ksec_borderdroid_images/screen1.png)

BorderDroid app is a kiosk app. First, we gave all requested permissions(accesibility etc.), set a pin for it and we can use it as kiosk mode. On kiosk mode,navigation buttons are not working and we can only see a pin enterence screen but when we give a correct pin, app not accepting it and says its wrong. Lets start our analysis with jadx.

### Code Analysis

```xml
...
...
...
          <activity android:theme="@style/Theme.AppCompat.NoActionBar" android:name="com.eightksec.borderdroid.SplashActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:name="com.eightksec.borderdroid.PinEntryActivity" android:exported="false" android:windowSoftInputMode="adjustResize|stateVisible"/>
        <activity android:name="com.eightksec.borderdroid.DashboardActivity" android:exported="false" android:launchMode="singleTop"/>
        <activity android:name="com.eightksec.borderdroid.NothingHereActivity" android:exported="false" android:excludeFromRecents="true" android:launchMode="singleTask"/>
        <activity android:theme="@style/Theme.AppCompat.NoActionBar" android:name="com.eightksec.borderdroid.WipeTimerActivity" android:exported="false" android:excludeFromRecents="true" android:launchMode="singleTask"/>
        <activity android:theme="@style/Theme.AppCompat.Dialog" android:name="com.eightksec.borderdroid.CountdownTimerActivity" android:exported="false"/>
        <activity android:name="com.eightksec.borderdroid.YouAreSecureActivity" android:exported="false" android:launchMode="singleTask" android:lockTaskMode="if_whitelisted"/>
        <service android:label="@string/accessibility_service_label" android:name="com.eightksec.borderdroid.service.KioskAccessibilityService" android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE" android:exported="false">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService"/>
            </intent-filter>
            <meta-data android:name="android.accessibilityservice" android:resource="@xml/accessibility_service_config"/>
        </service>
        <service android:name="com.eightksec.borderdroid.service.HttpUnlockService" android:exported="false" android:foregroundServiceType=""/>
        <receiver android:name="com.eightksec.borderdroid.receiver.RemoteTriggerReceiver" android:enabled="true" android:exported="true">
            <intent-filter>
                <action android:name="com.eightksec.borderdroid.ACTION_PERFORM_REMOTE_TRIGGER"/>
            </intent-filter>
        </receiver>
...
...
...

```
App has lots of activities and but they are not exported. Only `RemoteTriggerReceiver` is exported.

SplashActivity(Main Activity):

![](/assets/8ksec_borderdroid_images/main.png)

On main activity, app check the current status(first launch or re-login) and start `PinEntryActivity` with mode number.

![](/assets/8ksec_borderdroid_images/pin_entry_activity.png)

This activity check current mode and if pin is true, calls goToDashboard method and its send intent for `DashboardActivity`.

![](/assets/8ksec_borderdroid_images/start_countdown.png)

On dashboard screen, when we press the `Start Security` button, app calls `startCountdown` method. This method start a 10 sec. countdown screen. 

After 10 sec, this methods called:
```
countdownLauncer onActivityResult ->
 m73lambda$onCreate$0$comeightksecborderdroidDashboardActivity -> 
 handleActionSuccess -> 
 startSecurity
```
`startSecurity` method sends intent for `YouAreSecureActivity` class. At this step, the kiosk is active and ask for a pin for unlock. However, even though we gave the correct pin, it does not accept it.

![](/assets/8ksec_borderdroid_images/always_wrong_pin.png)

`onNumpadClick` method only checks for pin lenght and only shows wrong pin message :/

### Kiosk Bypass Methods

#### 1. Volume Keys Combination

![](/assets/8ksec_borderdroid_images/onkeypress.png)

`YouAreSecureActivity` class have implemented onKeyDown callback function. This function called when a button pressed.
- https://developer.android.com/reference/android/view/KeyEvent.Callback#onKeyDown(int,%20android.view.KeyEvent)

This method add `24` or `25` to `volumeSequence` array for volume up and volume down button presses and calls `checkVolumeSequence` method.

```java
private final List<Integer> targetSequence = YouAreSecureActivity$$ExternalSyntheticBackport0.m(24, 25, 24, 25);
```
![](/assets/8ksec_borderdroid_images/check_volume_sequence.png)

This method checks if key press sequence is `UP DOWN UP DOWN` and if the sequence is correct, unlock the kiosk mode.

With this method we can bypass kiosk mode without usb or additional app interaction.

PoC video: [Youtube](https://youtu.be/5ZX40gZAbvY)

#### 2. Unlock over HTTP

When app starts kiosk, `setKioskState` method send intent for `HttpUnlockService` class. This class start http server on port 8080.

![](/assets/8ksec_borderdroid_images/http_response.png)

This server checks the requests. If the uri is `/unlock`, pin send as json data and request method is POST, it calls another method(broadcastVulnerableUnlockIntentWithPın). This method create intent for `RemoteTriggerReceiver` and send the pin value as extra data.

![](/assets/8ksec_borderdroid_images/remote_unlock.png)

This class verify the pin and if its correct, http server stopped and kiosk deactivated. This method is a remote attack surface for us. We can bruteforce the pin to this http server and unlock the kiosk mode but this server runs local. We need to port forward for accessing the server.

PoC Video: [Youtube](https://youtu.be/Y-5eBy5Eh-s)

#### 3. Exported "RemoteTriggerReceiver" Component 

Http server use `RemoteTriggerReceiver` to unlock kiosk and also that receiver is exported in AndroidManifest.xml file. We can trigger this receiver with an intent and we can use this for bypassing kiosk.

Here is the adb command:
```sh
$ adb shell am broadcast -a "com.eightksec.borderdroid.ACTION_PERFORM_REMOTE_TRIGGER" --es "com.eightksec.borderdroid.EXTRA_TRIGGER_PIN" "123123"
```
Or you can develop a app that sends that intent :) 

PoC Video: [Youtube](https://youtu.be/YTq6hvsbQEo)

### Final

Thats all, see you!