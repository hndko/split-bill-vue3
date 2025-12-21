# 💰 Split Bill - Hitung Patungan dengan Mudah

Aplikasi web untuk menghitung dan membagi tagihan/bill secara adil antar peserta. Dibuat dengan Vue 3 + Vite.

![Dark Mode Theme](https://img.shields.io/badge/Theme-Dark%20Mode-1a1a1a)
![Vue 3](https://img.shields.io/badge/Vue-3-42b883)
![Vite](https://img.shields.io/badge/Vite-5-646cff)

## ✨ Fitur

- 📝 **Tambah Item Menu** - Input nama, harga (format Indonesia: 25.000), dan jumlah
- 👥 **Kelola Peserta** - Tambah/hapus peserta yang ikut patungan
- ✅ **Assign Menu ke Peserta** - Pilih siapa yang memesan menu apa
- 💵 **Hitung Pajak & Service** - Support persentase atau nominal fixed
- 📊 **Ringkasan Pembagian** - Lihat detail tagihan per orang
- 📥 **Download Gambar** - Simpan ringkasan sebagai gambar PNG

## 🚀 Cara Menjalankan

### Prerequisites

- Node.js 18+
- npm atau yarn

### Instalasi

```bash
# Clone repository
git clone https://github.com/username/split-bill-react.git
cd split-bill-react

# Install dependencies
npm install

# Jalankan development server
npm run dev
```

Buka [http://localhost:5173](http://localhost:5173) di browser.

### Build Production

```bash
npm run build
```

## 🎨 Teknologi

- **Vue 3** - Framework JavaScript dengan Composition API
- **Vite** - Build tool yang cepat
- **html2canvas** - Untuk export ringkasan sebagai gambar
- **CSS Variables** - Dark mode theme yang konsisten

## 📱 Tampilan

Aplikasi menggunakan dark mode theme dengan warna:

- Background: `#0f0f0f`
- Card: `#1a1a1a`
- Surface: `#1e1e1e`
- Primary: `#06b6d4` (Cyan)
- Accent: `#39ff14` (Neon Green)

## 📄 Lisensi

MIT License - Bebas digunakan untuk keperluan pribadi maupun komersial.
