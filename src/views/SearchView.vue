<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'

// ref() digunakan untuk menyimpan data yang diketik pengguna
const scanInput = ref('')
// useRouter() digunakan untuk pindah halaman
const router = useRouter()

function searchAsset() {
  if (!scanInput.value) {
    alert('Silakan masukkan hasil pindaian QR')
    return
  }

  // Ini adalah logika untuk mem-parsing input Anda "e-Leasing:610000010479"
  let assetId = ''
  if (scanInput.value.includes(':')) {
    // Ambil bagian setelah "e-Leasing:"
    assetId = scanInput.value.split(':')[1].trim()
  } else {
    // Jika pengguna hanya mengetik/paste nomornya saja
    assetId = scanInput.value.trim()
  }

  if (assetId) {
    // Pindah ke halaman detail, sambil membawa 'assetId' di URL
    router.push({ name: 'asset-detail', params: { id: assetId } })
  } else {
    alert('Format input tidak valid.')
  }
}
</script>

<template>
  <div class="search-container">
    <h1>Fixed Asset Checker</h1>
    <p>Silakan pindai (scan) QR code pada nameplate aset dan tempel (paste) hasilnya di sini.</p>

    <form @submit.prevent="searchAsset">
      <input
        v-model="scanInput"
        type="text"
        placeholder="Contoh: e-Leasing:610000010479"
        class="search-input"
      />
      <button type="submit" class="search-button">Cari Aset</button>
    </form>
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
  max-width: 150px; /* Atur ukuran logo Anda */
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
.search-button {
  width: 100%;
  padding: 12px 20px;
  font-size: 18px;
  font-weight: bold;
  cursor: pointer;
  background-color: #004a99; /* Ganti dengan warna perusahaan Anda */
  color: white;
  border: none;
  border-radius: 8px;
}
.search-button:hover {
  background-color: #003366;
}
</style>
