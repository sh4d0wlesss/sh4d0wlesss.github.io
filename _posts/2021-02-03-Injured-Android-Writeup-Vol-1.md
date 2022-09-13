---
title:  "[TR]Injured Android Writeup VOL-1"
date:   2021-02-03T00:48:00-0400
categories:
  - writeup
  - android
tags:
  - ctf
  - android
---

Merhaba, son günlerde kendimi mobil uygulama güvenliği konusunda geliştirmeye çalışmam sebebi ile bu konuda ctf'ler çözmeye başladım. Bu yazıda da Injured Android'in ilk partını yayınlıyorum. Geri kalan soruların bazılarını daha çözememiş olmam, bazılarında ise sorun yaşamış olmam sebebi ile ileriki bir tarihte onları da paylaşacağım inş :D
> Injured Android: https://github.com/B3nac/InjuredAndroid<br>Bu yazıdaki Versiyon: 1.0.10 Oct.19 versiyonu<br>Emulator: Genymotion<br>API: Android 7.0 

## XSS Test
Github sayfasında yazıldığına göre bu aşama flag içermiyor, eğlence amaçlı bir örnek koymak amacıyla eklenmiş.
```
XSSTEST is just for fun and to raise awareness on how WebViews can be made vulnerable to XSS.
```
İlgili koda jadx ile bakarsak:
```java
//b3nac.injuredandroid.DisplayPostXSS
String stringExtra = getIntent().getStringExtra("com.b3nac.injuredandroid.DisplayPostXSS");
        WebSettings settings = webView.getSettings();
        d.b(settings, "vulnWebView.settings");
        settings.setJavaScriptEnabled(true);
        webView.setWebChromeClient(new WebChromeClient());
        webView.loadData(stringExtra, "text/html", "UTF-8");



//buraya bizim verdiğimiz girdi nasıl geliyor derseniz o işlem de b3nac.injuredandroid.XSSTextActivity aktivitesinde gerçekleşiyor. Tanımlanan intent ile putExtra metodu kullanılarak gidilen aktiviteye text yollanıyor.

public void submitText(View view) {
    Intent intent = new Intent(this, DisplayPostXSS.class);
    intent.putExtra("com.b3nac.injuredandroid.DisplayPostXSS", ((EditText) findViewById(R.id.editText)).getText().toString());
    startActivity(intent);
}

```
vulnWebView ayarlarından javascript enable edilmiş, bu sayede verdiğimiz girdi ile burada javascript çalıştırabiliriz. Herhangi bir filtre vs de uygulanmadığına göre bir xss payloadı deneyelim.
```js
<script>alert("shadow")</script>
```

## 1. FLAG ONE - LOGIN
Bu aşamada bizden flagi bulup submit etmemizi istiyor. Aşağıya eklenen ünlem butonuyla hint alınıyor sanırım ama bu yazıda onu kullanmadan çözmeye çalışacağım(Gerçi ister istemez kodda görüyorsunuz :D ). İlgili koda bakalım.
```java
//b3nac.injuredandroid.FlagOneLoginActivity
public final void submitFlag(View view) {
        EditText editText = (EditText) findViewById(R.id.editText2);
        d.b(editText, "editText2");
        if (d.a(editText.getText().toString(), "F1ag_0n3")) {
            Intent intent = new Intent(this, FlagOneSuccess.class);
            new FlagsOverview().J(true);
            new j().b(this, "flagOneButtonColor", true);
            startActivity(intent);
        }

```
Eğer girdiğimiz input "F1ag_0n3" ise intent ile FlagOneSuccess sınıfını çağırıyor. 

Şimdi burada kafaya takılabilecek sorun şu: biz bu fonksiyonun strcmp tarzı bi şey olduğunu nerden anladık. Kod obfuscate edildiği için bize direkt olarak fonksiyon adlarını yazmak yerine harf falan yazıyor. biz bu fonksiyonun gerçekte ne yaptığına bakmak için tanımlandığı yere bakacağız. Koddaki fonksiyon adı olan "a" harfine sağ tıklayıp "go to decleration" dersek bizi tanımlandığı yere götürür. Koda bakarsak bir string karşılaştırma işlemi olduğunu zaten anlıyoruz:
```java
    public static boolean a(Object obj, Object obj2) {
        return obj == null ? obj2 == null : obj.equals(obj2);
    }
``` 
## 2. FLAG TWO - EXPORTED ACTİVİTY
Bu aşamada bizden main activity bypass edip export edilmiş olan başka bir aktivite çağırmamızı istiyor. Export edilen aktivite dediğimiz olayı kısaca açıklayayım. Eğer bir aktivite için AndroidManifest.xml dosyasında 
```java
android:exported="true"
```
şeklinde bir tanımlama yapmışsa veya bir intent filter tanımlamışsa bu aktivite export edilmiş oluyor. Bu sayede intent kullanarak bu aktiviteyi  başka bir yerden çağırabiliyoruz. Çok daha kaliteli ve mantıklı anlatım için şu yazıyı okumanızı kesinlikle tavsiye ederim:<br>
https://blog.mzfr.me/posts/2020-11-07-exported-activities/


