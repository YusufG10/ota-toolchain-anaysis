
# OTA Firmware ELF Analizi — `new-firmware.z1`

**Ders:** BİL 304 İşletim Sistemleri · OMÜ Bilgisayar Mühendisliği · Bahar 2025/2026
**Araştırma İş Parçacığı — Hedef dosya:** `new-firmware.z1` (MSP430 / Z1)

> **Özet bulgu:** Dosya, MSP430 çekirdekli, CC2420 telsizli bir **Z1 mote** firmware'idir. Üzerinde **Contiki** işletim sistemi, **RPL/uIP** ağ yığını çalışmaktadır.

Kullanılan araçlar: `file`, `msp430-readelf`, `msp430-size`, `msp430-nm`, `msp430-objdump`, `msp430-strings`.

---

## 1. ELF Sınıfı, Mimari ve Giriş Adresi

```
$ msp430-readelf -h new-firmware.z1
  Class:        ELF32
  Data:         2's complement, little endian
  Type:         EXEC (Executable file)
  Machine:      Texas Instruments msp430 microcontroller
  Entry point:  0x3100
  OS/ABI:       Standalone App
```

| Alan | Değer | Anlamı |
|------|-------|--------|
| Sınıf | ELF32 | 32-bit ELF kapsayıcısı |
| Veri düzeni | little-endian | MSP430'un doğal bayt sırası |
| Tür | EXEC | Tam bağlanmış çalıştırılabilir imaj |
| Makine | TI MSP430 | Hedef mikrodenetleyici |
| **Giriş adresi** | **0x3100** | Açılışta ilk çalışacak komut |

**Yorum:** Tür `EXEC` olduğu için bu bir nesne dosyası (`.o`) ya da kütüphane değil, cihaza yüklenip koşacak **tam bağlanmış** bir imajdır. Giriş adresi `0x3100`, reset vektörünün işaret ettiği adresle birebir aynıdır (Bkz. §6).

---

## 2. Bölümler (.text, .data, .bss, .rodata, .vectors)

```
$ msp430-readelf -S new-firmware.z1   (özet)
.far.text  0x10000  0x4a78   AX
.text      0x03100  0x976e   AX
.rodata    0x0c870  0x35fd   A
.data      0x01100  0x0150   WA
.bss       0x01250  0x1648   WA  (NOBITS)
.vectors   0x0ffc0  0x0040   AX
```

| Bölüm | Adres | Boyut | Anlamı |
|-------|-------|-------|--------|
| `.text` | 0x3100 | 38.766 B | Asıl program kodu (64 KB altı) |
| `.far.text` | 0x10000 | 19.064 B | **64 KB sınırının üstündeki kod — MSP430X 20-bit adresleme** |
| `.rodata` | 0xC870 | 13.821 B | Salt-okunur sabitler, metin dizgeleri |
| `.data` | 0x1100 | 336 B | İlk değerli değişkenler (RAM) |
| `.bss` | 0x1250 | 5.704 B | Sıfırla başlayan değişkenler (RAM, NOBITS) |
| `.vectors` | 0xFFC0 | 64 B | Kesme vektör tablosu (belleğin tepesinde) |

**Öne çıkan iki nokta:**
1. **`.far.text`**: Kod 64 KB'ı aşmış, linker taşan kısmı MSP430X'in 20-bit genişletilmiş adres alanına (0x10000+) koymuş. İmajın neden ~127 KB olduğunun nedeni budur.
2. **`.bss` NOBITS'tir**: dosyada hiç yer kaplamaz, açılışta sadece sıfırlanır.

`.data`'nın çalışma adresi (VMA) `0x1100` (RAM) iken yükleme adresi (LMA) `0xFE6E` (Flash)'tır → ilk değerli değişkenler Flash'ta saklanır, açılışta startup kodu RAM'e kopyalar.

---

## 3. Kod ve Veri Boyutları

```
$ msp430-size new-firmware.z1
   text     data    bss     dec     hex
  71715      336   5706   77757   12fbd
```

