# OBD-II / ELM327 Tam PID ve Komut Referansı

**Kaynak:** Car Scanner ELM OBD2 (com.ovz.carscanner) reverse engineering  
**Standart:** SAE J1979 / ISO 15031-5  
**Tarih:** 23 Mart 2026  
**Amaç:** OBD-II uygulaması geliştirmek isteyenler için eksiksiz referans  

---

## İÇİNDEKİLER

1. [ELM327 Adaptör İletişimi](#1-elm327-adaptör-i̇letişimi)
2. [Bağlantı Kurulumu](#2-bağlantı-kurulumu)
3. [AT Komutları Tam Listesi](#3-at-komutları-tam-listesi)
4. [OBD Protokolleri](#4-obd-protokolleri)
5. [Mode 01 — Canlı Veri (150 PID)](#5-mode-01--canlı-veri-150-pid)
6. [Mode 02 — Freeze Frame](#6-mode-02--freeze-frame)
7. [Mode 03 — DTC Okuma](#7-mode-03--dtc-arıza-kodu-okuma)
8. [Mode 04 — DTC Silme](#8-mode-04--dtc-silme)
9. [Mode 05 — O2 Sensör Monitör](#9-mode-05--o2-sensör-monitör-testi)
10. [Mode 06 — On-Board Monitoring](#10-mode-06--on-board-monitoring)
11. [Mode 09 — Araç Bilgileri](#11-mode-09--araç-bilgileri)
12. [Mode 22 — Extended / UDS](#12-mode-22--extended-diagnostics-uds)
13. [DTC Kod Yapısı](#13-dtc-arıza-kodu-yapısı)
14. [Üretici Özel Protokoller](#14-üretici-özel-protokoller)
15. [Bağlantı Profilleri](#15-bağlantı-profilleri)
16. [Veri Parse Formülleri](#16-veri-parse-formülleri)
17. [Örnek Kod Yapıları](#17-örnek-kod-yapıları)

---

## 1. ELM327 Adaptör İletişimi

### İletişim Temelleri

- **Satır sonu:** Her komut `\r` (CR, 0x0D) ile biter
- **Yanıt sonu:** `>` prompt karakteri yanıtın bittiğini gösterir
- **Çoklu ECU:** Birden fazla ECU yanıt verebilir — tüm yanıtları parse et
- **Hata:** `NO DATA`, `UNABLE TO CONNECT`, `?`, `ERROR`
- **Timeout:** Varsayılan ~200ms, `ATST` ile ayarlanır

### Bağlantı Adresleri

| Yöntem | Adres | Port | Açıklama |
|--------|-------|------|----------|
| **WiFi (Varsayılan)** | `192.168.0.10` | `35000` | Çoğu WiFi ELM327 klonu |
| **WiFi (PLX KiWi)** | `192.168.0.10` | `30000` | PLX KiWi WiFi |
| **WiFi (OBDLink)** | `192.168.0.74` | `23` | OBDLink WiFi |
| **WiFi (Alt. Subnet)** | `192.168.1.10` | `35000` | Alternatif ağ |
| **Bluetooth SPP** | UUID: `00001101-0000-1000-8000-00805F9B34FB` | — | Serial Port Profile (RFCOMM) |
| **Bluetooth LE** | GATT servisleri | — | BLE 4.0+ adaptörler |

WiFi ağ adı tespiti: `WiFi_OBDII` veya `Wi-Fi_OBDII` SSID'yi ara.

### TCP Socket Bağlantısı (WiFi)

```
1. WiFi ağına bağlan (SSID: WiFi_OBDII)
2. TCP socket aç → 192.168.0.10:35000
3. ATZ\r gönder → "ELM327 v1.5" benzeri yanıt bekle
4. Init sekansını çalıştır
```

### Bluetooth SPP Bağlantısı

```
1. Bluetooth cihaz taraması yap
2. "OBD" veya "ELM" içeren cihaz bul
3. SPP UUID ile RFCOMM bağlantısı kur:
   UUID: 00001101-0000-1000-8000-00805F9B34FB
4. ATZ\r gönder → yanıt bekle
5. Init sekansını çalıştır
```

---

## 2. Bağlantı Kurulumu

### Init Sekansı (Başlatma)

Aşağıdaki komutlar sırayla gönderilir. Her komuttan sonra `>` prompt'u bekle.

```
ATZ\r          → Adaptörü resetle          → Yanıt: "ELM327 v1.5" (sürüm bilgisi)
ATE0\r         → Echo kapat                → Yanıt: "OK"
ATS0\r         → Boşlukları kaldır         → Yanıt: "OK"
ATL0\r         → Linefeed kapat            → Yanıt: "OK"
ATH0\r         → Header gizle              → Yanıt: "OK"
ATAL\r         → Uzun mesajlara izin ver   → Yanıt: "OK"
ATSP0\r        → Protokol otomatik algıla  → Yanıt: "OK"
```

### Voltaj Okuma

```
ATRV\r         → Yanıt: "12.6V"   (akü voltajı)
```

### Protokol Seçimi (Manuel)

```
ATSP1\r   → SAE J1850 PWM (41.6 kbps)
ATSP2\r   → SAE J1850 VPW (10.4 kbps)
ATSP3\r   → ISO 9141-2 (5 baud init)
ATSP4\r   → ISO 14230 KWP (5 baud init)
ATSP5\r   → ISO 14230 KWP (fast init)
ATSP6\r   → ISO 15765 CAN 11-bit 500 kbps
ATSP7\r   → ISO 15765 CAN 29-bit 500 kbps
ATSP8\r   → ISO 15765 CAN 11-bit 250 kbps
ATSP9\r   → ISO 15765 CAN 29-bit 250 kbps
ATSPA\r   → SAE J1939 CAN 29-bit 250 kbps
ATSP0\r   → Otomatik algıla
```

---

## 3. AT Komutları Tam Listesi

### Genel Komutlar

| Komut | Açıklama | Yanıt |
|-------|----------|-------|
| `ATZ` | Adaptörü tamamen resetle | `ELM327 v1.5` |
| `ATD` | Tüm ayarları varsayılana döndür | `OK` |
| `ATE0` | Echo kapat (gönderilen komutu yankılama) | `OK` |
| `ATE1` | Echo aç | `OK` |
| `ATI` | ELM327 versiyon bilgisi | `ELM327 v1.5` |
| `AT@1` | Cihaz açıklama dizesi | Değişken |
| `AT@2` | Cihaz tanımlayıcı | Değişken |
| `ATRV` | Araç akü voltajını oku | `12.6V` |
| `ATWS` | Sıcak başlatma (warm start) | — |
| `ATLP` | Düşük güç moduna gir | — |

### Protokol Komutları

| Komut | Açıklama |
|-------|----------|
| `ATSP0-9,A` | Protokol seç (0=auto, 1-9=manuel, A=J1939) |
| `ATDP` | Aktif protokolü göster |
| `ATDPN` | Aktif protokol numarasını göster |
| `ATMA` | Tüm CAN mesajlarını monitör et |
| `ATCM` | CAN ID maskeleme ayarla |
| `ATCF` | CAN ID filtre ayarla |
| `ATAR` | Otomatik alımı ayarla |
| `ATAT0-2` | Adaptive Timing (0=off, 1=normal, 2=agresif) |

### Format Komutları

| Komut | Açıklama |
|-------|----------|
| `ATS0` | Yanıtlardaki boşlukları kaldır |
| `ATS1` | Yanıtlardaki boşlukları göster |
| `ATH0` | CAN header'ları gizle |
| `ATH1` | CAN header'ları göster |
| `ATL0` | Linefeed gizle |
| `ATL1` | Linefeed göster |
| `ATAL` | Uzun mesajlara izin ver (>7 byte) |

### CAN Bus Komutları

| Komut | Açıklama |
|-------|----------|
| `ATSH xxx` | CAN header/adres ayarla (ör: `ATSH 7E0`) |
| `ATST xx` | Timeout ayarla (hex, ×4ms) |
| `ATFC` | Flow Control ayarla |
| `ATCAF0` | CAN Auto Formatting kapat |
| `ATCAF1` | CAN Auto Formatting aç |
| `ATCEA` | CAN Extended Address |
| `ATCER` | CAN Error kontrolü (ELM327 v2.2+) |
| `ATTA xx` | Transmit Address ayarla (ELM327 v2.2+) |
| `ATCSM0` | CAN Silent Mode kapat |
| `ATCSM1` | CAN Silent Mode aç |

### J1939 Komutları

| Komut | Açıklama |
|-------|----------|
| `ATDM1` | DM1 (aktif arıza) monitör et |
| `ATMP xxxx` | J1939 PGN monitör et |
| `ATJE` | J1939 elm formatı |
| `ATJS` | J1939 SAE formatı |
| `ATJHF0/1` | J1939 Header Formatting |

---

## 4. OBD Protokolleri

### Otomatik Algılama Sırası

| # | Protokol | AT Kodu | Baud | Yıl | Araç Tipi |
|---|----------|---------|------|-----|-----------|
| 6 | ISO 15765-4 CAN 11-bit 500k | `ATSP6` | 500 kbps | 2008+ | **En yaygın (önce dene)** |
| 8 | ISO 15765-4 CAN 11-bit 250k | `ATSP8` | 250 kbps | 2004+ | Bazı AB araçları |
| 7 | ISO 15765-4 CAN 29-bit 500k | `ATSP7` | 500 kbps | 2004+ | Bazı araçlar |
| 9 | ISO 15765-4 CAN 29-bit 250k | `ATSP9` | 250 kbps | 2004+ | Nadir |
| 5 | ISO 14230-4 KWP (fast init) | `ATSP5` | 10.4 kbps | 1999-2008 | Eski Avrupa/Asya |
| 4 | ISO 14230-4 KWP (5 baud) | `ATSP4` | 10.4 kbps | 1999-2008 | Eski araçlar |
| 3 | ISO 9141-2 | `ATSP3` | 10.4 kbps | 1996-2004 | Eski araçlar |
| 1 | SAE J1850 PWM | `ATSP1` | 41.6 kbps | 1996-2008 | Eski Ford |
| 2 | SAE J1850 VPW | `ATSP2` | 10.4 kbps | 1996-2008 | Eski GM |
| A | SAE J1939 | `ATSPA` | 250 kbps | Hepsi | Ticari araçlar |
| — | ISO 27145-2 WWH-OBD | CAN üzeri | CAN | 2025+ | Yeni nesil |

### CAN Adresleme

| Adres | Gönderi → Alım | Açıklama |
|-------|----------------|----------|
| `7E0` → `7E8` | ECU #1 (Motor) | Ana motor kontrol ünitesi |
| `7E1` → `7E9` | ECU #2 (Şanzıman) | Otomatik şanzıman |
| `7E2` → `7EA` | ECU #3 | Ek kontrol ünitesi |
| `7E3` → `7EB` | ECU #4 | Ek kontrol ünitesi |
| `7E4` → `7EC` | ECU #5 | Ek kontrol ünitesi |
| `7E5` → `7ED` | ECU #6 | Ek kontrol ünitesi |
| `7E6` → `7EE` | ECU #7 | Ek kontrol ünitesi |
| `7E7` → `7EF` | ECU #8 | Ek kontrol ünitesi |
| `7DF` | — | Broadcast (tüm ECU'lar) |

---

## 5. Mode 01 — Canlı Veri (150 PID)

### Gönderme Formatı
```
01XX\r
```
XX = PID numarası (hex)

### Yanıt Formatı
```
41 XX AA BB CC DD
```
- `41` = Mode 01 yanıtı (01 + 0x40)
- `XX` = PID numarası
- `AA BB CC DD` = Veri byte'ları (PID'ye göre sayı değişir)

### Desteklenen PID Sorgulama

PID `00`, `20`, `40`, `60`, `80`, `A0`, `C0`, `E0` → 32-bit bitmap döndürür, hangi PID'lerin desteklendiğini gösterir.

**Bitmap okuma:** Her bit bir sonraki 32 PID'i temsil eder. MSB = ilk PID.

### Tam PID Tablosu

| PID (Hex) | Komut | Sensör Adı (EN) | Sensör Adı (TR) | Byte | Min | Max | Birim | Formül |
|-----------|-------|-----------------|-----------------|------|-----|-----|-------|--------|
| `00` | `0100` | PIDs supported [01-20] | Desteklenen PID'ler [01-20] | 4 | — | — | Bitmap | Bit maskeleme |
| `01` | `0101` | Monitor status since DTCs cleared (MIL status, DTC count) | DTC silindikten sonra monitör durumu (MIL, DTC sayısı) | 4 | — | — | — | A: bit 7=MIL on/off, bit 0-6=DTC sayısı |
| `02` | `0102` | Freeze frame DTC | Freeze frame arıza kodu | 2 | — | — | DTC | Standart DTC parse |
| `03` | `0103` | Fuel system status | Yakıt sistemi durumu | 2 | — | — | Enum | 1=OL, 2=CL, 4=OL-drive, 8=OL-fault, 16=CL-fault |
| `04` | `0104` | Calculated engine load | Hesaplanmış motor yükü | 1 | 0 | 100 | % | `A × 100 / 255` |
| `05` | `0105` | Engine coolant temperature | Motor soğutma suyu sıcaklığı | 1 | −40 | 215 | °C | `A − 40` |
| `06` | `0106` | Short term fuel trim — Bank 1 | Kısa vadeli yakıt ayarı — Bank 1 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `07` | `0107` | Long term fuel trim — Bank 1 | Uzun vadeli yakıt ayarı — Bank 1 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `08` | `0108` | Short term fuel trim — Bank 2 | Kısa vadeli yakıt ayarı — Bank 2 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `09` | `0109` | Long term fuel trim — Bank 2 | Uzun vadeli yakıt ayarı — Bank 2 | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `0A` | `010A` | Fuel pressure (gauge) | Yakıt basıncı (gösterge) | 1 | 0 | 765 | kPa | `A × 3` |
| `0B` | `010B` | Intake manifold absolute pressure | Emme manifold mutlak basıncı (MAP) | 1 | 0 | 255 | kPa | `A` |
| `0C` | `010C` | Engine RPM | Motor devri (RPM) | 2 | 0 | 16383.75 | rpm | `((A × 256) + B) / 4` |
| `0D` | `010D` | Vehicle speed | Araç hızı | 1 | 0 | 255 | km/h | `A` |
| `0E` | `010E` | Timing advance | Ateşleme zamanlama avansı | 1 | −64 | 63.5 | ° (BTDC) | `(A / 2) − 64` |
| `0F` | `010F` | Intake air temperature | Emme havası sıcaklığı | 1 | −40 | 215 | °C | `A − 40` |
| `10` | `0110` | Mass air flow sensor (MAF) rate | Kütle hava akış sensörü debisi | 2 | 0 | 655.35 | g/s | `((A × 256) + B) / 100` |
| `11` | `0111` | Throttle position | Gaz kelebeği pozisyonu | 1 | 0 | 100 | % | `A × 100 / 255` |
| `12` | `0112` | Commanded secondary air status | Komutlu ikincil hava durumu | 1 | — | — | Enum | Bit maskeleme |
| `13` | `0113` | Oxygen sensors present (2 banks) | Mevcut oksijen sensörleri (2 bank) | 1 | — | — | Bitmap | — |
| `14` | `0114` | O2 Sensor 1 — Voltage, STFT | O2 Sensör 1 — Voltaj, Kısa trim | 2 | 0-0 | 1.275-99.2 | V, % | `A/200`, `(B−128)×100/128` |
| `15` | `0115` | O2 Sensor 2 — Voltage, STFT | O2 Sensör 2 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `16` | `0116` | O2 Sensor 3 — Voltage, STFT | O2 Sensör 3 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `17` | `0117` | O2 Sensor 4 — Voltage, STFT | O2 Sensör 4 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `18` | `0118` | O2 Sensor 5 — Voltage, STFT | O2 Sensör 5 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `19` | `0119` | O2 Sensor 6 — Voltage, STFT | O2 Sensör 6 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `1A` | `011A` | O2 Sensor 7 — Voltage, STFT | O2 Sensör 7 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `1B` | `011B` | O2 Sensor 8 — Voltage, STFT | O2 Sensör 8 — Voltaj, Kısa trim | 2 | — | — | V, % | Aynı formül |
| `1C` | `011C` | OBD standards compliance | OBD standart uyumu | 1 | — | — | Enum | 1=OBD-II, 2=OBD, 3=OBD+OBD-II... |
| `1D` | `011D` | Oxygen sensors present (4 banks) | Mevcut O2 sensörleri (4 bank) | 1 | — | — | Bitmap | — |
| `1E` | `011E` | Auxiliary input status | Yardımcı giriş durumu | 1 | — | — | Bit | Bit 0=PTO aktif |
| `1F` | `011F` | Run time since engine start | Motor çalışma süresi | 2 | 0 | 65535 | saniye | `(A × 256) + B` |
| `20` | `0120` | PIDs supported [21-40] | Desteklenen PID'ler [21-40] | 4 | — | — | Bitmap | — |
| `21` | `0121` | Distance traveled with MIL on | MIL açıkken kat edilen mesafe | 2 | 0 | 65535 | km | `(A × 256) + B` |
| `22` | `0122` | Fuel rail pressure (relative to manifold vacuum) | Yakıt ray basıncı (manifold vakuma göre) | 2 | 0 | 5177.27 | kPa | `((A × 256) + B) × 0.079` |
| `23` | `0123` | Fuel rail gauge pressure (diesel/GDI) | Yakıt ray basıncı (dizel/GDI) | 2 | 0 | 655350 | kPa | `((A × 256) + B) × 10` |
| `24` | `0124` | O2 Sensor 1 — λ ratio, voltage | O2 Sensör 1 — Lambda, voltaj | 4 | 0 | 2/8 | ratio/V | `λ=((A×256)+B)×2/65536`, `V=((C×256)+D)×8/65536` |
| `25` | `0125` | O2 Sensor 2 — λ ratio, voltage | O2 Sensör 2 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `26` | `0126` | O2 Sensor 3 — λ ratio, voltage | O2 Sensör 3 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `27` | `0127` | O2 Sensor 4 — λ ratio, voltage | O2 Sensör 4 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `28` | `0128` | O2 Sensor 5 — λ ratio, voltage | O2 Sensör 5 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `29` | `0129` | O2 Sensor 6 — λ ratio, voltage | O2 Sensör 6 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `2A` | `012A` | O2 Sensor 7 — λ ratio, voltage | O2 Sensör 7 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `2B` | `012B` | O2 Sensor 8 — λ ratio, voltage | O2 Sensör 8 — Lambda, voltaj | 4 | — | — | ratio/V | Aynı formül |
| `2C` | `012C` | Commanded EGR | Komutlu EGR değeri | 1 | 0 | 100 | % | `A × 100 / 255` |
| `2D` | `012D` | EGR error | EGR hatası | 1 | −100 | 99.2 | % | `(A − 128) × 100 / 128` |
| `2E` | `012E` | Commanded evaporative purge | Komutlu buharlaşma temizleme | 1 | 0 | 100 | % | `A × 100 / 255` |
| `2F` | `012F` | Fuel tank level input | Yakıt seviyesi girişi | 1 | 0 | 100 | % | `A × 100 / 255` |
| `30` | `0130` | Warm-ups since DTCs cleared | DTC silindikten sonra ısınma sayısı | 1 | 0 | 255 | sayı | `A` |
| `31` | `0131` | Distance traveled since DTCs cleared | DTC silindikten sonra mesafe | 2 | 0 | 65535 | km | `(A × 256) + B` |
| `32` | `0132` | Evap system vapor pressure | Buharlaşma sistemi basıncı | 2 | −8192 | 8191.75 | Pa | `((A×256)+B) / 4` (işaretli) |
| `33` | `0133` | Absolute barometric pressure | Mutlak barometrik basınç | 1 | 0 | 255 | kPa | `A` |
| `34` | `0134` | O2 Sensor 1 — λ ratio, current | O2 Sensör 1 — Lambda, akım | 4 | 0 | 2/128 | ratio/mA | `λ=((A×256)+B)/32768`, `i=((C×256)+D)/256−128` |
| `35` | `0135` | O2 Sensor 2 — λ ratio, current | O2 Sensör 2 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `36` | `0136` | O2 Sensor 3 — λ ratio, current | O2 Sensör 3 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `37` | `0137` | O2 Sensor 4 — λ ratio, current | O2 Sensör 4 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `38` | `0138` | O2 Sensor 5 — λ ratio, current | O2 Sensör 5 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `39` | `0139` | O2 Sensor 6 — λ ratio, current | O2 Sensör 6 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `3A` | `013A` | O2 Sensor 7 — λ ratio, current | O2 Sensör 7 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `3B` | `013B` | O2 Sensor 8 — λ ratio, current | O2 Sensör 8 — Lambda, akım | 4 | — | — | ratio/mA | Aynı formül |
| `3C` | `013C` | Catalyst temperature B1S1 | Katalizör sıcaklığı B1S1 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `3D` | `013D` | Catalyst temperature B2S1 | Katalizör sıcaklığı B2S1 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `3E` | `013E` | Catalyst temperature B1S2 | Katalizör sıcaklığı B1S2 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `3F` | `013F` | Catalyst temperature B2S2 | Katalizör sıcaklığı B2S2 | 2 | −40 | 6513.5 | °C | `((A×256)+B)/10 − 40` |
| `40` | `0140` | PIDs supported [41-60] | Desteklenen PID'ler [41-60] | 4 | — | — | Bitmap | — |
| `41` | `0141` | Monitor status this drive cycle | Bu sürüş döngüsünde monitör durumu | 4 | — | — | — | PID 01 ile aynı yapı |
| `42` | `0142` | Control module voltage | Kontrol modülü voltajı | 2 | 0 | 65.535 | V | `((A × 256) + B) / 1000` |
| `43` | `0143` | Absolute load value | Mutlak yük değeri | 2 | 0 | 25700 | % | `((A × 256) + B) × 100 / 255` |
| `44` | `0144` | Commanded air-fuel equiv. ratio | Komutlu hava-yakıt eşdeğer oranı | 2 | 0 | 2 | λ | `((A × 256) + B) / 32768` |
| `45` | `0145` | Relative throttle position | Rölatif gaz kelebeği pozisyonu | 1 | 0 | 100 | % | `A × 100 / 255` |
| `46` | `0146` | Ambient air temperature | Ortam havası sıcaklığı | 1 | −40 | 215 | °C | `A − 40` |
| `47` | `0147` | Absolute throttle position B | Mutlak gaz kelebeği B | 1 | 0 | 100 | % | `A × 100 / 255` |
| `48` | `0148` | Absolute throttle position C | Mutlak gaz kelebeği C | 1 | 0 | 100 | % | `A × 100 / 255` |
| `49` | `0149` | Accelerator pedal position D | Gaz pedalı pozisyonu D | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4A` | `014A` | Accelerator pedal position E | Gaz pedalı pozisyonu E | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4B` | `014B` | Accelerator pedal position F | Gaz pedalı pozisyonu F | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4C` | `014C` | Commanded throttle actuator | Komutlu gaz kelebeği aktüatör | 1 | 0 | 100 | % | `A × 100 / 255` |
| `4D` | `014D` | Time run with MIL on | MIL açıkken çalışma süresi | 2 | 0 | 65535 | dakika | `(A × 256) + B` |
| `4E` | `014E` | Time since DTCs cleared | DTC silindikten sonra süre | 2 | 0 | 65535 | dakika | `(A × 256) + B` |
| `4F` | `014F` | Max values — λ, O2V, O2I, intake pressure | Maks değerler | 4 | — | — | çeşitli | `A=ratio, B=V, C=mA, D=kPa×10` |
| `50` | `0150` | Max value — air flow rate from MAF | Maks MAF hava akış | 4 | 0 | 2550 | g/s | `A × 10` |
| `51` | `0151` | Fuel type | Yakıt tipi | 1 | — | — | Enum | 1=Benzin, 4=Dizel... |
| `52` | `0152` | Ethanol fuel % | Etanol yakıt yüzdesi | 1 | 0 | 100 | % | `A × 100 / 255` |
| `53` | `0153` | Absolute evap system vapor press. | Mutlak buharlaşma basıncı | 2 | 0 | 327.675 | kPa | `((A×256)+B) / 200` |
| `54` | `0154` | Evap system vapor pressure (wide) | Buharlaşma basıncı (geniş) | 2 | −32767 | 32767 | Pa | `((A×256)+B) − 32767` |
| `55` | `0155` | Short term secondary O2 trim B1+B3 | Kısa O2 trim B1+B3 | 2 | −100 | 99.2 | % | Her byte: `(X−128)×100/128` |
| `56` | `0156` | Long term secondary O2 trim B1+B3 | Uzun O2 trim B1+B3 | 2 | — | — | % | Aynı formül |
| `57` | `0157` | Short term secondary O2 trim B2+B4 | Kısa O2 trim B2+B4 | 2 | — | — | % | Aynı formül |
| `58` | `0158` | Long term secondary O2 trim B2+B4 | Uzun O2 trim B2+B4 | 2 | — | — | % | Aynı formül |
| `59` | `0159` | Fuel rail abs. pressure | Yakıt ray mutlak basıncı | 2 | 0 | 655350 | kPa | `((A×256)+B) × 10` |
| `5A` | `015A` | Relative accelerator pedal position | Rölatif gaz pedalı pozisyonu | 1 | 0 | 100 | % | `A × 100 / 255` |
| `5B` | `015B` | Hybrid battery pack remaining life | Hibrit akü kalan ömür | 1 | 0 | 100 | % | `A × 100 / 255` |
| `5C` | `015C` | Engine oil temperature | Motor yağ sıcaklığı | 1 | −40 | 210 | °C | `A − 40` |
| `5D` | `015D` | Fuel injection timing | Yakıt enjeksiyon zamanlaması | 2 | −210 | 301.99 | ° | `(((A×256)+B)−26880) / 128` |
| `5E` | `015E` | Engine fuel rate | Motor yakıt tüketim oranı | 2 | 0 | 3212.75 | L/h | `((A×256)+B) / 20` |
| `5F` | `015F` | Emission requirements | Emisyon gereksinimleri | 1 | — | — | Enum | — |
| `60` | `0160` | PIDs supported [61-80] | Desteklenen PID'ler [61-80] | 4 | — | — | Bitmap | — |
| `61` | `0161` | Driver's demand engine torque | Sürücü talep motor torku | 1 | −125 | 130 | % | `A − 125` |
| `62` | `0162` | Actual engine torque | Gerçek motor torku | 1 | −125 | 130 | % | `A − 125` |
| `63` | `0163` | Engine reference torque | Motor referans torku | 2 | 0 | 65535 | Nm | `(A × 256) + B` |
| `64` | `0164` | Engine percent torque data | Motor tork yüzde verileri | 5 | — | — | % | 5 byte: idle, pt1, pt2, pt3, pt4 |
| `65` | `0165` | Auxiliary input / output supported | Yardımcı I/O desteği | 2 | — | — | Bitmap | — |
| `66` | `0166` | Mass air flow sensor (advanced) | Gelişmiş MAF sensörü | 5 | — | — | g/s | — |
| `67` | `0167` | Engine coolant temperature (dual) | Motor soğutma suyu (çift) | 3 | — | — | °C | — |
| `68` | `0168` | Intake air temperature sensor | Emme havası sıcaklık sensörü | 7 | — | — | °C | — |
| `69` | `0169` | Commanded EGR and EGR error | Komutlu EGR ve EGR hatası | 7 | — | — | %/% | — |
| `6A` | `016A` | Commanded Diesel intake air flow control | Dizel emme hava akış kontrolü | 5 | — | — | %/% | — |
| `6B` | `016B` | Exhaust gas recirculation temperature | Egzoz gaz geri dönüşüm sıcaklığı | 5 | — | — | °C | — |
| `6C` | `016C` | Commanded throttle actuator control | Komutlu gaz kelebeği aktüatör | 5 | — | — | %/% | — |
| `6D` | `016D` | Fuel pressure control system | Yakıt basınç kontrol sistemi | 6 | — | — | kPa | — |
| `6E` | `016E` | Injection pressure control system | Enjeksiyon basınç kontrol | 5 | — | — | kPa | — |
| `6F` | `016F` | Turbocharger compressor inlet pressure | Turbo kompresör giriş basıncı | 3 | — | — | kPa | — |
| `70` | `0170` | Boost pressure control | Boost basınç kontrolü | 9 | — | — | kPa | — |
| `71` | `0171` | Variable geometry turbo control | Değişken geometri turbo kontrolü | 5 | — | — | %/% | — |
| `72` | `0172` | Wastegate control | Wastegate kontrolü | 5 | — | — | %/% | — |
| `73` | `0173` | Exhaust pressure | Egzoz basıncı | 5 | — | — | kPa | — |
| `74` | `0174` | Turbocharger RPM | Turbo devri | 5 | — | — | rpm | — |
| `75` | `0175` | Turbocharger temperature A | Turbo sıcaklık A | 7 | — | — | °C | — |
| `76` | `0176` | Turbocharger temperature B | Turbo sıcaklık B | 7 | — | — | °C | — |
| `77` | `0177` | Charge air cooler temperature | Intercooler sıcaklığı | 5 | — | — | °C | — |
| `78` | `0178` | Exhaust gas temperature Bank 1 | Egzoz gazı sıcaklığı Bank 1 | 9 | — | — | °C | `((A×256)+B)/10 − 40` |
| `79` | `0179` | Exhaust gas temperature Bank 2 | Egzoz gazı sıcaklığı Bank 2 | 9 | — | — | °C | `((A×256)+B)/10 − 40` |
| `7A` | `017A` | Diesel particulate filter (DPF) temp | DPF sıcaklığı | 9 | — | — | °C | — |
| `7B` | `017B` | Diesel particulate filter (DPF) pressure | DPF basıncı | 9 | — | — | kPa | — |
| `7C` | `017C` | Diesel particulate filter (DPF) diff. pressure | DPF diferansiyel basınç | 5 | — | — | kPa | — |
| `7D` | `017D` | NOx NTE control area status | NOx NTE kontrol alanı durumu | 1 | — | — | Bitmap | — |
| `7E` | `017E` | PM NTE control area status | PM NTE kontrol alanı durumu | 1 | — | — | Bitmap | — |
| `7F` | `017F` | Engine run time | Motor çalışma süresi (genişletilmiş) | 13 | — | — | saniye | — |
| `80` | `0180` | PIDs supported [81-A0] | Desteklenen PID'ler [81-A0] | 4 | — | — | Bitmap | — |
| `81` | `0181` | Engine run time — AECD #1-#5 | Motor çalışma — AECD #1-#5 | 21 | — | — | saniye | — |
| `82` | `0182` | Engine run time — AECD #6-#10 | Motor çalışma — AECD #6-#10 | 21 | — | — | saniye | — |
| `83` | `0183` | NOx sensor | NOx sensörü | 5 | — | — | ppm | — |
| `84` | `0184` | Manifold surface temperature | Manifold yüzey sıcaklığı | 1 | — | — | °C | — |
| `85` | `0185` | NOx reagent system | NOx reaktif sistemi | 10 | — | — | — | — |
| `86` | `0186` | Particulate matter (PM) sensor | Partikül madde sensörü | 5 | — | — | mg/m³ | — |
| `87` | `0187` | Intake manifold absolute pressure (B) | Emme manifold basıncı (B) | 5 | — | — | kPa | — |
| `88` | `0188` | SCR Induce System | SCR teşvik sistemi | 13 | — | — | — | — |
| `89` | `0189` | Run time — AECD #11-#15 | Çalışma — AECD #11-#15 | 21 | — | — | saniye | — |
| `8A` | `018A` | Run time — AECD #16-#20 | Çalışma — AECD #16-#20 | 21 | — | — | saniye | — |
| `8B` | `018B` | Diesel Aftertreatment | Dizel arıtma sistemi | 7 | — | — | — | — |
| `8C` | `018C` | O2 Sensor (Wide Range) | O2 sensörü (geniş bant) | 17 | — | — | µA | — |
| `8D` | `018D` | Throttle position G | Gaz kelebeği G | 1 | — | — | % | — |
| `8E` | `018E` | Engine friction — percent torque | Motor sürtünme — tork yüzdesi | 1 | — | — | % | — |
| `90` | `0190` | PIDs supported [91-C0] | Desteklenen PID'ler [91-C0] | 4 | — | — | Bitmap | — |
| `91` | `0191` | PM sensor bank 1 & 2 | PM sensörü bank 1 & 2 | 5 | — | — | mg/m³ | — |
| `92` | `0192` | WWH-OBD vehicle OBD system info | WWH-OBD araç OBD sistem bilgisi | 5 | — | — | — | — |
| `93` | `0193` | WWH-OBD vehicle OBD counters | WWH-OBD araç OBD sayaçları | 5 | — | — | — | — |
| `94` | `0194` | NOx warning and inducement | NOx uyarı ve teşvik | 12 | — | — | — | — |
| `98` | `0198` | Exhaust gas temperature sensor (wide) | Egzoz sıcaklık sensörü (geniş) | 9 | — | — | °C | — |
| `99` | `0199` | Exhaust gas temperature sensor bank | Egzoz sıcaklık Bank | 9 | — | — | °C | — |
| `9A` | `019A` | Hybrid/EV vehicle system data | Hibrit/EV araç sistemi | 7 | — | — | — | — |
| `9B` | `019B` | Diesel exhaust fluid sensor data | Dizel egzoz sıvısı sensörü | 7 | — | — | — | — |
| `9C` | `019C` | O2 Sensor Data | O2 sensör verisi | 17 | — | — | — | — |
| `9D` | `019D` | Engine fuel rate (multi-fuel) | Motor yakıt oranı (çoklu yakıt) | 9 | — | — | g/s | — |
| `9E` | `019E` | Engine exhaust flow rate | Motor egzoz akış oranı | 5 | — | — | kg/h | — |
| `9F` | `019F` | Fuel system percentage use | Yakıt sistemi kullanım yüzdesi | 9 | — | — | % | — |
| `A0` | `01A0` | PIDs supported [A1-C0] | Desteklenen PID'ler [A1-C0] | 4 | — | — | Bitmap | — |
| `A1` | `01A1` | NOx sensor corrected data | NOx sensörü düzeltilmiş veri | 5 | — | — | ppm | — |
| `A2` | `01A2` | Cylinder fuel rate | Silindir yakıt oranı | 1 | — | — | mg/stroke | — |
| `A3` | `01A3` | EVAP system vapor pressure (ultra) | Buharlaşma basıncı (ultra) | 5 | — | — | Pa | — |
| `A4` | `01A4` | Transmission actual gear | Şanzıman gerçek vites | 4 | — | — | ratio | — |
| `A5` | `01A5` | Diesel Exhaust Fluid Dosing | DEF dozajlama | 4 | — | — | — | — |
| `A6` | `01A6` | Odometer | Kilometre sayacı | 4 | 0 | — | km | `((A×2^24)+(B×2^16)+(C×2^8)+D) / 10` |
| `A7` | `01A7` | NOx sensor concentration 3+4 | NOx konsantrasyon 3+4 | 9 | — | — | ppm | — |
| `A8` | `01A8` | NOx sensor corrected 3+4 | NOx düzeltilmiş 3+4 | 9 | — | — | ppm | — |
| `A9` | `01A9` | ABS Disable State | ABS devre dışı durumu | 1 | — | — | Bitmap | — |

### Ek PID'ler (Uygulamadan Çıkarılan — 0xAD → 0xFF)

| PID | Komut | Açıklama |
|-----|-------|----------|
| `AD` | `01AD` | Exhaust pressure control |
| `B1` | `01B1` | Sensor calibration data |
| `B3` | `01B3` | Sensor calibration data 2 |
| `B8` | `01B8` | EV / Hybrid data |
| `B9` | `01B9` | EV / Hybrid data 2 |
| `BB` | `01BB` | EV / Hybrid data 3 |
| `BF` | `01BF` | Extended data |
| `C0-FF` | `01C0-01FF` | Manufacturer specific extended PIDs |

---

## 6. Mode 02 — Freeze Frame

**Komut:** `02XX\r` (XX = PID numarası, Mode 01 ile aynı PID'ler)  
**Yanıt:** `42 XX FF AA BB...` (FF = freeze frame numarası, genelde 00)

Freeze Frame, bir DTC oluştuğu andaki sensör değerlerinin snapshot'ıdır.  
Mode 01'deki aynı PID numaraları ve formüller geçerlidir.

**Uygulamada bulunan: 146 Freeze Frame PID**

---

## 7. Mode 03 — DTC (Arıza Kodu) Okuma

### Komut
```
03\r
```

### Yanıt Formatı
```
43 XX YY ZZ WW ...
```
Her 2 byte bir DTC kodunu temsil eder.

### DTC Parse Algoritması

```
Byte 1 (XX): İlk yarım byte → Harf, İkinci yarım byte → İlk rakam
Byte 2 (YY): İki rakam daha

Harf belirleme (Byte1 >> 6):
  0 = P (Powertrain)
  1 = C (Chassis)
  2 = B (Body)
  3 = U (Network/Communication)

İkinci karakter (Byte1 bits 4-5):
  0-3 arası

Kalan: Byte1 bits 0-3 + Byte2

Örnek: Byte1=01, Byte2=33 → P0133
  01 = 0000 0001 → P + 0 + 1
  33 = 0011 0011 → 3 + 3
  Sonuç: P0133 = O2 Sensor Circuit Slow Response
```

### Yanıt "NO DATA" ise → Kayıtlı DTC yok (iyi haber!)

---

## 8. Mode 04 — DTC Silme

### Komut
```
04\r
```

### Yanıt
```
44    → Başarılı (DTC'ler silindi, MIL söndürüldü)
```

**DİKKAT:** Bu komut:
- Tüm DTC'leri siler
- MIL (Check Engine) ışığını söndürür
- Freeze frame verilerini siler
- Hazırlık monitörlerini sıfırlar
- Araç, tüm monitörleri yeniden tamamlayana kadar "ready" olmayabilir

---

## 9. Mode 05 — O2 Sensör Monitör Testi

**Komut:** `05TT00\r` (TT = Test ID)  
**Yanıt:** `45 TT 00 AA BB CC DD`

| Test ID | Açıklama |
|---------|----------|
| `01` | Rich to lean sensor threshold voltage |
| `02` | Lean to rich sensor threshold voltage |
| `03` | Low sensor voltage for switch time calculation |
| `04` | High sensor voltage for switch time calculation |
| `05` | Rich to lean sensor switch time |
| `06` | Lean to rich sensor switch time |
| `07` | Minimum sensor voltage for test cycle |
| `08` | Maximum sensor voltage for test cycle |
| `09` | Time between sensor transitions |

**NOT:** CAN protokolünde Mode 05 desteklenmez, Mode 06 kullanılır.

**Uygulamada bulunan: 142 PID**

---

## 10. Mode 06 — On-Board Monitoring

**Komut:** `06TT\r` (TT = Test ID)  
**Yanıt:** `46 TT ... `

Bu mod, araç üzerindeki teşhis monitörlerinin test sonuçlarını döndürür.  
CAN araçlarda O2 sensör testleri de burada yapılır.

| Test | Açıklama |
|------|----------|
| `00` | Desteklenen test ID'leri |
| `01-FF` | Üretici ve standart test sonuçları |

**Uygulamada bulunan: 141 PID**

---

## 11. Mode 09 — Araç Bilgileri

**Komut:** `09XX\r`  
**Yanıt:** `49 XX ...`

### Standart PID'ler

| PID | Komut | Bilgi | Açıklama |
|-----|-------|-------|----------|
| `00` | `0900` | Supported PIDs [01-20] | Desteklenen PID'ler |
| `01` | `0901` | VIN message count | VIN için mesaj sayısı |
| `02` | `0902` | **VIN** | Araç Kimlik Numarası (17 karakter ASCII) |
| `03` | `0903` | Calibration ID message count | Kalibrasyon ID mesaj sayısı |
| `04` | `0904` | **Calibration ID** | ECU kalibrasyon kimliği |
| `05` | `0905` | CVN message count | CVN mesaj sayısı |
| `06` | `0906` | **CVN** | Kalibrasyon Doğrulama Numarası |
| `07` | `0907` | In-use performance tracking count | Performans izleme mesaj sayısı |
| `08` | `0908` | **In-use perf. tracking (spark)** | Ateşlemeli motor performans izleme |
| `09` | `0909` | ECU name message count | ECU isim mesaj sayısı |
| `0A` | `090A` | **ECU name** | ECU ismi (ASCII) |
| `0B` | `090B` | **In-use perf. tracking (compression)** | Sıkıştırmalı motor perf. izleme |
| `0C` | `090C` | ESN message count | Motor seri no mesaj sayısı |
| `0D` | `090D` | **ESN** | Motor Seri Numarası |
| `0E` | `090E` | Exhaust regulation | Egzoz regülasyonu |
| `0F` | `090F` | WWH-OBD ECU OBD system info | WWH-OBD ECU bilgisi |
| `10` | `0910` | WWH-OBD in-use monitoring | WWH-OBD kullanım izleme |

### VIN Parse Örneği

```
Gönder: 0902\r
Yanıt: 49 02 01 57 42 41 50 46 36 31 32 34 35 36 37 38 39 30
Parse: 49 02 = Mode 09 PID 02 yanıtı
       01 = mesaj dizisi
       57 42 41... = ASCII → "WBAPF612456789.0"
```

### Tam Mode 09 PID Listesi (139 adet)

```
0901, 0902, 0903, 0904, 0906, 0907, 0908, 0909, 090A, 090B,
090D, 090E, 090F, 0910, 0914, 0915, 0916, 0917, 0918, 0919,
091A, 091C, 091F, 0920, 0921, 0923, 0924, 0925, 0926, 0927,
0928, 0929, 092E, 092F, 0930, 0931, 0932, 0934, 0935, 0936,
0937, 0939, 093B, 093D, 0943, 0944, 0947, 0948, 094C, 094E,
094F, 0950, 0951, 0952, 0953, 0954, 0955, 0956, 0957, 0959,
095A, 095F, 0961, 0963, 0964, 0965, 096A, 096B, 096C, 096E,
0970, 0977, 0978, 097B, 097D, 097E, 0980, 0983, 0985, 0987,
098A, 098B, 098F, 0990, 0991, 0992, 0993, 0996, 0997, 0998,
099D, 099E, 09A0, 09A1, 09A2, 09A5, 09A9, 09AB, 09AD, 09B1,
09B2, 09B3, 09B5, 09B8, 09B9, 09BA, 09BB, 09BD, 09BF, 09C1,
09C3, 09C4, 09C5, 09C8, 09C9, 09CD, 09CF, 09D0, 09D4, 09D6,
09D9, 09DA, 09DC, 09DE, 09E0, 09E4, 09E6, 09E7, 09EC, 09ED,
09F1, 09F2, 09F3, 09F9, 09FA, 09FB, 09FC, 09FE, 09FF
```

---

## 12. Mode 22 — Extended Diagnostics (UDS)

### UDS Service 0x22: Read Data By Identifier

**Komut:** `22XXXX\r` (XXXX = Data Identifier / DID)  
**Yanıt:** `62 XX XX AA BB CC...`

Bu mod SAE J1979 standardında değildir. UDS (ISO 14229) protokolünü kullanır.  
Üretici özel verilere erişmek için kullanılır.

### Header Ayarlama Gereksinimi

Extended PID'ler için önce ECU header'ı ayarlamak gerekebilir:

```
ATSH 7E0\r       → Motor ECU'ya yönlendir
22F190\r         → VIN oku (UDS)
22F18C\r         → ECU seri numarası oku
22F187\r         → Parça numarası oku
```

### Bilinen UDS DID'ler (Standard)

| DID | Komut | Açıklama |
|-----|-------|----------|
| `F186` | `22F186` | Active diagnostic session |
| `F187` | `22F187` | Spare part number |
| `F188` | `22F188` | ECU software version |
| `F189` | `22F189` | ECU calibration version |
| `F18A` | `22F18A` | System supplier ID |
| `F18B` | `22F18B` | ECU manufacturing date |
| `F18C` | `22F18C` | ECU serial number |
| `F190` | `22F190` | VIN (UDS yöntemi) |
| `F191` | `22F191` | ECU hardware version |
| `F194` | `22F194` | System name |
| `F195` | `22F195` | Software version |

### Üretici Özel DID'ler (Uygulamadan Çıkarılan)

| DID | Komut | Olası İşlev |
|-----|-------|-------------|
| `2101` | `222101` | Hyundai/Kia — genişletilmiş motor verisi |
| `2102` | `222102` | Hyundai/Kia — ek sensörler |
| `2103` | `222103` | Hyundai/Kia — şanzıman verisi |
| `2104` | `222104` | Hyundai/Kia — klima sistemi |
| `2105` | `222105` | Hyundai/Kia — akü/şarj |
| `2106` | `222106` | Hyundai/Kia — oksijen sensörleri |
| `2107` | `222107` | Hyundai/Kia — katalizör |
| `2108` | `222108` | Hyundai/Kia — EVAP sistemi |
| `2200` | `222200` | VW/Audi — genişletilmiş parametre 1 |
| `2201` | `222201` | VW/Audi — genişletilmiş parametre 2 |
| `2202` | `222202` | VW/Audi — genişletilmiş parametre 3 |
| `2204` | `222204` | VW/Audi — turbo basıncı |
| `2205` | `222205` | VW/Audi — enjeksiyon verisi |
| `F400` | `22F400` | Renault/Dacia — genişletilmiş veri |
| `F4xx` | `22F4xx` | WWH-OBD (ISO 27145-2) verileri |

### Hyundai/Kia 2101 Parse Yapısı

```
Gönder: 2101\r
Yanıt: 6101 XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX

Byte  1-2:  Engine RPM                  (A×256+B) / 4
Byte  3:    Coolant Temperature         A − 40
Byte  4:    Intake Air Temperature      A − 40
Byte  5-6:  MAF Air Flow                (A×256+B) / 100
Byte  7:    Throttle Position           A × 100 / 255
Byte  8-9:  Vehicle Speed               (A×256+B)
Byte  10:   Battery Voltage             A / 10
Byte  11:   Engine Load                 A × 100 / 255
...
(toplam ~32 byte veri)
```

### Nissan Consult II Özel PID Yapısı

```
Header: ATSH 7E0\r (veya uygun ECU adresi)

Nissan Consult II tescilli bir protokoldür.
ELM327 üzerinden çalışması için özel init sekansı gerekir:
  - ATSH 7E0\r
  - Consult II init komutu gönder
  - Veri talep et

Nissan Consult 3 (yeni araçlar, CAN 11-bit):
  - Standart CAN protokolü üzerinden çalışır
  - UDS Service 0x22 kullanır
```

### VW TP 2.0 Protokolü

```
VW TP 2.0 (Transport Protocol):
  - VW Group araçlarında kullanılan özel taşıma protokolü
  - CAN üzerinde çalışır
  - Kanal kurulumu gerektirir
  - Hedef ECU'ya bağlantı açılır, sonra UDS komutları gönderilir
  
Desteklenen platformlar:
  - MQB (Modular Querbaukasten) — 2012+
  - PQ26 — Polo, Ibiza vb.
  
İlgili araçlar:
  - VW Arteon, Atlas/Teramont, Tiguan II
  - Skoda Kodiaq, Karoq, Octavia A7, Superb MK3
  - Seat Arona, Ateca, Ibiza 5, Toledo 4
  - Audi A1 Mk2, A3 Mk3, TT Mk3, Q2, Q3 Mk2
```

---

## 13. DTC (Arıza Kodu) Yapısı

### DTC Format

```
XNNNN

X = Kategori harfi:
  P = Powertrain (Motor/Şanzıman)
  C = Chassis (Şasi)
  B = Body (Gövde)
  U = Network/Communication (Ağ/İletişim)

N = 4 haneli sayı (hex)

İlk hane:
  0 = SAE/ISO standardı (genel)
  1 = Üretici özel
  2 = SAE/ISO standardı (genişletilmiş)
  3 = Üretici özel + SAE
```

### Yaygın DTC Kodları

| DTC | Açıklama |
|-----|----------|
| **P0100-P0199** | Hava akış / MAF / MAP sensörleri |
| **P0200-P0299** | Yakıt sistemi / Enjektörler |
| **P0300-P0399** | Ateşleme sistemi / Misfire |
| **P0400-P0499** | Emisyon kontrol (EGR, EVAP) |
| **P0500-P0599** | Araç hızı / Idle kontrol |
| **P0600-P0699** | ECU iç arızalar |
| **P0700-P0799** | Şanzıman |
| **P0800-P0899** | Şanzıman (devam) |
| **P2000-P2999** | SAE genişletilmiş kodlar |
| **P3000-P3499** | SAE İLERİ |
| **C0001-C0999** | Şasi — Genel |
| **B0001-B0999** | Gövde — Genel |
| **U0001-U0999** | İletişim — Genel |
| **U0100-U0399** | CAN iletişim kayıpları |

### DTC Okuma Kodu (Pseudocode)

```python
def parse_dtc(byte1, byte2):
    # Harf belirleme
    letter_map = {0: 'P', 1: 'C', 2: 'B', 3: 'U'}
    letter = letter_map[(byte1 >> 6) & 0x03]
    
    # İkinci karakter
    second = (byte1 >> 4) & 0x03
    
    # Kalan rakamlar
    third = byte1 & 0x0F
    fourth = (byte2 >> 4) & 0x0F
    fifth = byte2 & 0x0F
    
    return f"{letter}{second}{third:X}{fourth:X}{fifth:X}"

# Örnek: byte1=0x01, byte2=0x33 → P0133
```

---

## 14. Üretici Özel Protokoller

### Hyundai / Kia

| Profil | Yıl | Protokol | Özel PID'ler |
|--------|-----|----------|-------------|
| `HyundaiKia` | Tümü | Otomatik | 2101-2108 |
| `HyundaiKia 2015+` | 2015+ | CAN 11-bit | Extended 2101+ |
| `HyundaiKiaCAN` | CAN araçlar | CAN | 2101-2108 |
| `HyundaiKia.CRDI` | Dizel | CAN | CRDI özel verileri |

### Nissan / Infiniti

| Profil | Yıl | Protokol | Özel |
|--------|-----|----------|------|
| `Nissan Consult II` | 1989-2010 | K-Line tescilli | Consult II özel |
| `Nissan Consult 3` | 2006+ | CAN 11-bit | UDS tabanlı |
| `Nissan_22` | — | UDS | Service 0x22 |
| `NissanCVT` | — | — | CVT şanzıman |
| `Nissan Leaf` | — | — | EV akü verisi |

### VW Group (VW, Audi, Skoda, Seat)

| Profil | Platform | Özel |
|--------|----------|------|
| `VW TP 2.0` | MQB/PQ26 | Transport Protocol 2.0 |
| `VW Arteon` | MQB | 2017+ |
| `VW Tiguan II / FL` | MQB | 2016+ |
| `VW Polo 5 FL (6C)` | PQ26 | 2014-2018 |
| `Skoda Kodiaq/Karoq` | MQB | 2017+ |
| `Skoda Octavia A7` | MQB | 2013+ |
| `Skoda SuperB MK3 MY 2020+` | MQB | 2020+ |
| `Seat Arona/Ibiza 5 (KJ)` | MQB-A0 | 2017+ |
| `Seat Ateca` | MQB | 2016+ |
| `Audi A1 Mk2, A3 Mk3, TT Mk3, Q2, Q3 Mk2` | MQB | 2012+ |

### Toyota / Lexus

| Profil | Protokol |
|--------|----------|
| `TOYOTA` | CAN standart |
| `ToyotaJDM` | Japonya iç pazar özel |
| `ToyotaTPMS` | Lastik basıncı izleme |

### Diğer Üreticiler

| Üretici | Profil | Özel Protokol |
|---------|--------|---------------|
| Renault/Dacia | `Renault/Nissan/Dacia` | Özel PID'ler |
| Mitsubishi | `Mitsubishi MUT-2` | MUT-2 tescilli |
| Chevrolet/GM/Opel | `GM-Opel` | CAN + J1850 |
| Jeep/Chrysler/Dodge | `JeepChryslerDodge` | CAN UDS |
| Subaru | `SubaruDTC` | Özel DTC formatı |
| VAZ/Lada | `VAZ/Lada` | K-Line + özel |
| Alfa Romeo | `AlfaRomeo` | CAN |
| Jaguar/Land Rover | `JaguarLandRover` | CAN UDS |
| Porsche | `Porsche` | CAN UDS |

---

## 15. Bağlantı Profilleri

| # | Profil Adı | Yıl | Protokol | Açıklama |
|---|-----------|-----|----------|----------|
| 1 | `OBDII/EOBD` | Tümü | Otomatik | Genel profil — her araçta dene |
| 2 | `OBD-II / EOBD + CAN 11 bit` | 2004+ | CAN 500k | Modern araçlar |
| 3 | `OBD-II / EOBD Diesel + CAN 11 bit` | 2004+ | CAN 500k | Dizel araçlar |
| 4 | `OBD-II / EOBD ~1999-2008 K-Line/KWP + extra sensors` | 1999-2008 | KWP2000 | Eski + ek sensörler |
| 5 | `OBD-II / EOBD ~2006-2009 CAN + extra sensors` | 2006-2009 | CAN | Geçiş dönemi + ek |
| 6 | `OBD-II / EOBD ~2010-2022 CAN + extra sensors` | 2010-2022 | CAN | Modern + ek sensörler |
| 7 | `OBD-II / EOBD ~2016+ CAN + extra sensors` | 2016+ | CAN | En yeni + ek sensörler |
| 8 | `OBD-II / EOBD + CAN and extra PIDs` | 2006+ | CAN | CAN + genişletilmiş PID'ler |
| 9 | `OBD-II + SAE J1850` | <2004 | J1850 | Eski Ford/GM |
| 10 | `OBDII/EOBD + SAE J1850 (Old Ford vehicles)` | Eski | J1850 PWM | Eski Ford kamyon |
| 11 | `WWH-OBD + CAN and extra PIDs (2025-)` | 2025+ | ISO 27145-2 | Yeni nesil |

---

## 16. Veri Parse Formülleri

### Genel Formül Tipleri

```python
# TİP 1: Direkt değer
value = A                              # Örnek: PID 0D (hız), PID 0B (MAP)

# TİP 2: Offset (−40)
value = A - 40                         # Örnek: PID 05 (soğutma suyu), PID 0F (emme havası)

# TİP 3: Yüzde (0-100%)
value = A * 100 / 255                  # Örnek: PID 04 (yük), PID 11 (gaz kelebeği)

# TİP 4: İşaretli yüzde (−100 ~ +99.2%)
value = (A - 128) * 100 / 128          # Örnek: PID 06-09 (yakıt trim)

# TİP 5: 16-bit direkt
value = (A * 256) + B                  # Örnek: PID 1F (çalışma süresi)

# TİP 6: 16-bit bölümlü
value = ((A * 256) + B) / 4            # Örnek: PID 0C (RPM)
value = ((A * 256) + B) / 100          # Örnek: PID 10 (MAF)
value = ((A * 256) + B) / 1000         # Örnek: PID 42 (voltaj)

# TİP 7: 16-bit çarpmatlı
value = ((A * 256) + B) * 0.079        # Örnek: PID 22 (yakıt basıncı)
value = ((A * 256) + B) * 10           # Örnek: PID 23 (dizel basınç)

# TİP 8: Sıcaklık (geniş aralık)
value = ((A * 256) + B) / 10 - 40      # Örnek: PID 3C-3F (katalizör)

# TİP 9: Lambda
value = ((A * 256) + B) * 2 / 65536    # Ratio
value = ((C * 256) + D) * 8 / 65536    # Voltaj

# TİP 10: İşaretli offset
value = (A / 2) - 64                    # Örnek: PID 0E (timing advance)
value = A - 125                          # Örnek: PID 61-62 (tork)

# TİP 11: 32-bit (Odometer)
value = ((A<<24) + (B<<16) + (C<<8) + D) / 10   # PID A6
```

### Tam Parse Kodu (Python)

```python
class OBDParser:
    @staticmethod
    def parse_response(raw_response):
        """
        ELM327 yanıtını parse et.
        raw_response: "41 0C 1A F8" gibi string
        """
        # Boşlukları temizle
        clean = raw_response.replace(" ", "").strip()
        
        # ">" prompt'unu kaldır
        clean = clean.replace(">", "")
        
        # Hata kontrolü
        if clean in ("NODATA", "ERROR", "?", "UNABLETOCONNECT", "CANERROR"):
            return None
        
        # Byte'lara ayır
        bytes_list = [int(clean[i:i+2], 16) for i in range(0, len(clean), 2)]
        
        if len(bytes_list) < 2:
            return None
        
        mode = bytes_list[0] - 0x40  # 41→01, 42→02 vb.
        pid = bytes_list[1]
        data = bytes_list[2:]
        
        return mode, pid, data
    
    @staticmethod
    def calculate_pid(pid, data):
        """PID değerini hesapla."""
        A = data[0] if len(data) > 0 else 0
        B = data[1] if len(data) > 1 else 0
        C = data[2] if len(data) > 2 else 0
        D = data[3] if len(data) > 3 else 0
        
        formulas = {
            0x04: lambda: A * 100.0 / 255,                    # Engine Load %
            0x05: lambda: A - 40,                               # Coolant Temp °C
            0x06: lambda: (A - 128) * 100.0 / 128,            # STFT Bank 1 %
            0x07: lambda: (A - 128) * 100.0 / 128,            # LTFT Bank 1 %
            0x08: lambda: (A - 128) * 100.0 / 128,            # STFT Bank 2 %
            0x09: lambda: (A - 128) * 100.0 / 128,            # LTFT Bank 2 %
            0x0A: lambda: A * 3,                                # Fuel Pressure kPa
            0x0B: lambda: A,                                    # MAP kPa
            0x0C: lambda: ((A * 256) + B) / 4.0,              # RPM
            0x0D: lambda: A,                                    # Speed km/h
            0x0E: lambda: (A / 2.0) - 64,                     # Timing ° BTDC
            0x0F: lambda: A - 40,                               # Intake Temp °C
            0x10: lambda: ((A * 256) + B) / 100.0,            # MAF g/s
            0x11: lambda: A * 100.0 / 255,                     # Throttle %
            0x1F: lambda: (A * 256) + B,                        # Runtime sec
            0x21: lambda: (A * 256) + B,                        # Distance w/MIL km
            0x22: lambda: ((A * 256) + B) * 0.079,            # Fuel Rail kPa
            0x23: lambda: ((A * 256) + B) * 10,               # Fuel Rail kPa (diesel)
            0x2C: lambda: A * 100.0 / 255,                     # EGR %
            0x2D: lambda: (A - 128) * 100.0 / 128,            # EGR Error %
            0x2E: lambda: A * 100.0 / 255,                     # EVAP Purge %
            0x2F: lambda: A * 100.0 / 255,                     # Fuel Level %
            0x30: lambda: A,                                    # Warm-ups count
            0x31: lambda: (A * 256) + B,                        # Distance since DTC km
            0x33: lambda: A,                                    # Baro Pressure kPa
            0x3C: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B1S1 °C
            0x3D: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B2S1 °C
            0x3E: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B1S2 °C
            0x3F: lambda: ((A * 256) + B) / 10.0 - 40,        # Cat Temp B2S2 °C
            0x42: lambda: ((A * 256) + B) / 1000.0,           # Module Voltage V
            0x43: lambda: ((A * 256) + B) * 100.0 / 255,      # Absolute Load %
            0x44: lambda: ((A * 256) + B) / 32768.0,           # Lambda ratio
            0x45: lambda: A * 100.0 / 255,                     # Relative Throttle %
            0x46: lambda: A - 40,                               # Ambient Temp °C
            0x47: lambda: A * 100.0 / 255,                     # Throttle B %
            0x48: lambda: A * 100.0 / 255,                     # Throttle C %
            0x49: lambda: A * 100.0 / 255,                     # Accel Pedal D %
            0x4A: lambda: A * 100.0 / 255,                     # Accel Pedal E %
            0x4B: lambda: A * 100.0 / 255,                     # Accel Pedal F %
            0x4C: lambda: A * 100.0 / 255,                     # Cmd Throttle %
            0x4D: lambda: (A * 256) + B,                        # Time w/MIL min
            0x4E: lambda: (A * 256) + B,                        # Time since DTC min
            0x51: lambda: A,                                    # Fuel Type enum
            0x52: lambda: A * 100.0 / 255,                     # Ethanol %
            0x5A: lambda: A * 100.0 / 255,                     # Rel Accel Pedal %
            0x5B: lambda: A * 100.0 / 255,                     # Hybrid Battery %
            0x5C: lambda: A - 40,                               # Oil Temp °C
            0x5D: lambda: (((A * 256) + B) - 26880) / 128.0,  # Injection Timing °
            0x5E: lambda: ((A * 256) + B) / 20.0,             # Fuel Rate L/h
            0x61: lambda: A - 125,                              # Driver Demand %
            0x62: lambda: A - 125,                              # Actual Torque %
            0x63: lambda: (A * 256) + B,                        # Ref. Torque Nm
            0xA6: lambda: ((A<<24)+(B<<16)+(C<<8)+D) / 10.0,  # Odometer km
        }
        
        if pid in formulas:
            return formulas[pid]()
        return None
```

### DTC Parse Kodu

```python
def parse_dtcs(response):
    """
    Mode 03 yanıtını parse et.
    response: "43 01 33 02 34 00 00" gibi
    """
    clean = response.replace(" ", "")
    if clean.startswith("43"):
        clean = clean[2:]  # "43" prefix'i kaldır
    
    dtcs = []
    letter_map = {0: 'P', 1: 'C', 2: 'B', 3: 'U'}
    
    for i in range(0, len(clean), 4):
        if i + 4 > len(clean):
            break
        byte1 = int(clean[i:i+2], 16)
        byte2 = int(clean[i+2:i+4], 16)
        
        if byte1 == 0 and byte2 == 0:
            continue  # Boş slot
        
        letter = letter_map[(byte1 >> 6) & 0x03]
        second = (byte1 >> 4) & 0x03
        third = byte1 & 0x0F
        fourth = (byte2 >> 4) & 0x0F
        fifth = byte2 & 0x0F
        
        dtc = f"{letter}{second}{third:X}{fourth:X}{fifth:X}"
        dtcs.append(dtc)
    
    return dtcs
```

### ELM327 Bağlantı Kodu

```python
import socket
import serial
import time

class ELM327Connection:
    """ELM327 adaptörüne bağlan ve komut gönder."""
    
    def __init__(self):
        self.conn = None
        self.protocol = None
    
    # ===== WiFi Bağlantı =====
    def connect_wifi(self, ip="192.168.0.10", port=35000, timeout=10):
        self.conn = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.conn.settimeout(timeout)
        self.conn.connect((ip, port))
        self._init_elm()
    
    # ===== Bluetooth Bağlantı =====
    def connect_bluetooth(self, port, baudrate=38400, timeout=10):
        """
        port: COM portu (Windows) veya /dev/rfcomm0 (Linux)
        """
        self.conn = serial.Serial(port, baudrate, timeout=timeout)
        self._init_elm()
    
    # ===== Init Sekansı =====
    def _init_elm(self):
        """ELM327 başlatma sekansı."""
        responses = {}
        responses['reset'] = self.send_command("ATZ")       # Reset
        time.sleep(1)
        responses['echo'] = self.send_command("ATE0")       # Echo off
        responses['spaces'] = self.send_command("ATS0")      # Spaces off
        responses['linefeed'] = self.send_command("ATL0")    # Linefeed off
        responses['headers'] = self.send_command("ATH0")     # Headers off
        responses['longmsg'] = self.send_command("ATAL")     # Allow long msgs
        responses['protocol'] = self.send_command("ATSP0")   # Auto protocol
        responses['voltage'] = self.send_command("ATRV")     # Battery voltage
        return responses
    
    # ===== Komut Gönder =====
    def send_command(self, cmd, timeout=5):
        """
        ELM327'ye komut gönder ve yanıtı al.
        """
        cmd_bytes = (cmd + "\r").encode('ascii')
        
        if isinstance(self.conn, socket.socket):
            self.conn.send(cmd_bytes)
            response = b""
            start = time.time()
            while time.time() - start < timeout:
                try:
                    chunk = self.conn.recv(1024)
                    response += chunk
                    if b">" in chunk:
                        break
                except socket.timeout:
                    break
        else:  # Serial
            self.conn.write(cmd_bytes)
            response = b""
            start = time.time()
            while time.time() - start < timeout:
                if self.conn.in_waiting:
                    chunk = self.conn.read(self.conn.in_waiting)
                    response += chunk
                    if b">" in chunk:
                        break
                time.sleep(0.01)
        
        return response.decode('ascii', errors='ignore').strip().replace(">","")
    
    # ===== PID Oku =====
    def read_pid(self, mode, pid):
        """
        OBD PID'i oku.
        mode: 1-9 (int)
        pid: 0x00-0xFF (int)
        """
        cmd = f"{mode:02X}{pid:02X}"
        return self.send_command(cmd)
    
    # ===== Canlı Veri Oku =====
    def get_rpm(self):
        resp = self.read_pid(1, 0x0C)
        mode, pid, data = OBDParser.parse_response(resp)
        return ((data[0] * 256) + data[1]) / 4.0
    
    def get_speed(self):
        resp = self.read_pid(1, 0x0D)
        mode, pid, data = OBDParser.parse_response(resp)
        return data[0]
    
    def get_coolant_temp(self):
        resp = self.read_pid(1, 0x05)
        mode, pid, data = OBDParser.parse_response(resp)
        return data[0] - 40
    
    def get_fuel_level(self):
        resp = self.read_pid(1, 0x2F)
        mode, pid, data = OBDParser.parse_response(resp)
        return data[0] * 100.0 / 255
    
    def get_voltage(self):
        resp = self.send_command("ATRV")
        return float(resp.replace("V","").strip())
    
    def get_vin(self):
        resp = self.read_pid(9, 0x02)
        # ASCII parse
        mode, pid, data = OBDParser.parse_response(resp)
        return bytes(data[1:]).decode('ascii', errors='ignore')  # İlk byte = count
    
    def get_dtcs(self):
        resp = self.send_command("03")
        return parse_dtcs(resp)
    
    def clear_dtcs(self):
        return self.send_command("04")
    
    # ===== Desteklenen PID'leri Bul =====
    def get_supported_pids(self):
        """Araçta desteklenen tüm PID'leri tespit et."""
        supported = []
        
        for base_pid in [0x00, 0x20, 0x40, 0x60, 0x80, 0xA0, 0xC0, 0xE0]:
            resp = self.read_pid(1, base_pid)
            result = OBDParser.parse_response(resp)
            if result is None:
                break
            
            mode, pid, data = result
            bitmap = (data[0] << 24) | (data[1] << 16) | (data[2] << 8) | data[3]
            
            for i in range(32):
                if bitmap & (1 << (31 - i)):
                    supported.append(base_pid + i + 1)
        
        return supported
    
    # ===== Bağlantıyı Kapat =====
    def disconnect(self):
        if self.conn:
            self.conn.close()
```

---

## 17. Örnek Kod Yapıları

### Komut-Yanıt Tablosu

| # | Gönderilen | Ham Yanıt | Parse | Sonuç |
|---|-----------|-----------|-------|-------|
| 1 | `0100\r` | `41 00 BE 3E B8 13` | Bitmap | PID 01-20 desteği |
| 2 | `0104\r` | `41 04 64` | 100×100/255 | **39.2% yük** |
| 3 | `0105\r` | `41 05 7B` | 123−40 | **83°C soğutma** |
| 4 | `010C\r` | `41 0C 1A F8` | (6912+248)/4 | **1790 RPM** |
| 5 | `010D\r` | `41 0D 3C` | 60 | **60 km/h** |
| 6 | `010F\r` | `41 0F 46` | 70−40 | **30°C emme** |
| 7 | `0110\r` | `41 10 01 FA` | 506/100 | **5.06 g/s MAF** |
| 8 | `0111\r` | `41 11 33` | 51×100/255 | **20% gaz** |
| 9 | `012F\r` | `41 2F 80` | 128×100/255 | **50.2% yakıt** |
| 10 | `0142\r` | `41 42 30 D4` | 12500/1000 | **12.5V ECU** |
| 11 | `0146\r` | `41 46 37` | 55−40 | **15°C ortam** |
| 12 | `015C\r` | `41 5C 6E` | 110−40 | **70°C yağ** |
| 13 | `01A6\r` | `41 A6 00 01 86 A0` | 100000/10 | **10000.0 km** |
| 14 | `ATRV` | `12.6V` | Direkt | **12.6V akü** |
| 15 | `0902\r` | `49 02 01 57 42 41...` | ASCII | **VIN** |
| 16 | `03\r` | `43 01 33 00 00 00 00` | DTC parse | **P0133** |
| 17 | `04\r` | `44` | — | **DTC silindi** |

### Fuel Type Enum Değerleri (PID 51)

| Değer | Yakıt Tipi |
|-------|-----------|
| 0 | Not available |
| 1 | Gasoline (Benzin) |
| 2 | Methanol |
| 3 | Ethanol |
| 4 | Diesel |
| 5 | LPG |
| 6 | CNG |
| 7 | Propane |
| 8 | Electric |
| 9 | Bifuel — Gasoline |
| 10 | Bifuel — Methanol |
| 11 | Bifuel — Ethanol |
| 12 | Bifuel — LPG |
| 13 | Bifuel — CNG |
| 14 | Bifuel — Propane |
| 15 | Bifuel — Electric |
| 16 | Bifuel — Gasoline/Electric |
| 17 | Hybrid Gasoline |
| 18 | Hybrid Ethanol |
| 19 | Hybrid Diesel |
| 20 | Hybrid Electric |
| 21 | Hybrid Mixed |
| 22 | Hybrid Regenerative |
| 23 | Bifuel — Diesel |

### OBD Standards Enum (PID 1C)

| Değer | Standart |
|-------|---------|
| 1 | OBD-II (CARB) |
| 2 | OBD (EPA) |
| 3 | OBD + OBD-II |
| 4 | OBD-I |
| 5 | Not OBD compliant |
| 6 | EOBD (Europe) |
| 7 | EOBD + OBD-II |
| 8 | EOBD + OBD |
| 9 | EOBD + OBD + OBD-II |
| 10 | JOBD (Japan) |
| 11 | JOBD + OBD-II |
| 12 | JOBD + EOBD |
| 13 | JOBD + EOBD + OBD-II |
| 17 | EMD (Engine Manufacturer Diagnostics) |
| 18 | EMD+ |
| 19 | HD OBD-C |
| 20 | HD OBD |
| 21 | WWH OBD |
| 23 | HD EOBD-I |
| 24 | HD EOBD-I N |
| 25 | HD EOBD-II |
| 26 | HD EOBD-II N |
| 28 | OBDBR-1 (Brazil) |
| 29 | OBDBR-2 |
| 30 | KOBD (Korea) |
| 31 | IOBD I (India) |
| 32 | IOBD II |
| 33 | HD EOBD-IV |

### Monitor Status Bit Maskeleri (PID 01)

```
Byte A:
  Bit 7: MIL on/off (1=MIL açık, Check Engine yanıyor)
  Bit 6-0: DTC sayısı (0-127)

Byte B:
  Bit 0: Misfire test available
  Bit 1: Fuel system test available
  Bit 2: Components test available
  Bit 3: Reserved
  Bit 4: Misfire test incomplete
  Bit 5: Fuel system test incomplete
  Bit 6: Components test incomplete
  Bit 7: Reserved

Byte C (Benzin):
  Bit 0: Catalyst test available
  Bit 1: Heated catalyst available
  Bit 2: Evap system available
  Bit 3: Secondary air available
  Bit 4: A/C refrigerant available
  Bit 5: O2 sensor available
  Bit 6: O2 sensor heater available
  Bit 7: EGR system available

Byte C (Dizel):
  Bit 0: NMHC catalyst available
  Bit 1: NOx/SCR monitor available
  Bit 2: Reserved
  Bit 3: Boost pressure available
  Bit 4: Reserved
  Bit 5: Exhaust gas sensor available
  Bit 6: PM filter monitoring available
  Bit 7: EGR/VVT system available

Byte D: Aynı yapıda — test incomplete bitleri
```

---

## Özet İstatistikler

| Kategori | Sayı |
|----------|------|
| Mode 01 PID'leri | **150** |
| Mode 02 PID'leri (Freeze Frame) | **146** |
| Mode 03 PID'leri (DTC) | **158** |
| Mode 05 PID'leri (O2 Monitor) | **142** |
| Mode 06 PID'leri (On-Board) | **141** |
| Mode 09 PID'leri (Araç Bilgi) | **139** |
| Mode 22 Extended PID'ler (UDS) | **142** |
| **TOPLAM PID** | **~1018** |
| AT Komutları | **30+** |
| Desteklenen Protokoller | **10** |
| Üretici Profilleri | **25+** |
| Bağlantı Profilleri | **11** |

---

## Android İzinleri (Geliştirme İçin)

Kendi uygulamanızda kullanmanız gereken izinler:

```xml
<!-- Bluetooth Classic -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />

<!-- WiFi -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- Konum (BT tarama için Android 6+) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Arka plan servis -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />

<!-- Opsiyonel -->
<uses-feature android:name="android.hardware.bluetooth" android:required="false" />
<uses-feature android:name="android.hardware.location" android:required="false" />
```

---

*OBD-II / ELM327 Tam PID ve Komut Referansı — Oluşturulma: 23 Mart 2026*  
*Kaynak: Car Scanner ELM OBD2 (com.ovz.carscanner) reverse engineering analizi*
