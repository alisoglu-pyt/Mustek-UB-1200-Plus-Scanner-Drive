# Mustek 1200 UB Plus — Windows & Linux USB Sürücüsü & Tarama Merkezi

`USB\VID_05D8&PID_4002` (`05d8:4002`) donanım kimliğine sahip **Mustek ScanExpress 1200 UB Plus** (ve aynı GT-6801 yonga setini paylaşan **Mustek BearPaw 1200 CU**) tarayıcılar için geliştirilmiş çapraz platform (Windows 10/11 & Linux) kullanıcı modu USB sürücüsü ve masaüstü tarama uygulamasıdır.

## Hazır Derlenmiş Taşınabilir Paketler (`dist/`)

Kurulum veya Python ortamı gerektirmeden doğrudan çalıştırılabilir tek dosya paketler `dist/` klasöründedir:

1. **Windows 10 / 11 (64-bit) Taşınabilir Tek Dosya (`.exe`):**
   - `dist\Mustek_1200_UB_Plus-Win64.exe`
   - İçerisinde Python çalışma zamanı, `libusb-1.0.dll`, `SBfw.usb` / `PS1fw.usb` firmware dosyaları ve tek tıkla WinUSB kurulumu için `zadig-2.9.exe` gömülüdür.
2. **Linux (`x86_64`) Taşınabilir AppImage:**
   - `dist/Mustek_1200_UB_Plus-x86_64.AppImage`
   - Çalıştırmak için:
     ```bash
     chmod +x Mustek_1200_UB_Plus-x86_64.AppImage
     ./Mustek_1200_UB_Plus-x86_64.AppImage
     ```
   - Normal kullanıcı olarak USB erişimi için uygulama üst çubuğundaki **"🔐 Linux USB İzni Tanımla (udev)"** butonuna tıklayabilir veya `dist/99-mustek-1200ub.rules` dosyasını bir kez kurabilirsiniz:
     ```bash
     sudo cp dist/99-mustek-1200ub.rules /etc/udev/rules.d/
     sudo udevadm control --reload-rules && sudo udevadm trigger
     ```

---

## Teknik Mimari

1. **Alt Katman USB Erişimi (`WinUSB` / `libusb-1.0`):** Windows 10/11 64-bit çekirdek sürücü imzalama engeline takılmadan Microsoft'un yerleşik imzalı `WinUSB.sys` sürücüsünü, Linux'ta ise doğrudan `libusb-1.0` arayüzünü kullanır.
2. **GT-6801 Mikrodenetleyici Firmware Yükleyici (`driver/gt6801_protocol.py`):** Tarayıcıda kalıcı ROM bulunmadığından açılışta `firmware/SBfw.usb` (8051 mikrodenetleyici yazılımı, 7.792 bayt) 64 baytlık USB kontrol paketleriyle (`0x2009` / `0x200A`) dahili SRAM'e yüklenir ve `0x69 0x01 0xC0 0x1C` komutuyla önyüklenir.
3. **CIS Sensör Kalibrasyonu (`driver/calibration.py`):** Her tarama oturumunda otomatik AFE (Analog Kazanç/Offset), RGB LED pozlama dengesi ve piksel bazlı siyah/beyaz *shading* düzeltmesi uygulanarak dikey çizgilenmeler önlenir.
4. **Masaüstü Tarama Arayüzü (`ui/main_window.py`):**
   - Canlı Windows PnP (`CfgMgr32`) ve Linux USB (`udev` / `libusb`) bağlantı takibi
   - 75 DPI hızlı cam önizleme ve fareyle milimetrik kırpma bölgesi seçimi
   - 50, 75, 150, 300, 600 ve 1200 DPI tarama (Renkli, Gri Tonlama, Siyah-Beyaz Belge, 8-bit / 12-bit ADC)
   - Canlı satır satır görüntü akışı, parlaklık/kontrast/döndürme ve **Çok Sayfalı PDF / PNG / JPG / TIFF** dışa aktarma

---

## İlk Kurulum — Windows (Yalnızca 1 Kez Yapılır)

1. `dist\Mustek_1200_UB_Plus-Win64.exe` dosyasını (veya kaynak koddan `baslat.bat` / `python main.py`) çalıştırın.
2. Üst çubuktaki **"⚡ 1. Adım: Zadig ile WinUSB Kur"** butonuna tıklayın.
3. Zadig penceresinde:
   - Üst listede **`Unknown Device #1`** (`05D8` `4002`) seçili olduğundan emin olun *(görünmüyorsa `Options -> List All Devices` işaretleyin)*.
   - Hedef sürücü olarak **`WinUSB`** seçili iken **`Install Driver`** butonuna tıklayın.
4. Kurulum biter bitmez uygulama üst çubuğu otomatik olarak **yeşile ("Bağlı ve Hazır")** döner!

---

## Yeniden Derleme Komutları

```powershell
# Windows 64-bit tek dosya (.exe) derleme:
python packaging/build_windows.py

# Linux x86_64 AppImage derleme (Linux üzerinde):
bash packaging/build_linux_appimage.sh
```

---

## Komut Satırı (CLI) Kullanımı

```powershell
# Bağlantı ve sürücü durumunu kontrol et
python cli_test.py --status

# Firmware yükle ve GT-6801 yonga kimliğini doğrula
python cli_test.py --test

# Tarama kafasını başlangıç (Home) noktasına çek
python cli_test.py --home

# 300 DPI renkli tarama yapıp dosyaya kaydet
python cli_test.py --scan cikti.png --dpi 300 --mode color
```