| Bölge | Boyut | Yerleşim |
|-------|-------|----------|
| text | 71.715 B | `.text`+`.rodata`+`.far.text`+`.vectors` → **Flash** (kalıcı) |
| data | 336 B | İlk değerli değişkenler → **RAM** (kopyası Flash'ta) |
| bss | 5.706 B | Sıfır başlangıçlı → **RAM** |

**Yorum:** Flash ~72 KB (MSP430F2617'nin ~92 KB'ının %77'si). RAM = data+bss ≈ 6 KB, çipin ~8 KB RAM'inin büyük kısmı. Gömülü sistemde asıl darboğaz RAM olduğundan bu kritik bir kullanımdır.

### Bellek Yerleşim Diyagramı

![Bellek Haritası](memmap.png)

*Şekil 1. Bölümlerin MSP430F2617 (Z1) bellek haritası üzerine gerçek adreslerle yerleşimi.*

> Not: Bu imaj MSP430 (Z1) hedeflidir; harita bu yüzden MSP430F2617 üzerine çizilmiştir. Şartnamedeki "CC1352R bellek haritası" görselleştirmesi ARM (CC1352R) firmware'i içindir.

---

## 4. Sembol Tablosu ve Anlamlı Semboller

```
$ msp430-nm new-firmware.z1 | wc -l   → 1045 sembol
```
Tür dağılımı: 385 global fonksiyon (T), 145 yerel (t), 57+114 bss (B/b), 21+23 data (D/d), 237 mutlak/linker (A).

<img width="1614" height="1167" alt="memmap" src="https://github.com/user-attachments/assets/b1d0c1f8-bb41-4582-8567-91ae12895509" />


| Sembol grubu | Anlamı |
|--------------|--------|
| `main`, `_reset_vector__`, `__stack` | Startup ve bellek sınır sembolleri |
| `__bss_start/end`, `__data_load_start` | Açılışta RAM ilklendirme sınırları |
| `cc2420_init/send/receive/interrupt` | CC2420 telsiz sürücüsü (Z1) |
| `etimer_process`, `ctimer_process` | Contiki olay tabanlı zamanlayıcı process'leri |
| `rpl_*`, `tcpip`, `uip*` | RPL yönlendirme ve uIP ağ yığını |
| `hello_world_process`, `accmeter_process` | Uygulama katmanı (Z1 ivmeölçeri dahil) |

**Yorum:** Sembol tablosu korunmuş (stripped değil) → her fonksiyonun adı ve adresi görünür. Analiz ve hata ayıklama için kritik.

---

## 5. Kesme Vektörleri ve Başlangıç Adresi

```
$ msp430-objdump -s -j .vectors new-firmware.z1   (son satır)
 fff0 24368c37 d8377633 fe357633 76330031  ...v3.5v3v3.1
```

Tablonun çoğu girdisi `0x3376` → tanımsız/varsayılan kesme işleyicisi. Birkaç gerçek ISR farklı adreslere (0x36F4, 0x3778, 0x353E, 0x35C2) yönlenir (timer/telsiz).

**En kritik girdi:** `0xFFFE` reset vektörü, içeriği `00 31` (little-endian) = **`0x3100`** = ELF başlığındaki giriş adresi. Boot zinciri:

```
Enerji → CPU 0xFFFE'yi okur → 0x3100'e dallanır → startup → main (0x313E)
```

---

## 6. Neden "Ham Binary" Değil de "ELF Executable"?

Kanıtlar:
- **Derleyici izi** (`.comment`): `GCC (GNU) 4.7.2 ... mspgcc dev 20120911`
- **8 adet `.debug_*` bölümü** + `.symtab`/`.strtab` → kaynak satır eşlemesi ve sembol isimleri korunmuş
- Program/bölüm başlıkları her segmentin yükleme/çalışma adresini ayrı tanımlıyor (`.data` LMA≠VMA)

**Sonuç:** Ham binary, yapısı olmayan düz bayt yığınıdır; nereye yükleneceği, kod/veri ayrımı, `.data`'nın Flash'tan RAM'e kopyalanması bilinmez. ELF executable bütün bunları (header + program başlıkları + bölüm tablosu + sembol + debug) içinde taşır. `readelf`/`nm`/`objdump` ile bu kadar bilgiyi çıkarabilmemizin nedeni budur.

---

## 7. Sonuç

`new-firmware.z1`, Contiki tabanlı, CC2420 telsizli bir **Z1 (MSP430F2617)** düğümü için derlenmiş, RPL/uIP içeren bir **ELF32 çalıştırılabilir** imajdır. Kod 64 KB'ı aştığı için **MSP430X genişletilmiş adres alanı** (`.far.text`) kullanılmıştır. Flash'ın ~%77'si, RAM'in büyük kısmı doludur. Reset vektörü giriş adresiyle tutarlıdır, sembol/debug bilgisi korunmuştur. ~127 KB'lık bu yapılı imajın **tek pakette taşınamaması**, OTA aktarımında parçalama, sıralama ve bütünlük denetimini zorunlu kılar.
