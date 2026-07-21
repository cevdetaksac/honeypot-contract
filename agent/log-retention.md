# Agent log retention

Client floor: **4.7.6**

## Policy

- Yerel kalıcı log retention süresi **7 takvim günü**dür: bugün + önceki 6 gün.
- Günlük loglar yerel tarih ile adlandırılır:
  - `client-YYYY-MM-DD.log`
  - `threats-YYYY-MM-DD.log`
  - `lifecycle-YYYY-MM-DD.log`
- Gün değişiminde yeni dosyaya doğrudan append edilir. Gece yarısında aktif
  dosyayı rename eden klasik rotation kullanılmaz; SYSTEM daemon ve Guardian'ın
  aynı log ailesine yazdığı durumda cross-process rename yarışı önlenir.
- Tarihi retention penceresinin dışında kalan dosyalar startup/gün değişiminde
  best-effort silinir. Log cleanup hatası agent çalışmasını durdurmaz.
- Eski `client.log`, `client.log.1` vb. dosyalar da dosya yaşı 7 günlük pencereyi
  geçtiğinde migration cleanup kapsamında silinir.

## Log roots

- SYSTEM / Session 0: `%ProgramData%\YesNext\CloudHoneypotClient`
- Interactive GUI: `%APPDATA%\YesNext\CloudHoneypotClient`

GUI “Logları Aç” aksiyonu kendi root'undaki güncel tarihli client logunu açar.

## Update log compatibility

Updater liveness kontrolleri sabit `%ProgramData%\YesNext\CloudHoneypotClient\
update-install.log` adını izlediği için aktif ad değişmez. Update helper başında:

1. Aktif dosyadaki eski tarihli satırlar timestamp'e göre
   `update-install-YYYY-MM-DD.log` dosyalarına ayrılır.
2. Bugünün satırları aktif `update-install.log` içinde kalır.
3. Yedi günlük pencere dışındaki arşivler silinir.

Installer'ın `%LOCALAPPDATA%\honeypot-installer.log` dosyası her installer
başlangıcında sıfırlandığı için ayrıca retention gerektirmez.

## Acceptance

- [ ] Aynı gün tüm yeni satırlar aynı tarihli dosyaya gider.
- [ ] Yerel gün değişince process restart gerekmeden yeni tarihli dosya açılır.
- [ ] Bugün + önceki 6 gün korunur; 7 gün öncesi silinir.
- [ ] Daemon ve Guardian midnight rename yarışı yaşamaz.
- [ ] Threat log cwd'ye değil canonical APPDATA/ProgramData root'una yazılır.
- [ ] Updater liveness için aktif `update-install.log` adı korunur.
- [ ] Log cleanup/permission hatası agent motorunu durdurmaz.
