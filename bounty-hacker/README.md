# TryHackMe: Bounty Hacker — FTP'den Root'a

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine, bir FTP sunucusunda bırakılmış dosyalardan yola çıkarak SSH ile giriş yapıp root yetkisine ulaştığım bir CTF.

## Keşif Yapma

Her zaman olduğu gibi ilk adım Nmap ile port taraması. IP'yi aldığım an taramayı başlattım ve 3 açık port buldum:

- **21 (FTP)** — vsFTPd 3.0.5
- **22 (SSH)** — OpenSSH 8.2p1
- **80 (HTTP)** — Apache 2.4.41

Web tarafında düz bir sayfa vardı, ilginç bir şey yoktu. Ama FTP portu dikkatimi çekti çünkü anonymous login açık olabilir bunu Nmap çıktısından görüyoruz.

## FTP ile Bilgi Toplama

FTP'ye anonymous olarak bağlandım ve iki dosya buldum: `task.txt` ve `locks.txt`. Bu dosyaları indirip incelediğimde task.txt içinde sunucuyu kimin deploy ettiğini gördüm kullanıcı adı **lin**. locks.txt ise bir şifre listesiydi, yani bir wordlist.

Elimde bir kullanıcı adı ve bir şifre listesi var. SSH portu açık. Yapılması gereken şey belli.

## Hydra ile SSH Brute Force

Hydra aracını kullanarak lin kullanıcısı için SSH brute force yaptım. Wordlist olarak FTP'den indirdiğim locks.txt dosyasını verdim.

```
hydra -l lin -P locks.txt ssh://HEDEF_IP
```

Kısa sürede şifreyi buldu: `RedDr4gonSynd1cat3`. Burada önemli olan nokta şu: genel bir wordlist (rockyou.txt gibi) yerine hedeften elde ettiğim özel bir wordlist kullandım. Bu hem daha hızlı hem de daha etkili oldu.

## SSH ile Giriş ve User Flag

Bulduğum bilgilerle SSH üzerinden giriş yaptım. Home dizininde user.txt dosyasını bulup ilk flag'i aldım.

## Yetki Yükseltme (Privilege Escalation)

Sisteme girdikten sonra her zaman yaptığım ilk şey: `sudo -l`. Bu komut, kullanıcının root olarak neleri çalıştırabildiğini gösterir. Çıktıda lin kullanıcısının **tar** aracını root olarak ve şifre sormadan çalıştırabildiğini gördüm.

tar aslında dosya arşivleme aracı ama bir özelliği var: `--checkpoint-action` parametresi ile belirli aralıklarla harici komut çalıştırabiliyor. Root olarak çalışan bir tar'ın başlattığı komut da root olur.

GTFOBins'ten tar için privesc komutunu buldum:

```
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Root shell aldım! /root dizinindeki root.txt dosyasından son flag'i de aldım.

## Kapanış ve Geliştirici Gözünden

Bu makinede FTP'de bırakılmış dosyalardan yola çıkarak SSH brute force ile giriş yapıp, yanlış yapılandırılmış bir sudo izni üzerinden root yetkisine ulaştım.

Bir yazılımcı olarak bu açıkların nasıl önlenebileceğine bakmak istiyorum:

- **FTP Anonymous Erişim:** FTP'de anonymous login açık bırakılmış ve hassas dosyalar (kullanıcı adı, şifre listesi) herkesin erişimine sunulmuş. Anonymous erişim kapatılmalı veya en azından hassas dosyalar FTP dizininde tutulmamalı.
- **SSH Brute Force:** Şifre tabanlı SSH girişine karşı fail2ban gibi araçlarla rate limiting uygulanmalı. Daha iyisi, key-based authentication kullanılmalı.
- **Sudo Yanlış Yapılandırması:** tar gibi araçlara sudo yetkisi verilmeden önce GTFOBins kontrol edilmeli. İçinden komut çalıştırılabilen araçlara sudo izni verilmesi, doğrudan root shell demek.