Şimdi öncelikle export edilen aktiviteleri bulmamız gerekiyor. Bu tanımlamaları Androidanifest.xml dosyasından bulabiliyoruz demiştik. 
```java
<activity android:name="b3nac.injuredandroid.QXV0aA" android:exported="true"/>

<activity android:name="b3nac.injuredandroid.b25lActivity" android:exported="true"/>
```
Bu yazdıklarım export edilen aktivitelerden bazıalrı, diğerleri başka flagler için ayarlandığı için sadece bunları deneyeceğim. Bu aşamada test için intent tanımladığınız bir POC uygulama hazırlayabileceğiniz gibi adb(android debug bridge) üzerinden activity manager ile de test edebilirsiniz. İşimize yarayacak parametreler için kaynak: <br>
https://developer.android.com/studio/command-line/adb#IntentSpec
```ps
> adb shell am start -n b3nac.injuredandroid/b3nac.injuredandroid.b25lActivity
Starting: Intent { cmp=b3nac.injuredandroid/.b25lActivity }
```
Burada -n parametresi ile gideceğimiz komponenti belirttik ve bize 2. flag geldi. Listede yazan diğer export edilmiş aktiviteyi verince bi login ekranı geliyor, demek ki o başka flagin adımıymış :)


## 3. FLAG THREE - RESOURCES
Koduna bakalım:
```java
public final void submitFlag(View view) {
        EditText editText = (EditText) findViewById(R.id.editText2);
        d.b(editText, "editText2");
        if (d.a(editText.getText().toString(), getString(R.string.cmVzb3VyY2VzX3lv))) {
            Intent intent = new Intent(this, FlagOneSuccess.class);
            new FlagsOverview().L(true);
            new j().b(this, "flagThreeButtonColor", true);
            startActivity(intent);
        }
    }
```
Bir önceki aşamalarda gördüğümüz d.a fonksiyonu yani string compare fonksiyonuna Resourcelardan aldığı stringi veriyor ve bizim vereceğimiz input ile karşılaştırıyor. 
Jadx'de values altından string.xml bulunamıyor. Obfuscated kodlarda bunu bulmak için "Resources -> resources.arsc -> res -> values -> strings.xml" dosyasına bakıyoruz. Koddaki değeri dosyada arattığımızda flag çıkıyor:
```
<string name="cmVzb3VyY2VzX3lv">F1ag_thr33</string>

Flag: F1ag_thr33
```
Kaynaklar:
- https://stackoverflow.com/questions/33195571/obfuscating-strings-xml-file-in-android-app
- https://stackoverflow.com/questions/27548810/android-compiled-resources-resources-arsc
- Bonus: https://rammic.github.io/2015/07/28/hiding-secrets-in-android-apps/

## 4. FLAG FOUR - LOGIN 2
Bu aşamada da bizden flagi girmemiz bekleniyor. Koda bakalım:
```java
//b3nac.injuredandroid.FlagFourActivity

public final void submitFlag(View view) {
        EditText editText = (EditText) findViewById(R.id.editText2);
        d.b(editText, "editText2");
        String obj = editText.getText().toString();
        byte[] a2 = new g().a();
        d.b(a2, "decoder.getData()");
        if (d.a(obj, new String(a2, d.p.c.f3141a))) {
            Intent intent = new Intent(this, FlagOneSuccess.class);
            new FlagsOverview().I(true);
            new j().b(this, "flagFourButtonColor", true);
            startActivity(intent);
        }
    }
```
Kodun 4. satırında bizden aldığı stringiobj stringine atıyor. Ardından a2 isimli btye arrayine de g().a() metodundan gelen değeri atıyor. Bu değerin ne olduğuna bakarsak karşımıza bir base64 ifade çıkmakta:
```java
//b3nac.injuredandroid.g

private byte[] f1911a = Base64.decode("NF9vdmVyZG9uZV9vbWVsZXRz", 0);

    public byte[] a() {
        return this.f1911a;
    }

```
Burada base64 değeri decode edip return ediyor. Ardından aldığımız bu değeri bizim verdiğimiz inpu ile karşılaştırıyor. Base64 decode edip verirsek flagi alıyoruz.
```bash
sh4d0w@faruk-pc:/mnt/c/Users/Faruk Arslan$ echo "NF9vdmVyZG9uZV9vbWVsZXRz" | base64 -d
4_overdone_omelets  <-- flag
```

