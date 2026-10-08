# TryHackMe: Room 404 — Production'da Unutulan .git Klasörü

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu challenge, bir geliştiricinin production'a deploy ederken `.git` klasörünü silmeyi unutmasının ne kadar tehlikeli olduğunu gösterdi. Hacker's Holiday Challenge serisinden çok kolay zorlukta bir oda ama öğrettiği ders büyük.

## İlk Bakış

Makineyi başlattığımda port 8080'de bir web uygulaması çalışıyordu. "The Byte Lotus guest-experience platform" adında bir otel misafir deneyimi platformu. Oda açıklamasında önemli bir ipucu vardı: "gece vardiyasındaki geliştirici web sitesinden fazlasını deploy etti."

Bu cümle hemen aklıma version control dosyalarını getirdi. Geliştiriciler bazen `.git`, `.env`, `.svn` gibi dosyaları production'a deploy ederken silmeyi unutuyorlar.

## .git Klasörünün Keşfi

Doğrudan tarayıcıdan `http://HEDEF_IP:8080/.git/` adresine gittim. Directory listing açıktı ve Git repository'sinin tüm iç yapısı görünüyordu: HEAD, refs, objects klasörleri. Bu, uygulamanın tüm kaynak kodunun ve commit geçmişinin açıkta olduğu anlamına geliyor.

Git objelerini tek tek okumaya çalışmak verimsiz olurdu çünkü Git verisini zlib ile sıkıştırılmış binary olarak tutuyor. Bunun yerine `git-dumper` aracını kullandım. Bu araç, HTTP üzerinden erişilebilen `.git` klasörünü otomatik olarak indirip yerel bir Git repository'si olarak yeniden oluşturuyor:

```
git-dumper http://HEDEF_IP:8080/.git/ dumped_repo
```

## Kaynak Kod Analizi

İndirilen repository'nin içine baktığımda birkaç dosya gördüm:

- `app.js` — uygulamanın JavaScript kodu
- `index.html` — ana sayfa
- `README.md` — iç dokümantasyon

README.md dosyasını okuduğumda içinde staging flag'i ve "launch öncesi kaldır" notu vardı. Geliştirici bu notu kaldırmayı ve `.git` klasörünü temizlemeyi unutmuş. Flag bulundu, challenge tamamlandı.

## Geliştirici Gözünden

Bu challenge teknik olarak basit ama verdiği mesaj çok önemli. Bir frontend/full-stack geliştirici olarak `.git` klasörünün production'da açık kalması beni özellikle etkileyen bir konu çünkü bu hatayı herkes yapabilir.

`.git` klasörü açık kalırsa:

- **Tüm kaynak kodu** sızar — proprietary code, iş mantığı, API endpoint'leri
- **Commit geçmişi** okunabilir — eski commit'lerde kaldırılmış şifreler, API key'leri, secret'lar bulunabilir
- **Geliştirici bilgileri** görünür — commit author isimleri ve e-posta adresleri

Bu nasıl engellenir:

```nginx
# Nginx — .git erişimini engelle
location ~ /\.git {
    deny all;
    return 404;
}
```

```apache
# Apache — .git erişimini engelle
<DirectoryMatch "^/.*/\.git/">
    Deny from all
</DirectoryMatch>
```

Ayrıca deploy sürecinde `.git` klasörünü otomatik olarak silmek veya CI/CD pipeline'ında kontrol etmek en güvenli yöntem. `.dockerignore` ve `.gitignore` bu konuda yardımcı olabilir ama asıl çözüm web sunucu konfigürasyonunda bu dizinlere erişimi engellemek.
