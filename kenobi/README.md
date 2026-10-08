# TryHackMe: Kenobi — SMB'den SSH Key'e, Root'a PATH Manipulation ile

Siber güvenlik tarafına geçiş yapan bir yazılımcıyım. Bu makine bana SMB enumeration, ProFTPD exploit ve PATH manipulation ile yetki yükseltme gibi konuları tek bir saldırı zincirinde gösterdi. Star Wars temalı ama içindeki teknikler gayet gerçekçi.

## Keşif

IP'yi aldığım an Nmap ile tarama başlattım. 7 açık port çıktı: 21 (FTP — ProFTPD 1.3.5), 22 (SSH), 80 (HTTP), 111 ve 2049 (NFS/RPC), 139 ve 445 (SMB). Bu kadar çok açık port görmek, saldırı yüzeyinin geniş olduğu anlamına geliyor. Soru şu: hangisinden girmeliyim?

SMB ve NFS dikkatimi çekti. İkisi de dosya paylaşım protokolleri ve yanlış yapılandırılmışlarsa içlerinde hassas bilgiler bulunabilir.

## SMB ile Bilgi Toplama

SMB paylaşımlarını listelemek için smbclient kullandım:

```
smbclient -L \\HEDEF_IP -N
```

`-N` flag'i şifresiz (anonim) bağlantı demek. 3 paylaşım çıktı ve bunlardan biri `anonymous` adında. İçine girip baktığımda `log.txt` adında bir dosya buldum. Bu dosya altın değerindeydi çünkü içinde Kenobi kullanıcısının SSH private key'inin yolunu (`/home/kenobi/.ssh/id_rsa`) ve ProFTPD'nin yapılandırma bilgilerini gördüm.

Elimde bir SSH key yolu var ama o dosyaya doğrudan erişemiyorum. Bu key'i bir şekilde erişebileceğim bir yere taşımam lazım.

## ProFTPD Exploit ile SSH Key'i Kopyalama

ProFTPD versiyonunun 1.3.5 olduğunu biliyorum. Searchsploit ile baktığımda `mod_copy` modülünde bir zafiyet buldum. Bu modül, kimlik doğrulaması gerektirmeden sunucu üzerinde dosya kopyalama yapmanıza izin veriyor: `SITE CPFR` (copy from) ve `SITE CPTO` (copy to) komutlarıyla.

Netcat ile FTP portuna bağlanıp SSH key'ini NFS ile erişebileceğim `/var/tmp` dizinine kopyaladım:

```
nc HEDEF_IP 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

Bu teknik beni çok etkiledi. Bir FTP sunucusunda kimlik doğrulamadan geçmeden dosya kopyalayabilmek, düşündüğünüzde ne kadar tehlikeli bir yanlış yapılandırma.

## NFS ile Key'e Erişim ve SSH Girişi

Daha önce Nmap taramasında NFS servisini görmüştüm. `showmount` ile kontrol ettiğimde `/var` dizininin herkese açık paylaşıldığını gördüm. Bu dizini kendi makineme mount ettim ve kopyaladığım id_rsa dosyasına ulaştım:

```
showmount -e HEDEF_IP
mkdir /mnt/kenobi
mount HEDEF_IP:/var /mnt/kenobi
cp /mnt/kenobi/tmp/id_rsa .
chmod 600 id_rsa
```

`chmod 600` çok önemli bir detay. SSH, key dosyasının izinleri çok açıksa (644 gibi) bağlantıyı reddediyor. Bu hatayı ilk denemede yaşadım ve öğrendim.

SSH ile giriş yaptım ve user flag'i aldım:

```
ssh -i id_rsa kenobi@HEDEF_IP
```

## Yetki Yükseltme — PATH Manipulation

Sisteme girdikten sonra her zaman yaptığım kontrollere başladım. `sudo -l` bir şey vermedi. SUID taramasına geçtim:

```
find / -perm -u=s -type f 2>/dev/null
```

Listede `/usr/bin/menu` adında alışılmadık bir dosya gördüm. Çalıştırdığımda 3 seçenekli bir menü çıktı: durum kontrolü, kernel versiyonu, ifconfig. Bu binary arka planda sistem komutları çalıştırıyor olmalı. `strings` komutuyla binary'nin içine baktığımda `curl` komutunu tam yol belirtmeden çağırdığını gördüm.

İşte burada PATH manipulation devreye giriyor. Binary `/usr/bin/curl` yerine sadece `curl` yazıyorsa, PATH'te ilk bulduğu `curl`'ü çalıştırır. Ben de `/tmp` dizininde sahte bir `curl` oluşturup PATH'e ekledim:

```
cd /tmp
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
/usr/bin/menu
```

Menüden herhangi bir seçenek seçtiğimde binary benim sahte curl'ümü çalıştırdı ve root shell geldi.

## Kapanış ve Geliştirici Gözünden

Bu makinede 4 farklı servisin yanlış yapılandırmasını zincirleme olarak kullandım: SMB anonim erişim → ProFTPD mod_copy → NFS herkese açık → SUID binary'de tam yol kullanmama.

Bir yazılımcı olarak bu açıkların nasıl önlenebileceğine bakmak istiyorum:

- **SMB Anonim Erişim:** Anonim paylaşımlar kapatılmalı veya en azından hassas dosyalar (log'lar, yapılandırma bilgileri) bu paylaşımlarda tutulmamalı.
- **ProFTPD mod_copy:** Bu modül kimlik doğrulaması gerektirmiyor. Ya modül devre dışı bırakılmalı ya da güncel bir versiyon kullanılmalı. Servisler her zaman güncel tutulmalı.
- **NFS Yapılandırması:** `/var` gibi geniş bir dizini `*` (herkese) açmak son derece tehlikeli. Belirli subnet'e ve belirli dizine kısıtlanmalı.
- **PATH Manipulation:** SUID binary'lerde komutlar mutlaka tam yol ile çağrılmalı (`/usr/bin/curl` gibi). Aksi halde saldırgan kendi sahte binary'sini PATH'e ekleyerek root komutu çalıştırabilir.