## 5.FLAG FİVE - EXPORTED BROADCAST RECEIVER
Koda bakalım:
```java
//b3nac.injuredandroid.FlagFiveActivity
private FlagFiveReceiver u = new FlagFiveReceiver();

```
Kodda en başta bir broadcast receiver tanımlanmış. Broadcast receiver ile sistemden gelecek olan broadcast çağrılarını dinleyip ona göre işlem yapıyoruz, fonksiyon tetikliyoruz diyebiliriz. Ardından sendBroadcast metodu ile broadcast yapılıyor. <br>
b3nac.injuredandroid.FlagFiveReceiver sınıfında da diğer kodda tanımlanmış olan intent ile gelen broadcastlerin sayısına bakıp ona göre bize flag verriyot. 3 kez kodu tetiklediğimizde flag çıkıyor.

## 6. FLAG SIX - LOGIN 3
 Bu aşamada da bizden direkt olarak flag istiyor.Koda bakalım:
 ```java
 //b3nac.injuredandroid.FlagSixLoginActivity
public final void submitFlag(View view) {
        EditText editText = (EditText) findViewById(R.id.editText3);
        d.b(editText, "editText3");
        if (d.a(editText.getText().toString(), k.a("k3FElEG9lnoWbOateGhj5pX6QsXRNJKh///8Jxi8KXW7iDpk2xRxhQ=="))) {
            Intent intent = new Intent(this, FlagOneSuccess.class);
            FlagsOverview.D = true;
            new j().b(this, "flagSixButtonColor", true);
            startActivity(intent);
        }
    }
 ```
 Bizden aldığı string ile k.a() fonksiyonundan dönen değer ile karşılaştırıyor. Bu fonksiyona parametre olarak da uzunca bir string yolluyor.Bu fonksiyonun ne yaptığına bakarlım.
 ```java
//b3nac.injuredandroid.k
public static String a(String str) {
        if (c(str)) {
            try {
                SecretKey generateSecret = SecretKeyFactory.getInstance("DES").generateSecret(new DESKeySpec(f1917a));
                byte[] decode = Base64.decode(str, 0);
                Cipher instance = Cipher.getInstance("DES");
                instance.init(2, generateSecret);
                return new String(instance.doFinal(decode));
            } catch (InvalidKeyException | NoSuchAlgorithmException | InvalidKeySpecException | BadPaddingException | IllegalBlockSizeException | NoSuchPaddingException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println("Not a string!");
            return str;
        }
    }
 ```
 DES falan görünce encrypt decrypt vs aklımıza geliyor tabii. a fonksiyonu bir decrypttion işlemi yapıyor aslında ama nasıl yaptığı pek de umrumuzda değil çünkü vereceğimiz input ve kullanılan fonksiyon belli olduğu için frida ile fonksiyonu hooklayıp değeri konsola loglatabiliriz. Bu aşamada başta baya bi zorlandım çünkü kullandığım emülatör versiyonunda bir türlü beceremedim muhtemelen benim bir hatamdan vs kaynaklı :D Ama en son 7.0 versiyonlu bir android emulatörde frida ile hooklamayı başardım.
 _Not: Fonksiyonun aslında ne yaptığını vs ayrıntılı incelemek isterseniz İnjuredAndroid'in eski sürümlerini indirip bakmanızı tavsiye ederim. Obfuscated olunca bazen kafa karışabiliyor :)_

 Frida scriptimiz şu şekilde:
 ```js
console.log("Script loaded successfully ");
Java.perform(function x() {
    console.log("Inside java perform function");
    var my_class = Java.use("b3nac.injuredandroid.k");
    
    var string_class = Java.use("java.lang.String");

    my_class.a.overload("java.lang.String").implementation = function (x) { 
        var my_string = string_class.$new("k3FElEG9lnoWbOateGhj5pX6QsXRNJKh///8Jxi8KXW7iDpk2xRxhQ==");
        console.log("Original arg: " + x);
        var ret = this.a(my_string);
        console.log("Return value: " + ret);
        console.log("disaridayim")
        return ret;
    };
    
});
 ```
 Bu scripti başka bir writeuptan alıp değiştirdim. Frida için hazır template vs de bulup editleyebilrisiniz. Frida konularına başka yazılarda daha ayrıntılı ve adım adım anlatımlı bir şekilde değinmeye çalışacağım. Neyse, bu kodda da ilk olarak "k" sınıfından bir fonksiyon overload edeceğimiz için sınıfı bir değişkene atıyoruz(referans gibi). Ardından 
 ```js
 my_class.a.overload("java.lang.String").implementation = function (x) {
 ``` 
 ile decrypt fonksiyonumuzu overload ediyoruz. Neden sadece implement yazmadık vs konularına da ileriki yazılarda değineceğim. Ardından decrypt edeceğimiz stringi tanımlayıp fonksiyona veriyoruz ve sonuçları logluyoruz. Çalıştıralım:
 ```
 [sh4d0wless@paradise flag6]$ frida -l deneme.js -U -f b3nac.injuredandroid --no-pause
     ____
    / _  |   Frida 14.2.8 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Spawning `b3nac.injuredandroid`...                                      
Script loaded successfully 
Spawned `b3nac.injuredandroid`. Resuming main thread!                   
[Google Nexus 6::b3nac.injuredandroid]-> Inside java perform function
icerdeyim
Original arg: k3FElEG9lnoWbOateGhj5pX6QsXRNJKh///8Jxi8KXW7iDpk2xRxhQ==
Return value: {This_Isn't_Where_I_Parked_My_Car}
disaridayim
Script loaded successfully 
Inside java perform function
 ```
 Scripti çalıştırdıktan sonra uygulamamız açılıyor. İlgili adımı açıp bi string sallayıp verdiğimizde fonksiyon tetikleniyor ve orjinal fonksiyon yerine bizim hook ile değiştirdiğimiz fonksiyon çalışıyor ve flag geliyor :)
 ```
 Flag: {This_Isn't_Where_I_Parked_My_Car}
 ```
