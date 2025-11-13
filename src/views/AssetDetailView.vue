<script setup>
import { ref, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import Papa from 'papaparse'

// ------------------------------------------------------------------
// 1. URL BACA DATA (READ-ONLY) - URL .csv Anda
// ------------------------------------------------------------------
const GOOGLE_SHEET_READ_URL =
  'https://docs.google.com/spreadsheets/d/e/2PACX-1vTizT9NlMSSn77pWOCRVUoorr0GByWdrpmRCMMTZpRY89-Z4fpDu9z9b8FzQe5ruGu2Im88VLOkRjRP/pub?gid=0&single=true&output=csv'
// ------------------------------------------------------------------

// ------------------------------------------------------------------
// 2. URL "WRITE API" ANDA - DARI LOG ERROR ANDA
// ------------------------------------------------------------------
const GOOGLE_SCRIPT_WRITE_URL =
  'https://script.google.com/macros/s/AKfycbxHjhPMK20NIJP-g7F5d2aQb0szHTNR4gJGd-_nJZHUpzlGt_r5xISFfO7gbkMIEjPN/exec'
// ------------------------------------------------------------------

const route = useRoute()
const router = useRouter()

const assetId = ref(route.params.id)
const asset = ref(null) // Data dari PapaParse (Read-Only)
const loading = ref(true)
const error = ref(null)

// Variabel untuk menampung data inputan
const inputPartNumber = ref('')
const inputModel = ref('')
const inputAssetName = ref('')

// Variabel status untuk proses submit
const isSubmitting = ref(false)
const submitMessage = ref(null)

// Fungsi yang berjalan saat halaman dibuka
onMounted(() => {
  Papa.parse(GOOGLE_SHEET_READ_URL, {
    download: true,
    header: true,
    complete: (results) => {
      const data = results.data
      asset.value = data.find(
        (row) => row['Asset Number'] && row['Asset Number'].toString().trim() === assetId.value,
      )
      if (asset.value) {
        inputPartNumber.value = asset.value['Serial No. / Parts No.'] || ''
        inputAssetName.value = asset.value['Asset Name'] || ''
        inputModel.value = asset.value.Model || ''
      }
      loading.value = false
    },
    error: (err) => {
      error.value = 'Gagal memuat data dari Google Sheet: ' + err.message
      loading.value = false
    },
  })
})

// ==================================================================
// FUNGSI submitData() YANG SUDAH DIPERBAIKI
// ==================================================================
async function submitData() {
  if (!inputPartNumber.value || !inputModel.value || !inputAssetName.value) {
    submitMessage.value = {
      type: 'error',
      text: 'Semua data (Part No, Asset Name, Model) wajib diisi.',
    }
    return
  }

  isSubmitting.value = true
  submitMessage.value = null

  try {
    const response = await fetch(GOOGLE_SCRIPT_WRITE_URL, {
      method: 'POST',
      // 1. KEMBALIKAN KE 'cors' AGAR BISA BACA RESPON
      mode: 'cors',

      headers: {
        // 2. GANTI 'application/json' MENJADI 'text/plain'
        // Ini adalah FIX UTAMA untuk menghindari redirect Google
        'Content-Type': 'text/plain',
      },

      // 3. Body tetap dikirim sebagai string JSON
      body: JSON.stringify({
        assetId: assetId.value,
        partNumber: inputPartNumber.value,
        model: inputModel.value,
        assetName: inputAssetName.value,
      }),
    })

    // 4. SEKARANG KITA BISA BACA RESPONNYA DENGAN AMAN
    const result = await response.json()

    if (result.status === 'success') {
      submitMessage.value = { type: 'success', text: result.message }
    } else {
      submitMessage.value = { type: 'error', text: 'Error dari API: ' + result.message }
    }
  } catch (err) {
    // 5. Kembalikan ke error handling normal
    submitMessage.value = { type: 'error', text: 'Gagal menghubungi server API: ' + err.message }
  } finally {
    isSubmitting.value = false
  }
}
// ==================================================================

function backToSearch() {
  router.push('/')
}
</script>

<template>
  <div class="detail-container">
    <button @click="backToSearch" class="back-button">&laquo; Kembali ke Scan</button>

    <!-- Tampilan saat Loading -->
    <div v-if="loading" class="status-box">
      <h2>Mencari data aset...</h2>
    </div>

    <!-- Tampilan jika Aset DITEMUKAN -->
    <div v-else-if="asset" class="asset-wrapper">
      <!-- Bagian 1: Menampilkan data (Read-Only) -->
      <div class="asset-info">
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
      </div>

      <hr class="divider" />

      <!-- Bagian 2: Form untuk INPUT data baru -->
      <div class="asset-form">
        <h2>Input / Update Data</h2>

        <form @submit.prevent="submitData">
          <div class="form-group">
            <label for="partNumber">Part Number (Serial No.):</label>
            <input
              id="partNumber"
              v-model="inputPartNumber"
              type="text"
              placeholder="Masukkan part number"
            />
          </div>

          <div class="form-group">
            <label for="assetName">Asset Name:</label>
            <input
              id="assetName"
              v-model="inputAssetName"
              type="text"
              placeholder="Masukkan nama aset"
            />
          </div>

          <div class="form-group">
            <label for="model">Model:</label>
            <input id="model" v-model="inputModel" type="text" placeholder="Masukkan model" />
          </div>

          <!-- Perbaikan typo type.submit" menjadi type="submit" -->
          <button type="submit" class="submit-button" :disabled="isSubmitting">
            {{ isSubmitting ? 'Menyimpan...' : 'Simpan Data' }}
          </button>
        </form>

        <div v-if="submitMessage" :class="['submit-status', submitMessage.type]">
          {{ submitMessage.text }}
        </div>
      </div>
    </div>

    <!-- Perbaikan urutan v-else-if -->
    <!-- Tampilan jika ada Error (misal: Gagal load / API error) -->
    <div v-else-if="error" class="status-box error">
      <h1>Terjadi Kesalahan</h1>
      <p>{{ error }}</p>
    </div>

    <!-- Tampilan jika Aset TIDAK DITEMUKAN -->
    <div v-else class="status-box error">
      <h1>Aset Tidak Ditemukan</h1>
      <p>
        Aset dengan nomor <strong>{{ assetId }}</strong> tidak dapat ditemukan.
      </p>
    </div>
  </div>
</template>

<style scoped>
/* Style tidak berubah */
.detail-container {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}
.asset-info {
  display: flex;
  flex-direction: column;
  gap: 16px;
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
.divider {
  border: 0;
  border-top: 2px solid #eee;
  margin: 30px 0;
}
.asset-form h2 {
  border-bottom: 1px solid #ccc;
  padding-bottom: 10px;
}
.form-group {
  margin-bottom: 20px;
}
.form-group label {
  display: block;
  font-weight: bold;
  margin-bottom: 8px;
}
.form-group input {
  width: 95%;
  padding: 12px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 5px;
}
.submit-button {
  width: 100%;
  padding: 12px 20px;
  font-size: 18px;
  font-weight: bold;
  cursor: pointer;
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 8px;
}
.submit-button:disabled {
  background-color: #aaa;
}
.submit-status {
  margin-top: 15px;
  padding: 10px;
  border-radius: 5px;
  text-align: center;
}
.submit-status.success {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}
.submit-status.error {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}
</style>
