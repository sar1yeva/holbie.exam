

İmtahanın bu hissəsində **awk**, **sed**, **if**, **for**, **while** və çoxsətirli əmrlər qadağan olduğu üçün bütün analizləri yalnız icazə verilən əsas alətlərlə (grep, cut, sort, uniq, wc və s.) və **tək sətirlik (one-liner) boru xətləri (|)** ilə həll etməlisən.
Bu məhdudiyyətlərə uyğun hazırlanmış, imtahanda birbaşa köçürüb istifadə edə biləcəyin **Loq Analizi Qızıl Qeydləri**:
## 1. Web Server Loqlarının Analizi (Apache / Nginx access.log)
### A. Hücumçu IP-sinin və Alətinin Təsbiti
```bash
# Ən çox sorğu göndərən top 10 IP ünvanını tapmaq
cut -d' ' -f1 access.log | sort | uniq -c | sort -nr | head -n 10

# Hücumçunun skaner alətini (User-Agent) tapmaq (Adətən loqun son sütunlarında olur)
cut -d'"' -f6 access.log | sort | uniq -c | sort -nr | head -n 10

# Müəyyən bir şübhəli IP-nin hansı səhifələrə müraciət etdiyini görmək
grep "HÜCUMÇU_IP" access.log | cut -d'"' -f2 | sort | uniq -c

```
-----------

N3t@dmin241

CRITICAL ERR: TRUNCATED_SECTOR_FLAG(shadow_AUTH_SUBSYSTEM_FAIL_0x4421}


---------
### B. Veb Boşluqlarının Hücum İzlərinin Tapılması
Yuxarıdakı veb tapşırıqlarında istifadə etdiyimiz boşluqların (IDOR, SSRF, Command Injection) loqlardakı izlərini tapmaq:
```bash
# 1. Command Injection cəhdlərini tapmaq (whoami, id, cat, /bin, cmd, net user)
grep -iE "(whoami|id|cat|echo|bin/sh|bin/bash|cmd.exe|net%20user)" access.log

# 2. SSRF cəhdlərini tapmaq (localhost, 127.0.0.1, 169.254)
grep -iE "(127.0.0.1|localhost|169.254.169.254)" access.log

# 3. IDOR / Directory Traversal (.. / ..) cəhdlərini tapmaq
grep -F "../" access.log

```
### C. Status Kodlarına Görə Süzgəcləmə (Uğurlu vs Uğursuz)
```bash
# Hücumçunun uğurlu (200 OK) olan Command Injection və ya SSRF sorğularını tapmaq
grep "HÜCUMÇU_IP" access.log | grep " 200 "

# Dirbuster/Gobuster (Directory Skan) tərəfindən tapıla bilməyən (404 Not Found) səhifələri saymaq
grep " 404 " access.log | cut -d'"' -f2 | sort | uniq -c | sort -nr | head -n 20

```
## 2. Linux Sistem və Giriş Loqları (/var/log/auth.log və ya secure)
Sistemə parolların sınanması (Brute-Force) və ya uğurlu kənar girişlərin analizi:
```bash
# SSH vasitəsilə uğursuz giriş cəhdlərini (Brute-force) və hədəf istifadəçiləri tapmaq
grep "Failed password" auth.log | cut -d' ' -f11 | sort | uniq -c | sort -nr

# SSH ilə sistemə daxil ola bilmiş (Uğurlu) İP və istifadəçiləri tapmaq
grep "Accepted password" auth.log

```
## 3. Zaman Oxu və İncləmə (Timeline Analysis)
İnsidentin dəqiq baş vermə vaxtını süzgəcləmək:
```bash
# Hücumun baş verdiyi dəqiq saat aralığını (məsələn saat 14:30 - 14:35 arası) kəsib çıxarmaq
grep "24/Jun/2026:14:3[0-5]" access.log > insident_vaxti.log

# Müəyyən bir faylın (məsələn webshell.php) loqlarda ilk görünmə vaxtını tapmaq
grep "webshell.php" access.log | head -n 1

```
## 4. Böyük Həcmli Faylların və Qovluqların Tapılması (find, xargs, du)
Hücumçunun sistemə yüklədiyi böyük həcmli zərərli proqramları və ya oğurlamaq üçün arxivlədiyi (zip/tar) dataları tapmaq:
```bash
# Son 2 gündə dəyişdirilmiş və ya yaradılmış .php fayllarını tapmaq
find /var/www/html/ -type f -name "*.php" -mmin -2880

# Sistemdəki 50MB-dan böyük olan bütün faylları tapmaq
find / -type f -size +50M 2>/dev/null

# Find ilə tapılan şübhəli loq fayllarının sətir sayını sürətlə hesablamaq (xargs ilə)
find /var/log/ -name "*.log" | xargs wc -l

```
### 💡 İmtahan üçün Sürətli Yaddaş Kartı (Cheat-Sheet):
 * cut -d' ' -f1 -> Sətiri boşluğa (' ') görə bölür və birinci hissəni (adətən IP ünvanını) götürür.
 * sort | uniq -c | sort -nr -> Bu üçlük ayrılmazdır. Məlumatları qruplaşdırır, təkrar sayını göstərir və ən çox təkrarlanandan ən az olana doğru sıralayır.
 * grep -i -> Böyük/kiçik hərf fərqini aradan qaldırır (WhoAmI ilə whoami eyni sayılır).
 * grep -F -> Simvolları regex kimi deyil, olduğu kimi axtarır (məsələn ../ axtarışında nöqtələri xüsusi simvol saymır).
 * 2>/dev/null -> find əmrinin sonuna mütləq yazın ki, icazə xətaları (Permission denied) ekranı doldurmasın.