## 7. FLAG SEVEN - SQLITE
Bu aşamada bizden hem bir flag hem de bir parola istiyor. Koda bakarsak:
```java
//b3nac.injuredandroid.FlagSevenSqliteActivity
private final String v = "c3FsaXRl";
private final String w = "ZjFhZy1wYTU1";
private byte[] x = Base64.decode("c3FsaXRl", 0);
private byte[] y = Base64.decode(this.w, 0);
```
Buradaki değerleri base64 decode edince şu sonuçlar çıkıyor:
```
x = "sqlite"
y = "f1ag-pa55"
```
Ardından diğer satırlara baktığımızda karşımıza başka base64'ler çıkmaya devam ediyor.Bunların da decode edilmiş hali şu şekilde:
```java
SQLiteDatabase writableDatabase = this.u.getWritableDatabase();
        ContentValues contentValues = new ContentValues();
        contentValues.put("title", Base64.decode("VGhlIGZsYWcgaGFzaCE=", 0));
        contentValues.put("subtitle", Base64.decode("MmFiOTYzOTBjN2RiZTM0MzlkZTc0ZDBjOWIwYjE3Njc=", 0));
        writableDatabase.insert("Thisisatest", null, contentValues);
        contentValues.put("title", Base64.decode("VGhlIGZsYWcgaXMgYWxzbyBhIHBhc3N3b3JkIQ==", 0));
        contentValues.put("subtitle", h.c());
```
```
VGhlIGZsYWcgaGFzaCE=  -> "The flag hash!"
MmFiOTYzOTBjN2RiZTM0MzlkZTc0ZDBjOWIwYjE3Njc= -> "2ab96390c7dbe3439de74d0c9b0b1767"

VGhlIGZsYWcgaXMgYWxzbyBhIHBhc3N3b3JkIQ== -> "The flag is also a password!"
```
Koddan da anlaşılacağı üzere bir database oluşturulup buraya veriler yazılıyor. Flag hash olarak verilen değeri Google'da aratarak md5 olduğunu bulabilirsiniz.
```
2ab96390c7dbe3439de74d0c9b0b1767 : hunter2
```
Kodda ikinci olarak eklenen subtitle değeri için h.c() metodundan dönen değeri alıyor. Bu değere gidip baktığımızda:
```java
//b3nac.injuredandroid.h
private static String f1914c = "9EEADi^^:?;FC652?5C@:5]7:C632D6:@]4@>^DB=:E6];D@?";
...
...

    static String c() {
        return f1914c;
    }

```
Bu değer yine bir encoding vs uygulanarak elde edilmiş bir değer gibi gözükmekte. Nasıl encode edildiğini bulmak için cyberchefte deneme yaptım ancak bulamadım. Ardından koddaki hintlere bakınca "Not all encodings are the same, some need to be rotated." yazısını gördüm. Rotate dediği için gibi rot13 rot47 denemelerini yaparak rot47 ile sonuca ulaşabildim.
```
https://injuredandroid.firebaseio.com/sqlite.json
```
Bu adrese gidince karşımıza flag çıkmakta. Sonuç olarak:
```
Flag: "S3V3N_11"
Parola: "hunter2"
```
şeklinde cevapları girdiğimizde başarılı bir şekilde bu aşamayı da tamamlamış oluyoruz.

