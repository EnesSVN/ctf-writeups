# TryHackMe: Simple CTF — İlk CTF ve SQLi ile Root

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu benim ilk CTF'im. CMS Made Simple üzerindeki bir SQL injection açığından başlayıp, hash kırma ve yanlış yapılandırılmış bir sudo izni üzerinden root yetkisine ulaştım.

## Keşif

IP elimize geçtiği an Nmap ile port taraması yaptım. 3 açık port buldum:

- **21 (FTP)** — vsFTPd 3.0.3, anonymous login açık
- **80 (HTTP)** — Apache 2.4.18
- **2222 (SSH)** — OpenSSH 7.2p2 (standart dışı port!)

FTP'ye anonymous olarak bağlandım ama içi boştu. Web tarafına baktım, ana dizinde sadece Apache varsayılan sayfası vardı.

## Web Keşfi ve Zafiyet Bulma

Gobuster ile gizli dizin taraması yaptım ve `/simple` dizinini buldum. Bu dizinde **CMS Made Simple** adında bir içerik yönetim sistemi çalışıyordu. Bilinen bir yazılım olduğu için searchsploit ile açık aradım:

```
searchsploit cms made simple
```

CVE-2019-9053 numaralı bir SQL injection açığı çıktı. Bu açık kimlik doğrulaması gerektirmiyordu, yani giriş yapmadan kullanabilirdim. Elimde hiçbir credential olmadığı için bu tam aradığım şeydi.

## SQL Injection ile Bilgi Sızdırma

Exploit'i çalıştırdığımda veritabanından kullanıcı adını (`mitch`), salt değerini ve şifre hash'ini sızdırmayı başardım. Bu time-based blind SQLi tekniği kullanıyordu, yani her karakter için sunucuya bir sorgu gönderip yanıt süresinden doğru/yanlış çıkarımı yapıyordu.

## Hash Kırma ve SSH Girişi

John the Ripper ile hash'i kırmaya çalıştım. VPN bağlantısında paket kaybı olduğu için hash bazen bozuk geldi ama sonunda şifreyi elde ettim.

SSH ile giriş yaptım. Burada önemli bir detay: SSH standart 22 portu yerine 2222'de çalışıyordu:

```
ssh mitch@HEDEF_IP -p 2222
```

Home dizininde user flag'i aldım.

## Yetki Yükseltme

Sisteme girdikten sonra ilk iş `sudo -l`. Bu komut, kullanıcının root olarak neleri çalıştırabileceğini gösterir. Mitch kullanıcısının **vim** editörünü root olarak ve şifre sormadan çalıştırabildiğini gördüm.

Vim içinden sistem komutu çalıştırılabilir. Root olarak açılan bir Vim'in başlattığı shell de root olur:

```
sudo vim -c ':!/bin/sh'
```

Root shell geldi, flag'i aldım. Makine tamamen ele geçirildi.

## Kapanış ve Geliştirici Gözünden

Bu makinede bir web uygulamasındaki SQL injection açığından başlayıp, hash kırma ve yanlış yapılandırılmış bir sudo izni üzerinden root yetkisine ulaştım.

Bir yazılımcı olarak bu açıkların nasıl önlenebileceğine bakmak istiyorum:

- **SQL Injection:** Kullanıcıdan gelen veri doğrudan SQL sorgusuna eklendiği için oluşuyordu. Parametreli sorgular (prepared statements) kullanılsaydı, bu açık mümkün olmazdı.
- **Sudo Yanlış Yapılandırması:** Vim gibi içinden komut çalıştırılabilen bir araca sudo izni vermek doğrudan root shell demek. GTFOBins sitesinden kontrol edilmeli, bu tür araçlara sudo izni verilmemeli.
- **SSH Port Değiştirme:** SSH'ı 2222'ye taşımak güvenlik sağlamaz. `-p-` ile tüm portlar taranır ve bulunur. Gerçek güvenlik key-based authentication ve fail2ban ile sağlanır.
