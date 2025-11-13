<script setup>
import { ref, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
// KEMBALIKAN PAPAPARSE
import Papa from 'papaparse'

// ------------------------------------------------------------------
// 1. URL BACA DATA (CSV) - PASTI BENAR (JANGAN DIUBAH LAGI)
// ------------------------------------------------------------------
const GOOGLE_SHEET_READ_URL =
  'https://docs.google.com/spreadsheets/d/e/2PACX-1vTizT9NlMSSn77pWOCRVUoorr0GByWdrpmRCMMTZpRY89-Z4fpDu9z9b8FzQe5ruGu2Im88VLOkRjRP/pub?gid=1100163174&single=true&output=csv'
// ------------------------------------------------------------------

// ------------------------------------------------------------------
// 2. URL "WRITE API" ANDA - PASTI BENAR (JANGAN DIUBAH LAGI)
// ------------------------------------------------------------------
const GOOGLE_SCRIPT_WRITE_URL =
  'https://script.google.com/macros/s/AKfycbyQuFcOi27fJ-0V1HSXpGzuNDmcpU1wfG5gPpg1SudloTzUkNtw06HWRo6-e6lWnLZFbw/exec'
// ------------------------------------------------------------------

const route = useRoute()
const router = useRouter()

const assetId = ref(route.params.id)
const asset = ref(null) // Data dari PapaParse
const loading = ref(true)
const error = ref(null)

// --- Variabel untuk Alur Logika ---
const isEditing = ref(false)
const isNewEntry = ref(false)
// ---------------------------------

// --- Variabel Input Baru ---
const inputPartNumber = ref('')
const inputModel = ref('')
const inputPartName = ref('')
// -------------------------

// Variabel status untuk proses submit
const isSubmitting = ref(false)
const submitMessage = ref(null)

// ==================================================================
// FUNGSI onMounted (MENGGUNAKAN PAPAPARSE)
// ==================================================================
onMounted(() => {
  loading.value = true
  error.value = null

  Papa.parse(GOOGLE_SHEET_READ_URL, {
    download: true,
    header: true,
    skipEmptyLines: true,
    cache: false,
    complete: (results) => {
      const data = results.data

      asset.value = data.find(
        (row) => row['ASSET NO'] && row['ASSET NO'].toString().trim() === assetId.value,
      )

      if (asset.value) {
        // --- Mengambil data dari kolom yang benar ---
        inputPartNumber.value = asset.value['PART NO'] || '' // Kolom G
        // FIX DUPLIKAT: Ambil 'PART NAME' (jika ada) ATAU 'ASSET NAME_1'
        inputPartName.value = asset.value['PART NAME'] || asset.value['ASSET NAME_1'] || '' // Kolom E
        inputModel.value = asset.value['MODEL'] || '' // Kolom F

        // --- LOGIKA TAMPILAN BARU ---
        // Tampilkan form HANYA jika MODEL atau PART NO masih kosong
        if (!asset.value['MODEL'] || !asset.value['PART NO']) {
          isNewEntry.value = true
          isEditing.value = true
        } else {
          isNewEntry.value = false
          isEditing.value = false
        }
        // ------------------------------------
      } else {
        error.value = `Aset dengan nomor ${assetId.value} tidak ditemukan.`
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

// ==================================================================
// FUNGSI SUBMITDATA (POST) - KEMBALI KE "FIRE AND FORGET"
// ==================================================================
function submitData() {
  // Hapus 'async'
  // Validasi baru
  if (!inputPartNumber.value || !inputModel.value || !inputPartName.value) {
    submitMessage.value = {
      type: 'error',
      text: 'Semua data (Model, Part No, Part Name) wajib diisi.',
    }
    return
  }

  isSubmitting.value = true
  submitMessage.value = null // Hapus pesan error lama

  // "Fire and Forget" - Kirim data tapi jangan tunggu respons
  fetch(GOOGLE_SCRIPT_WRITE_URL, {
    method: 'POST',
    mode: 'cors',
    headers: {
      'Content-Type': 'text/plain',
    },
    // --- PAYLOAD BARU ---
    body: JSON.stringify({
      assetId: assetId.value,
      partNumber: inputPartNumber.value,
      model: inputModel.value,
      partName: inputPartName.value, // Menggunakan partName
    }),
  }).catch((err) => {
    // Kita "menelan" error-nya (Abaikan).
    // Kita tahu ini akan 'Failed to fetch' (atau error lain), jadi kita abaikan.
    console.warn('Ignoring expected fetch error (this is intentional):', err.message)
  })

  // JANGAN TUNGGU (await) fetch-nya.

  // Langsung set status submitting ke false
  isSubmitting.value = false

  // Langsung kembali ke halaman scanner
  // (Pesan 'Berhasil' akan tersirat karena langsung kembali)
  confirmAndGoBack()
}
// ==================================================================

// --- FUNGSI-FUNGSI BANTU UNTUK TOMBOL ---

function startEditMode() {
  isEditing.value = true
}

function cancelEditMode() {
  // Kembalikan data ke nilai semula (sebelum diedit)
  inputPartNumber.value = asset.value['PART NO'] || ''
  inputPartName.value = asset.value['PART NAME'] || asset.value['ASSET NAME_1'] || ''
  inputModel.value = asset.value['MODEL'] || ''

  isEditing.value = false
}

function confirmAndGoBack() {
  router.push('/') // Kembali ke halaman scanner (SearchView)
}
</script>

<template>
  <div class="detail-container">
    <button @click="confirmAndGoBack" class="back-button">&laquo; Kembali ke Scan</button>

    <!-- Tampilan saat Loading -->
    <div v-if="loading" class="status-box">
      <h2>Mencari data aset...</h2>
    </div>

    <!-- Tampilan jika Aset DITEMUKAN -->
    <div v-else-if="asset" class="asset-wrapper">
      <div class="asset-info">
        <h1>Detail Aset</h1>
        <!-- ============================================================ -->
        <!-- URUTAN TAMPILAN SESUAI PERMINTAAN ANDA (READ-ONLY) -->
        <!-- ============================================================ -->
        <div class="info-item">
          <strong>ASSET NO:</strong>
          <span>{{ asset['ASSET NO'] }}</span>
          <!-- 1. Asset No (Kolom B) -->
        </div>
        <div class="info-item">
          <strong>ASSET NAME (Deskripsi):</strong>
          <!-- FIX DUPLIKAT HEADER: 'ASSET NAME' adalah Kolom C -->
          <span>{{ asset['ASSET NAME'] }}</span>
          <!-- 2. Asset Name (Kolom C) -->
        </div>
        <div class="info-item">
          <strong>LOCATION:</strong>
          <span>{{ asset['LOCATION'] }}</span>
          <!-- (Data tambahan, Kolom I) -->
        </div>

        <!-- 
          ============================================================
          BAGIAN 2: LOGIKA TAMPILAN (Menyatu dengan di atas)
          ============================================================
        -->

        <!-- TAMPILAN BACA (READ-ONLY) -->
        <div v-if="!isEditing" class="read-only-view">
          <div class="info-item">
            <strong>MODEL:</strong>
            <span>{{ inputModel || '-' }}</span>
            <!-- 3. Model (Kolom F) -->
          </div>
          <div class="info-item">
            <strong>PART NO:</strong>
            <span>{{ inputPartNumber || '-' }}</span>
            <!-- 4. Part No (Kolom G) -->
          </div>
          <div class="info-item">
            <strong>PART NAME:</strong>
            <span>{{ inputPartName || '-' }}</span>
            <!-- 5. Part Name (Kolom E) -->
          </div>
          <div class="info-item">
            <strong>ACQ. DATE:</strong>
            <span>{{ asset['ACQ. DATE'] }}</span>
            <!-- 6. Acq. Date (Kolom H) -->
          </div>

          <!-- Tombol Aksi untuk mode baca -->
          <div class="button-group">
            <button @click="startEditMode" class="edit-button">Edit</button>
            <button @click="confirmAndGoBack" class="confirm-button">
              Konfirmasi & Lanjut Scan
            </button>
          </div>
        </div>

        <!-- TAMPILAN FORM (EDIT / INPUT BARU) -->
        <div v-else class="asset-form">
          <h2 v-if="isNewEntry" class="form-title">Input Data Baru</h2>
          <h2 v-else class="form-title">Update Data Part</h2>

          <form @submit.prevent="submitData">
            <!-- ============================================================ -->
            <!-- URUTAN TAMPILAN SESUAI PERMINTAAN ANDA (FORM EDIT) -->
            <!-- ============================================================ -->
            <div class="form-group">
              <label for="model">MODEL:</label>
              <!-- 3. Model (Kolom F) -->
              <input id="model" v-model="inputModel" type="text" placeholder="Masukkan model" />
            </div>

            <div class="form-group">
              <label for="partNumber">PART NO:</label>
              <!-- 4. Part No (Kolom G) -->
              <input
                id="partNumber"
                v-model="inputPartNumber"
                type="text"
                placeholder="Masukkan part number"
              />
            </div>

            <div class="form-group">
              <label for="partName">PART NAME:</label>
              <!-- 5. Part Name (Kolom E) -->
              <input
                id="partName"
                v-model="inputPartName"
                type="text"
                placeholder="Masukkan Part Name"
              />
            </div>

            <!-- Data read-only tetap ditampilkan di bawah form -->
            <div class="info-item readonly-in-form">
              <strong>ACQ. DATE:</strong>
              <span>{{ asset['ACQ. DATE'] }}</span>
              <!-- 6. Acq. Date (Kolom H) -->
            </div>

            <!-- Tombol Aksi untuk mode edit -->
            <div class="button-group">
              <button type="submit" class="submit-button" :disabled="isSubmitting">
                {{
                  isSubmitting ? 'Menyimpan...' : isNewEntry ? 'Submit Data' : 'Simpan Perubahan'
                }}
              </button>

              <button
                v-if="!isNewEntry"
                @click="cancelEditMode"
                type="button"
                class="cancel-button"
              >
                Batal
              </button>
            </div>
          </form>

          <!-- Status message (sukses/error) -->
          <!-- 
            CATATAN: Pesan error di bawah ini hanya akan tampil jika validasi gagal.
            Error 'Failed to fetch' akan diabaikan.
          -->
          <div v-if="submitMessage" :class="['submit-status', submitMessage.type]">
            {{ submitMessage.text }}
          </div>
        </div>
      </div>
      <!-- Penutup .asset-info -->
    </div>
    <!-- Penutup v-else-if="asset" -->

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
/* ============================================================
  CSS LENGKAP (TIDAK BERUBAH DARI SEBELUMNYA)
  ============================================================
*/
.detail-container {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  background-color: #fafafa; /* Latar belakang sedikit abu-abu */
}
.asset-info {
  display: flex;
  flex-direction: column;
  gap: 10px; /* Jarak antar item dikurangi */
}
.asset-info h1 {
  /* Style untuk H1 */
  color: #333;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
  margin-top: 0;
}
.info-item {
  display: flex;
  flex-direction: column;
  padding: 8px; /* Padding dikurangi */
  border-bottom: 1px solid #eee;
}
.info-item strong {
  font-size: 14px;
  color: #555;
  margin-bottom: 4px; /* Jarak label ke data dikurangi */
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
/* ============================================================
  STYLE UNTUK FORM (SUDAH DISESUAIKAN)
  ============================================================
*/
.asset-form {
  padding-top: 10px; /* Beri sedikit jarak di atas form */
}
.form-title {
  /* Menggantikan .asset-form h2 */
  border-bottom: 1px solid #ccc;
  padding-bottom: 10px;
  color: #333;
  margin-top: 0;
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
  width: 95%; /* Diberi 95% agar tidak mepet */
  padding: 12px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 5px;
}
/* Style untuk info-item di dalam form */
.readonly-in-form {
  background-color: #f0f0f0; /* Beri latar belakang abu-abu */
  border-radius: 4px;
}

/* Style untuk Grup Tombol */
.button-group {
  margin-top: 20px;
  display: flex;
  gap: 10px; /* Memberi jarak antar tombol */
}
.button-group button {
  flex-grow: 1; /* Membuat tombol memenuhi space */
  font-size: 16px;
  font-weight: bold;
  padding: 12px 10px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}

.submit-button {
  /* width: 100%; */ /* Tidak perlu jika flex-grow: 1 */
  background-color: #28a745;
  color: white;
}
.submit-button:disabled {
  background-color: #aaa;
}
.confirm-button {
  background-color: #28a745;
  color: white;
}
.edit-button {
  background-color: #ffc107;
  color: #212529;
}
.cancel-button {
  background-color: #6c757d;
  color: white;
}

/* Style untuk status message */
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