_Not: Tüm bu işlemleri yapmak yerine bu aktiviteyi çalıştırıp adb shell ile ilgili klasöre giderseniz orada sqlite dosyalarını görebilirsiniz. O dosyaları adb pull ile çekip bir sql görüntüleyicide açarsanız aynı değerleri hash şeklinde oradan kolayca bulabilirsiniz._
```
vbox86p:/data/data/b3nac.injuredandroid/databases # ls
Thisisatest.db  Thisisatest.db-journal
```

## 8. FLAG EIGHT - AWS
Sorunun adından da anlaşıldığı üzere bu soruda aws(Amazon Web Services) ile ilgili bi şey bulmamız bekleniyor. Koduna baktığımda işe yarar pek bir şey bulamayıp önceki adımlarda yaptığım gibi strings.xml'e baktım
```
AWS_ID = AKIAZ36DGKTUIOLDOBN6
AWS_SECRET = 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
```
aws diye aratınca karşımıza zaten credentialler çıkıyor. Bunları nasıl kullanacağız derseniz aws'nin komut satırından kullanmak için bir aws-cli isimli aracı var.
- https://aws.amazon.com/cli/

Bunu kurduktan sonra elde ettiğimiz key ile konfigure etmemiz lazım.
```
[sh4d0wless@paradise ~]$ aws configure
AWS Access Key ID [None]: AKIAZ36DGKTUIOLDOBN6
AWS Secret Access Key [None]: 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
Default region name [None]: 
Default output format [None]:
```
Ardından ne yapacağımı bilemedim ancak bug bounty blog paylaşımlarından vs. s3 bucket diye bir şey gördüğüm için onu araştırdım.
- https://aws.amazon.com/s3/ <br>
Şimdi bizim bu girdiğimiz hesapta s3 bucket var mı diye bakalım:
- https://docs.aws.amazon.com/cli/latest/reference/s3/ls.html

```
[sh4d0wless@paradise ~]$ aws s3 ls
2020-01-11 04:37:02 injuredandroid
```
Çıkan sonuca bir ls daha attığımızda flag geliyor.
```
[sh4d0wless@paradise ~]$ aws s3 ls s3://injuredandroid
2020-01-11 04:47:15         19 C10ud_S3cur1ty_lol


Flag: C10ud_S3cur1ty_lol
```
## 9. FLAG NINE - FIREBASE

Bu aşamada da firebase'e dair bir şeyler bulmamız gerekiyor. Koda bakalım:
```java
public FlagNineFirebaseActivity() {
        byte[] decode = Base64.decode("ZmxhZ3Mv", 0);
        this.v = decode;
        d.m.b.d.b(decode, "decodedDirectory");
        Charset charset = StandardCharsets.UTF_8;
        d.m.b.d.b(charset, "StandardCharsets.UTF_8");
        this.w = new String(decode, charset);
        f b2 = f.b();
        d.m.b.d.b(b2, "FirebaseDatabase.getInstance()");
        d d2 = b2.d();
        d.m.b.d.b(d2, "FirebaseDatabase.getInstance().reference");
        this.x = d2;
        d h = d2.h(this.w);
        d.m.b.d.b(h, "database.child(refDirectory)");
        this.y = h;
    }

```
Şimdi burayı tam anlayamadım, sanırım firebase instance2ı için bi referans oluşturmuş. Verilen base64 stringi de decode edince 
```
ZmxhZ3Mv -> flags/
```
directorysi geliyor. Verilen hintlere bakarsak 
```
Use the .json trick with database url
```
 demiş. Bu database url'ini bulmak için strings.xml'e bakarsak:

