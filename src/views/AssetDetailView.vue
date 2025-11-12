<script setup>
import { ref, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import Papa from 'papaparse'

// ------------------------------------------------------------------
// !!! PENTING: GANTI DENGAN URL GOOGLE SHEET .CSV ANDA !!!
// ------------------------------------------------------------------
const GOOGLE_SHEET_URL =
  'https://docs.google.com/spreadsheets/d/e/2PACX-1vTizT9NlMSSn77pWOCRVUoorr0GByWdrpmRCMMTZpRY89-Z4fpDu9z9b8FzQe5ruGu2Im88VLOkRjRP/pub?gid=0&single=true&output=csv'
// ------------------------------------------------------------------

const route = useRoute() // Untuk membaca parameter dari URL
const router = useRouter() // Untuk tombol "Kembali"

const assetId = ref(route.params.id) // Ambil '610000010479' dari URL
const asset = ref(null) // Untuk menyimpan data aset yang ditemukan
const loading = ref(true)
const error = ref(null)

// onMounted() adalah fungsi yang berjalan otomatis saat halaman dibuka
onMounted(() => {
  if (!GOOGLE_SHEET_URL.includes('http')) {
    error.value = 'URL Google Sheet belum diatur. Silakan cek file AssetDetailView.vue'
    loading.value = false
    return
  }

  // Gunakan PapaParse untuk mengambil dan membaca data CSV dari URL
  Papa.parse(GOOGLE_SHEET_URL, {
    download: true, // Download file dari URL
    header: true, // Gunakan Baris 1 sebagai 'key' (penting!)
    complete: (results) => {
      // 'results.data' adalah array dari semua data aset Anda
      const allAssets = results.data

      // Cari aset yang 'Asset Number'-nya sama dengan 'assetId' dari URL
      // Pastikan nama kolom 'Asset Number' di Google Sheet Anda sama persis
      asset.value = allAssets.find((row) => row['Asset Number'] === assetId.value)

      loading.value = false // Selesai loading
    },
    error: (err) => {
      error.value = 'Gagal memuat data dari Google Sheet. Cek koneksi atau URL.'
      loading.value = false
    },
  })
})

function backToSearch() {
  router.push('/') // Kembali ke halaman utama
}
</script>

<template>
  <div class="detail-container">
    <button @click="backToSearch" class="back-button">&laquo; Kembali ke Pencarian</button>

    <div v-if="loading" class="status-box">
      <h2>Mencari data aset...</h2>
    </div>

    <div v-else-if="asset" class="asset-info">
      <h1>Detail Aset</h1>
      <div class="info-item">
        <strong>Nomor Aset:</strong>
        <span>{{ asset['Asset Number'] }}</span>
      </div>
      <div class="info-item">
        <strong>Deskripsi Aset:</strong>
        <span>{{ asset['Asset Description'] }}</span>
      </div>
      <div class="info-item">
        <strong>Lokasi:</strong>
        <span>{{ asset['Location'] }}</span>
      </div>
      <div class="info-item">
        <strong>Serial No. / Parts No.:</strong>
        <span>{{ asset['Serial No. / Parts No.'] }}</span>
      </div>
      <div class="info-item">
        <strong>Tanggal Akuisisi:</strong>
        <span>{{ asset['Acquisition Date'] }}</span>
      </div>
      <div class="info-item">
        <strong>Kondisi:</strong>
        <span>{{ asset['Condition'] }}</span>
      </div>
    </div>

    <div v-else-if="!loading && !asset" class="status-box error">
      <h1>Aset Tidak Ditemukan</h1>
      <p>
        Aset dengan nomor <strong>{{ assetId }}</strong> tidak dapat ditemukan di database.
      </p>
    </div>

    <div v-else-if="error" class="status-box error">
      <h1>Terjadi Kesalahan</h1>
      <p>{{ error }}</p>
    </div>
  </div>
</template>

<style scoped>
.detail-container {
  max-width: 800px;
  margin: 40px auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}
.asset-info {
  display: flex;
  flex-direction: column;
  gap: 16px; /* Jarak antar item */
}
.info-item {
  display: flex;
  flex-direction: column;
  padding: 10px;
  border-bottom: 1px solid #eee;
}
.info-item strong {
  font-size: 14px;
  color: #555;
  margin-bottom: 5px;
  text-transform: uppercase;
}
.info-item span {
  font-size: 18px;
  color: #000;
}
.back-button {
  margin-bottom: 25px;
  padding: 8px 15px;
  font-size: 14px;
  cursor: pointer;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 5px;
}
.status-box {
  text-align: center;
  padding: 40px;
}
.error {
  color: #d8000c;
  background-color: #ffd2d2;
  border: 1px solid #d8000c;
  border-radius: 8px;
}
</style>
