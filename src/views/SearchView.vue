<script setup>
import { ref, watch } from 'vue' // <-- 1. Impor 'watch'
import { useRouter } from 'vue-router'

const scanInput = ref('')
const router = useRouter()

// Fungsi ini sekarang akan dipanggil oleh 'watch'
function performSearch(scanValue) {
  if (!scanValue) return // Keluar jika kosong

  let assetId = ''
  if (scanValue.includes(':')) {
    // Mem-parsing "e-Leasing:610000010479"
    assetId = scanValue.split(':')[1].trim()
  } else {
    // Jika hanya nomornya saja
    assetId = scanValue.trim()
  }

  // 2. Tambahkan pengecekan panjang ID
  // Aset nomor Anda panjangnya 12 digit (610000000675)
  // Ini mencegah pencarian prematur jika ada yang mengetik manual
  if (assetId && assetId.length >= 12) {
    // 3. Langsung pindah halaman (eksekusi pencarian)
    router.push({ name: 'asset-detail', params: { id: assetId } })

    // 4. Kosongkan input agar siap untuk scan berikutnya
    scanInput.value = ''
  }
}

// 5. Ini adalah inti perubahannya
// 'watch' akan mengawasi variabel scanInput
// Begitu scanInput berubah, fungsi performSearch akan dipanggil
watch(scanInput, (newScanValue) => {
  performSearch(newScanValue)
})
</script>

<template>
  <div class="search-container">
    <img src="/logo.png" alt="Company Logo" class="logo" />
    <h1>Fixed Asset Checker</h1>
    <p>Arahkan scan Anda ke kotak di bawah ini. Pencarian akan berjalan otomatis.</p>

    <input
      v-model="scanInput"
      type="text"
      placeholder="Scan atau paste di sini..."
      class="search-input"
      autofocus
    />
  </div>
</template>

<style scoped>
.search-container {
  max-width: 600px;
  margin: 40px auto;
  padding: 20px;
  text-align: center;
  font-family: Arial, sans-serif;
}
.logo {
  max-width: 150px;
  margin-bottom: 20px;
}
.search-input {
  width: 95%;
  padding: 12px;
  font-size: 16px;
  margin: 20px 0;
  border: 2px solid #ccc;
  border-radius: 8px;
}
/* 8. Kita sudah bisa hapus style untuk .search-button */
</style>