```
<string name="firebase_database_url">https://injuredandroid.firebaseio.com</string>

Url: https://injuredandroid.firebaseio.com
```
çıkıyor. firebase databaselerde (tam olarak bilmesem de) sonuna .json ekleyip denemeler yaparak arka taraftaki konfigurasyonu anlayabiliyoruz.
- Örnek: https://medium.com/@danangtriatmaja/firebase-database-takover-b7929bbb62e1

Ancak bizim örneğimizde permission denied dönüyor. Hintte söylendiği gibi flags directorysi ile denersek:
- https://injuredandroid.firebaseio.com/flags.json
```
"[nine!_flag]"
```
Bunu direk flag olarak submit edemiyoruz.
```java
public final void submitFlag(View view) {
        EditText editText = (EditText) findViewById(R.id.editText2);
        d.m.b.d.b(editText, "editText2");
        byte[] decode = Base64.decode(editText.getText().toString(), 0);
        d.m.b.d.b(decode, "decodedPost");
        Charset charset = StandardCharsets.UTF_8;
        d.m.b.d.b(charset, "StandardCharsets.UTF_8");
        this.y.b(new b(this, new String(decode, charset)));
    }
```
Bizden aldığı stringi base64 decode ettikten sonra karşılaştırma yapıyor. Flagi base64 encode edip verirsek başarılı bir şekilde bu aşamayı da tamamlamış oluyoruz.
```
Flag: W25pbmUhX2ZsYWdd
```
## 10. FLAG TEN - ÇÖZEMEDİM :D

## 11. FLAG ELEVEN - DEEPLINKS
Bu aşamada manifest dosyasında bir intent filter tanımlanmış:
```java
activity android:label="@string/title_activity_deep_link" android:name="b3nac.injuredandroid.DeepLinkActivity">
            <intent-filter android:label="filter_view_flag11">
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="flag11"/>
            </intent-filter>
            <intent-filter android:label="filter_view_flag11">
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="https"/>
            </intent-filter>
        </activity>
```
Burada tanımlanan deeplink scheme'ları ile bu aktiviteyi intent ile tetikleyebiliyoruz. Yani "https://" veya "flag11://" kullanabiliyoruz:
```bash
[sh4d0wless@paradise ~]$ adb shell am start -W -a android.intent.action.VIEW -d "https://"
Starting: Intent { act=android.intent.action.VIEW dat=https:///... }
Status: ok
Activity: b3nac.injuredandroid/.DeepLinkActivity
ThisTime: 228
TotalTime: 228
WaitTime: 232
Complete
# adb shell am -> activity manager kullan
# -W           -> işlemi tamamlamak için uygulamanın launc edilmesini bekle 
# -a           -> intent ile tetiklenecek action
# -d           -> data_uri tanımlaması
```
Bu şekilde çalıştırdığımızda bize deeplink aktivitesini açıyor. Bizden flag istiyor ama onu nereden alacağız? İpuçlarına baktığımızda:
```
This is one part of the puzzle.
Find the compiled treasure.
```
İlk başta native library dosyalarına vs baktım ama onlarda diğer challneglar ile ilgili fonksiyonlar vs vardı. Ardından bu ctf'in eski versiyonu için yazılmış bir writeupa baktım ve orada /res/values/ altında compiled dosyalar bulunmaktaymış. Ben apktool ile decompress ettiğimde orada değil de /assets/ klasörünün altında bu dosyaları buldum. Bu dosyayı çalıştırınca:
```
[sh4d0wless@paradise assets]$ ls
flutter_assets  meŉu  narnia.arm64  narnia.x86_64
[sh4d0wless@paradise assets]$ chmod +x meŉu 
[sh4d0wless@paradise assets]$ ./meŉu 
HIIMASTRING
```
çıktısını verdi. Bu cevabı flag olarak girince de kabul etti :) 
```
Flag: HIIMASTRING
```

