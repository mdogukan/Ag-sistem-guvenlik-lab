# Ağ ve Sistem Güvenliği Lab Projesi

Bu proje, staj sürecim kapsamında sanal ortamda kurduğum, gerçekçi ölçekte bir kurumsal ağın güvenliğini uçtan uca deneyimlemek amacıyla hazırlanmıştır. Sıfırdan bir ağ altyapısı kurup, üzerine kimlik yönetimi, loglama, ve hem uç nokta hem ağ seviyesinde saldırı tespiti ekledim; ayrıca gerçek saldırı senaryolarını (brute-force, SQL injection) deneyerek bu savunma katmanlarının nasıl çalıştığını kanıtladım.

## Kullanılan Araçlar

- **VMware Workstation Pro** — sanallaştırma platformu
- **pfSense** — firewall / router
- **Windows Server 2022** — Active Directory Domain Controller
- **Windows 11** — domain istemcisi
- **Kali Linux** — saldırgan makine
- **OWASP Broken Web Applications (DVWA)** — hedef zafiyetli web uygulaması
- **Sysmon** — uç nokta (endpoint) loglama
- **Suricata** — ağ seviyesinde saldırı tespit sistemi (IDS)

## Ağ Mimarisi

Tüm sanal makineler, pfSense'in LAN arayüzüne bağlı `10.10.10.0/24` adresli izole bir sanal ağda (VMware host-only network) çalışıyor. pfSense, bu iç ağ ile dış dünya arasındaki tek geçiş noktası; NAT ile internete çıkışı sağlıyor, DHCP ile iç ağdaki makinelere IP dağıtıyor.

```
İnternet
   │
 [pfSense: WAN/LAN]
   │
 Sanal LAN (10.10.10.0/24)
   │
   ├── DC01 (Domain Controller) — 10.10.10.10
   ├── Windows 11 (domain istemcisi)
   ├── Ubuntu
   ├── Kali Linux (saldırgan)
   └── OWASP BWA / DVWA — 10.10.10.104
```

## 1. Ağ Altyapısı ve Firewall

pfSense üzerinde WAN/LAN ayrımı yapılandırıldı, NAT ile internet çıkışı ve DHCP ile iç ağ IP dağıtımı sağlandı.

![pfSense Dashboard](screenshots/01-pfsense-dashboard.png)

## 2. Kimlik ve Erişim Yönetimi (Active Directory)

DC01 üzerinde bir Organizational Unit (OU) ve bu OU altında kullanıcılar oluşturuldu. Bu OU'ya bağlı bir Group Policy Object (GPO) ile minimum 10 karakter ve karmaşıklık zorunluluğu içeren bir parola politikası tanımlandı.

![AD OU ve Kullanıcılar](screenshots/03-ad-ou-kullanicilar.png)
![GPO Bağlantısı](screenshots/04-gpo-baglanti.png)

## 3. Güvenlik Politikasının Doğrulanması

Tanımlanan parola politikasının gerçekten çalıştığını doğrulamak için domaine katılmış bir Windows 11 istemcisinde zayıf bir şifre denendi ve sistem tarafından reddedildi.

![GPO Parola Reddi](screenshots/07-gpo-sifre-reddi.png)

## 4. Saldırı Simülasyonu ve Uç Nokta Tespiti

Kali Linux üzerinden DC01'e karşı `netexec` ile bir SMB brute-force (şifre tahmin) saldırısı gerçekleştirildi.

![Brute-force Başarılı](screenshots/08-bruteforce-basarili.png)

Bu saldırı, DC01 üzerine kurulan Sysmon ve Windows'un yerleşik güvenlik loglaması (Event Viewer) sayesinde tespit edildi — başarısız giriş denemeleri, kaynak IP adresiyle birlikte kayıt altına alındı.

![Event Viewer - Başarısız Girişler](screenshots/05-event-4625-liste.png)
![Event Viewer - Kaynak IP Detayı](screenshots/06-event-4625-detay.png)

## 5. Web Uygulama Güvenlik Testi (SQL Injection)

OWASP Broken Web Applications üzerindeki DVWA hedef alınarak, kullanıcı girdisinin veritabanı sorgusuyla doğrudan birleştirilmesinden kaynaklanan bir SQL Injection açığı test edildi. `1' OR '1'='1` girdisiyle, tek bir kullanıcı yerine veritabanındaki tüm kullanıcı kayıtları elde edildi.

![SQL Injection Sonucu](screenshots/09-sql-injection-sonuc.png)

## 6. Ağ Seviyesinde Saldırı Tespiti (Suricata IDS)

pfSense üzerine kurulan Suricata IDS, SQL Injection saldırı trafiğini ağ seviyesinde (hedefe ulaşmadan) tespit edip uyarı üretti.

![Suricata Alerts](screenshots/02-suricata-alerts.png)

## Öğrenilenler

Bu proje sürecinde ağ topolojisi kurulumu, Active Directory ile kimlik/erişim yönetimi, Windows ve Linux platformlarında loglama, uç nokta ve ağ seviyesinde saldırı tespiti, ve temel web uygulama güvenlik açıkları konularında uygulamalı deneyim kazandım. Ayrıca gerçek bir laboratuvar ortamında karşılaşılan DNS, Kerberos ve ağ görünürlüğü (promiscuous mode) gibi teknik sorunları teşhis edip çözme sürecinden de önemli bir troubleshooting deneyimi edindim.
