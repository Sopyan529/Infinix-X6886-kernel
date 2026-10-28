# Custom GKI Linux Kernel 5.10 for Infinix X6886 (Helio G99 / MT6789) - Droidspaces Ready

[![Android Version](https://img.shields.io/badge/Android-15%20(API%2035)-green.svg)](https://android.com)
[![Kernel Version](https://img.shields.io/badge/Kernel-5.10.226%20GKI-blue.svg)](https://kernel.org)
[![Compiler](https://img.shields.io/badge/Clang-12.0.5%20(r416183b)-orange.svg)](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86)
[![Droidspaces](https://img.shields.io/badge/Droidspaces-100%25%20Supported-brightgreen.svg)](https://github.com/ravindu644/Droidspaces-OSS)
[![Root](https://img.shields.io/badge/Root-KernelSU%20Ready-red.svg)](https://kernelsu.org)

Source code kernel kustom **Generic Kernel Image (GKI) Linux 5.10** yang telah dioptimasi khusus untuk **Infinix X6886 (Infinix Zero 30 4G / Helio G99 / MediaTek MT6789)** yang berjalan pada **Android 15**.

Kernel ini dilengkapi dengan seluruh dependensi wajib (*MUST HAVE: 100% Passed*) untuk menjalankan aplikasi container / virtualisasi ruang kedua **[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)** tanpa menyebabkan bootloop atau bentrok dengan driver hardware MediaTek berkat implementasi **kABI Safe Padding (Slot 3, 4, 5)**.

---

## 📱 Spesifikasi Perangkat Target

| Parameter | Spesifikasi |
| :--- | :--- |
| **Model Perangkat** | Infinix X6886 |
| **Pabrikan (OEM)** | Infinix / Transsion Holdings |
| **SoC / Chipset** | MediaTek Helio G99 (`MT6789`) |
| **Arsitektur CPU** | ARM64 (`aarch64` / ARMv8.2-A) |
| **Versi Android** | Android 15 (SDK 35) |
| **Build OS Bawaan** | `X6886-15.1.2.129SP03(BASE001PF001AZ)` |
| **Kernel Baseline** | Linux 5.10.226-android12-9-00047-g4968e29b7f92-ab12786767 |
| **AVB Rollback Index** | **5** (Proteksi ARP Hardware Bypass) |

---

## ✨ Fitur Utama & Modifikasi Kernel

### 1. Dukungan Penuh Droidspaces & Linux Containers
- ✅ **`CONFIG_SYSVIPC=y`** & **`CONFIG_SYSVIPC_SYSCTL=y`**: Inter-Process Communication untuk sinkronisasi thread di dalam container.
- ✅ **`CONFIG_IPC_NS=y`**: Isolasi IPC Namespace.
- ✅ **`CONFIG_PID_NS=y`**: Isolasi Process ID Namespace.
- ✅ **`CONFIG_OVERLAY_FS=y`**: Filesystem layering untuk sandbox Droidspaces.
- ✅ **`CONFIG_NAMESPACES=y`**, **`CONFIG_UTS_NS=y`**, **`CONFIG_NET_NS=y`**: Isolasi jaringan, hostname, dan filesystem mount.
- ✅ **`CONFIG_BINFMT_MISC=y`**: Menjalankan binary multi-arsitektur di userspace.

### 2. Perlindungan kABI (Kernel Module Interface)
- Menggunakan patch padding kABI resmi Droidspaces pada `include/linux/sched.h` menggunakan **Slot 3, 4, dan 5** (`_ANDROID_KABI_REPLACE`).
- Mencegah perubahan ukuran `struct task_struct`, sehingga driver bawaan vendor MediaTek (GPU Mali, Touchscreen, Kamera, Modem) **dijamin tetap cocok (CRC match) dan tidak mengalami bootloop / reboot berulang**.
- **SELinux Safe**: Tidak mengaktifkan `CONFIG_USER_NS` statis yang dilarang (*Neverallow*) oleh SELinux Policy Android 15.

---

## 🛠️ Persyaratan Lingkungan Build (Host OS)

Pastikan komputer/laptop Anda menggunakan sistem operasi Linux (Ubuntu 20.04/22.04 LTS, Debian, atau Arch Linux).

### 1. Install Paket Dependencies
```bash
sudo apt update
sudo apt install -y bc bison build-essential ccache curl flex g++-multilib \
gcc-multilib git gnupg gperf libelf-dev libncurses5-dev libssl-dev \
libxml2-utils lz4 lzop make python3 rsync unzip zip zlib1g-dev
```

### 2. Download Toolchain Clang Resmi AOSP (Clang 12.0.5 `r416183b`)
Kernel ini wajib dikompilasi menggunakan compiler resmi yang identik dengan OEM Infinix:
```bash
mkdir -p toolchain/clang-r416183b
cd toolchain/clang-r416183b
curl -L "https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android12-release/clang-r416183b.tar.gz" | tar -xz
cd ../..
```

---

## 🔨 Panduan Kompilasi (Build Step-by-Step)

### 1. Clone Repositori Ini
```bash
git clone https://github.com/Sopyan529/Infinix-X6886-kernel.git -b main kernel
cd kernel
```

### 2. Export Environment Variables
```bash
export WORKSPACE_DIR="$(pwd)"
export TOOLCHAIN_DIR="$WORKSPACE_DIR/../toolchain/clang-r416183b"
export PATH="$TOOLCHAIN_DIR/bin:$PATH"

export ARCH=arm64
export SUBARCH=arm64
export LLVM=1
export LLVM_IAS=1
export CLANG_TRIPLE=aarch64-linux-gnu-
export CROSS_COMPILE=aarch64-linux-gnu-
```

### 3. Konfigurasi Defconfig
Gunakan konfigurasi Droidspaces v3:
```bash
make O=out droidspace_v3_defconfig
```

### 4. Mulai Kompilasi Kernel
Gunakan semua core CPU Anda untuk mempercepat proses linking LTO:
```bash
make -j$(nproc --all) O=out Image.gz
```
Hasil biner terkompresi akan berada di: `out/arch/arm64/boot/Image.gz`.

---

## 📦 Panduan Repack & Sign AVB (Anti-Rollback Protection Safe)

> [!IMPORTANT]
> Bootloader MediaTek Infinix X6886 memeriksa tanda tangan AVB 2.0 dan nilai Rollback Index. Jika boot image di-flash tanpa tanda tangan yang valid, perangkat akan langsung mati di logo Infinix atau memicu ARP trigger!

### 1. Repack Image
Satukan `Image.gz` baru dengan header dan generic ramdisk Android v4:
```bash
# Pastikan Anda memiliki ramdisk.img bawaan boot.img Infinix X6886
python3 -c "
import struct
def align_4k(n): return (n + 4095) & (~4095)

with open('out/arch/arm64/boot/Image.gz', 'rb') as f: k_data = f.read()
with open('ramdisk.img', 'rb') as f: r_data = f.read()
with open('boot_stock_header.bin', 'rb') as f: orig_hdr = bytearray(f.read(4096))

k_sz = len(k_data)
r_sz = len(r_data)
struct.pack_into('<I', orig_hdr, 8, k_sz)
struct.pack_into('<I', orig_hdr, 12, r_sz)

with open('boot_unsigned.img', 'wb') as f:
    f.write(orig_hdr)
    f.write(k_data)
    f.write(b'\x00' * (align_4k(k_sz) - k_sz))
    f.write(r_data)
    f.write(b'\x00' * (align_4k(r_sz) - r_sz))
"
```

### 2. Sign AVB Footer dengan Rollback Index = 5
Gunakan `avbtool`:
```bash
./avbtool.py add_hash_footer \
    --image boot_unsigned.img \
    --partition_size 67108864 \
    --partition_name boot \
    --algorithm SHA256_RSA2048 \
    --key testkey_rsa2048.pem \
    --rollback_index 5 \
    --rollback_index_location 3 \
    --prop com.android.build.boot.fingerprint:Infinix/X6886-OP/Infinix-X6886:12/SP1A.210812.016/251014V1355:user/release-keys \
    --prop com.android.build.boot.os_version:12 \
    --prop com.android.build.boot.security_patch:2025-02-05

mv boot_unsigned.img boot_signed_droidspace.img
```

---

## ⚡ Panduan Flashing ke Ponsel

### Metode 1: Via Fastboot Mode
1. Masuk ke mode Fastboot:
   ```bash
   adb reboot bootloader
   ```
2. Pastikan perangkat terdeteksi:
   ```bash
   fastboot devices
   ```
3. Flash ke partisi `boot`:
   ```bash
   fastboot flash boot boot_signed_droidspace.img
   fastboot reboot
   ```

### Metode 2: Via OrangeFox / TWRP Recovery (AnyKernel3)
1. Salin file zip installer `Infinix_X6886_Kernel_5.10_v2.zip` ke memori HP.
2. Masuk ke OrangeFox Recovery.
3. Pilih file zip -> **Swipe to Flash**.
4. Reboot System.

---

## 🚨 Troubleshooting & Penyelamatan Bootloop (BROM Mode)

Jika perangkat mengalami kendala saat modifikasi kernel, Anda bisa memulihkan partisi `boot` secara instan menggunakan exploit **Antumbra (HeapBait)** tanpa menyalakan layar:

```bash
# Flash kembali boot asli via USB BROM
antumbra -d DA_BR_MT6789_transsion.bin write boot_a dumps/boot.img
antumbra -d DA_BR_MT6789_transsion.bin reboot
```
*Tahan tombol Volume Atas (+) & Volume Bawah (-) lalu colokkan kabel USB saat HP mati untuk masuk ke BROM mode.*

---

## 📜 Lisensi & Kredit

- **Linux Kernel Source**: GNU General Public License v2 (GPL-2.0)
- **kABI Patches**: [Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS) oleh ravindu644 & kontributor
- **Pengembang Kernel**: [Sopyan529](https://github.com/Sopyan529)
