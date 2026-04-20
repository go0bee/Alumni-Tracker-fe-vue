<script setup>
import { ref, onMounted } from "vue";

const API_BASE_URL = import.meta.env.VITE_API_URL;

const alumniList = ref([]);
const newAlumni = ref({
  nama: "",
  keywords: "",
});
const editMode = ref(false);
const editingAlumniId = ref(null);
const editingAlumni = ref({ nama: "", keywords: "" });
const loading = ref(false);
const message = ref("");
const activeTab = ref("tracking");
const trackingResult = ref(null);
const selectedEvidence = ref(null);
const viewingAlumniId = ref(null);
const newEvidence = ref({
  source_name: "Manual Verifikasi",
  raw_data_url: "",
  snippet_content: "",
  extracted_score: 1.0,
});
const excelFile = ref(null);
const excelInputRef = ref(null);

// Fetch all targets from backend
const fetchAlumni = async () => {
  try {
    const response = await fetch(`${API_BASE_URL}/targets/`);

    const text = await response.text(); // ambil response mentah
    console.log("API URL:", API_BASE_URL);
    console.log("RAW RESPONSE:", text);
    console.log("ENV:", import.meta.env);
    console.log("API:", import.meta.env.VITE_API_URL);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${text}`);
    }

    const data = JSON.parse(text);
    alumniList.value = data;
  } catch (error) {
    console.error("Error fetching alumni:", error);
    message.value = "Gagal mengambil data alumni";
  }
};

// Fetch evidence for a specific alumni
const fetchEvidence = async (id) => {
  try {
    const response = await fetch(`${API_BASE_URL}/evidence/${id}`);
    if (!response.ok) throw new Error("Failed to fetch evidence");
    selectedEvidence.value = await response.json();
    viewingAlumniId.value = id;
    activeTab.value = "hasil";
    // Reset manual form when viewing different alumni
    newEvidence.value = {
      source_name: "Manual Verifikasi",
      raw_data_url: "",
      snippet_content: "",
      extracted_score: 1.0,
    };
  } catch (error) {
    console.error("Error fetching evidence:", error);
    message.value = "Gagal mengambil bukti pelacakan";
  }
};

// Submit manual evidence
const submitManualEvidence = async () => {
  if (!newEvidence.value.raw_data_url || !newEvidence.value.snippet_content) {
    message.value = "URL dan Konten bukti harus diisi";
    return;
  }

  loading.value = true;
  try {
    const response = await fetch(
      `${API_BASE_URL}/evidence/${viewingAlumniId.value}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(newEvidence.value),
      },
    );

    if (!response.ok) throw new Error("Failed to add manual evidence");

    message.value = "Bukti manual berhasil ditambahkan";
    // Refresh evidence list
    await fetchEvidence(viewingAlumniId.value);
    // Refresh alumni list to see score/status updates
    await fetchAlumni();
  } catch (error) {
    console.error("Error adding manual evidence:", error);
    message.value = "Gagal menambahkan bukti manual";
  } finally {
    loading.value = false;
  }
};

// Delete evidence
const deleteEvidence = async (evidenceId) => {
  if (!confirm("Apakah Anda yakin ingin menghapus bukti ini?")) return;

  loading.value = true;
  try {
    const response = await fetch(`${API_BASE_URL}/evidence/${evidenceId}`, {
      method: "DELETE",
    });

    if (!response.ok) throw new Error("Failed to delete evidence");

    message.value = "Bukti berhasil dihapus";
    // Refresh evidence list
    await fetchEvidence(viewingAlumniId.value);
    // Refresh alumni list to see score/status updates
    await fetchAlumni();
  } catch (error) {
    console.error("Error deleting evidence:", error);
    message.value = "Gagal menghapus bukti";
  } finally {
    loading.value = false;
  }
};

// Add new alumni target
const addAlumni = async () => {
  if (!newAlumni.value.nama || !newAlumni.value.keywords) {
    message.value = "Nama dan Keywords harus diisi";
    return;
  }

  loading.value = true;
  try {
    const response = await fetch(`${API_BASE_URL}/targets/`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(newAlumni.value),
    });

    if (!response.ok) throw new Error("Failed to add alumni");

    await fetchAlumni();
    newAlumni.value = { nama: "", keywords: "" };
    message.value = "Alumni berhasil ditambahkan";
  } catch (error) {
    console.error("Error adding alumni:", error);
    message.value = "Gagal menambahkan alumni";
  } finally {
    loading.value = false;
  }
};

const startEditAlumni = (alumni) => {
  editMode.value = true;
  editingAlumniId.value = alumni.id;
  editingAlumni.value = { nama: alumni.nama_asli, keywords: alumni.keywords };
  message.value = 'Mode edit aktif: ubah data dan tekan "Simpan Perubahan"';
};

const cancelEditAlumni = () => {
  editMode.value = false;
  editingAlumniId.value = null;
  editingAlumni.value = { nama: "", keywords: "" };
  message.value = "";
};

const updateAlumni = async () => {
  if (!editingAlumni.value.nama || !editingAlumni.value.keywords) {
    message.value = "Nama dan Keywords harus diisi untuk update";
    return;
  }
  loading.value = true;
  try {
    const response = await fetch(
      `${API_BASE_URL}/targets/${editingAlumniId.value}`,
      {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(editingAlumni.value),
      },
    );
    if (!response.ok) throw new Error("Failed to update alumni");
    await fetchAlumni();
    cancelEditAlumni();
    message.value = "Alumni berhasil diperbarui";
  } catch (error) {
    console.error("Error updating alumni:", error);
    message.value = "Gagal memperbarui alumni";
  } finally {
    loading.value = false;
  }
};

const deleteAlumni = async (alumniId) => {
  if (
    !confirm(
      "Apakah Anda yakin ingin menghapus alumni ini dan seluruh bukti terkait?",
    )
  )
    return;
  loading.value = true;
  try {
    const response = await fetch(`${API_BASE_URL}/targets/${alumniId}`, {
      method: "DELETE",
    });
    if (!response.ok) throw new Error("Failed to delete alumni");
    await fetchAlumni();
    if (viewingAlumniId.value === alumniId) {
      selectedEvidence.value = null;
      viewingAlumniId.value = null;
    }
    message.value = "Alumni berhasil dihapus";
  } catch (error) {
    console.error("Error deleting alumni:", error);
    message.value = "Gagal menghapus alumni";
  } finally {
    loading.value = false;
  }
};

// Trigger tracking for an alumni
const trackAlumni = async (id) => {
  loading.value = true;
  message.value = "Sedang melacak...";
  try {
    const response = await fetch(`${API_BASE_URL}/track/${id}`);
    if (!response.ok) throw new Error("Failed to track alumni");

    const result = await response.json();
    message.value = `Pelacakan selesai: Status ${result.current_status}, Score ${result.score}`;
    trackingResult.value = {
      targetId: id,
      status: result.current_status,
      score: result.score,
      top_results: result.detail.top_results || [],
      best_match: result.detail.best_match || null,
    };
    await fetchAlumni();
    activeTab.value = "hasil";
  } catch (error) {
    console.error("Error tracking alumni:", error);
    message.value = "Gagal melacak alumni";
  } finally {
    loading.value = false;
  }
};

// Handle file input
const handleExcelUpload = (event) => {
  excelFile.value = event.target.files[0];
};

// Upload excel to backend
const uploadExcel = async () => {
  if (!excelFile.value) {
    message.value = "Pilih file Excel terlebih dahulu (.xlsx)";
    return;
  }

  loading.value = true;
  message.value = "Sedang upload dan import Excel...";

  try {
    const formData = new FormData();
    formData.append("file", excelFile.value);

    const response = await fetch(`${API_BASE_URL}/admin/import-excel`, {
      method: "POST",
      body: formData,
    });

    if (!response.ok) {
      const errText = await response.text();
      throw new Error(errText);
    }

    const result = await response.json();
    message.value = `Import selesai. Inserted: ${result.inserted}, Skipped: ${result.skipped}`;

    // excelFile.value = null;
    await fetchAlumni();
  } catch (error) {
    console.error("Error uploading excel:", error);
    message.value = "Gagal upload/import Excel";
  } finally {
    excelFile.value = null;
    if (excelInputRef.value) excelInputRef.value.value = "";
    loading.value = false;
  }
};

// Start tracking all alumni
const startTrackingAll = async () => {
  loading.value = true;
  message.value = "Sedang menjalankan tracking semua alumni...";

  try {
    const response = await fetch(`${API_BASE_URL}/track/start`, {
      method: "POST",
    });

    if (!response.ok) {
      const errText = await response.text();
      throw new Error(errText);
    }

    const result = await response.json();
    message.value = `Tracking selesai. Total Target: ${result.total_targets}, Evidence ditambah: ${result.total_evidence_added}`;

    await fetchAlumni();
    activeTab.value = "hasil";
  } catch (error) {
    console.error("Error start tracking all:", error);
    message.value = "Gagal menjalankan tracking semua alumni";
  } finally {
    loading.value = false;
  }
};

onMounted(() => {
  fetchAlumni();
});
</script>

<template>
  <div class="alumni-container">
    <!-- Header -->
    <div class="header-section">
      <div class="header-content">
        <h1 class="main-title">🎓 Alumni Tracker</h1>
        <p class="subtitle">
          Sistem pelacakan dan verifikasi status alumni secara real-time
        </p>
      </div>
    </div>

    <!-- Tab Navigation -->
    <div class="tab-navigation">
      <button
        :class="['tab-btn', { active: activeTab === 'tracking' }]"
        @click="activeTab = 'tracking'"
      >
        <span class="tab-icon">📋</span>
        Menu Tracking
      </button>
      <button
        :class="['tab-btn', { active: activeTab === 'hasil' }]"
        @click="activeTab = 'hasil'"
      >
        <span class="tab-icon">📊</span>
        Menu Hasil
      </button>
    </div>

    <!-- Alert Message -->
    <transition name="fade">
      <div
        v-if="message"
        :class="[
          'alert',
          message.includes('Gagal') ? 'alert-error' : 'alert-success',
        ]"
      >
        {{ message }}
      </div>
    </transition>

    <!-- Tracking Tab -->
    <div v-if="activeTab === 'tracking'" class="tab-content">
      <!-- Import Excel + Start Tracking All -->
      <div class="card add-card">
        <div class="card-header">
          <h2>📂 Import Excel Alumni</h2>
          <p class="card-subtitle">
            Upload file Excel alumni (.xlsx) lalu jalankan tracking otomatis
          </p>
        </div>

        <div class="card-body">
          <div class="form-group">
            <input
              ref="excelInputRef"
              type="file"
              accept=".xlsx"
              class="input-field"
              @change="handleExcelUpload"
            />

            <button
              @click="uploadExcel"
              :disabled="loading"
              class="btn btn-primary btn-block"
            >
              <span v-if="!loading">⬆️ Upload & Import Excel</span>
              <span v-else>Processing...</span>
            </button>

            <button
              @click="startTrackingAll"
              :disabled="loading"
              class="btn btn-track btn-block"
            >
              <span v-if="!loading">🚀 Start Tracking Semua Alumni</span>
              <span v-else>Tracking...</span>
            </button>
          </div>
        </div>
      </div>
      <!-- Add Alumni Section -->
      <div class="card add-card">
        <div class="card-header">
          <h2>➕ Tambah Alumni Baru</h2>
          <p class="card-subtitle">
            Masukkan nama dan keywords untuk memulai pelacakan
          </p>
        </div>
        <div class="card-body">
          <div class="form-group">
            <div class="input-wrapper">
              <input
                v-if="editMode"
                v-model="editingAlumni.nama"
                placeholder="Edit Nama Alumni"
                class="input-field"
              />

              <input
                v-else
                v-model="newAlumni.nama"
                placeholder="Nama Lengkap Alumni"
                class="input-field"
              />
              <span class="input-icon">👤</span>
            </div>
            <div class="input-wrapper">
              <input
                v-if="editMode"
                v-model="editingAlumni.keywords"
                placeholder="Edit Keywords"
                class="input-field"
              />

              <input
                v-else
                v-model="newAlumni.keywords"
                placeholder="Keywords (LinkedIn, Company, dll)"
                class="input-field"
              />
              <span class="input-icon">🔍</span>
            </div>
            <div class="button-row">
              <button
                v-if="!editMode"
                @click="addAlumni"
                :disabled="loading"
                class="btn btn-primary btn-block"
              >
                <span v-if="!loading">Tambah Alumni</span>
                <span v-else>Menambahkan...</span>
              </button>
              <button
                v-else
                @click="updateAlumni"
                :disabled="loading"
                class="btn btn-primary btn-block"
              >
                <span v-if="!loading">Simpan Perubahan</span>
                <span v-else>Menyimpan...</span>
              </button>
              <button
                v-if="editMode"
                @click="cancelEditAlumni"
                class="btn btn-secondary btn-block"
              >
                Batalkan
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Alumni List Section -->
      <div class="card">
        <div class="card-header">
          <h2>👥 Daftar Alumni</h2>
          <p class="card-subtitle">Total: {{ alumniList.length }} alumni</p>
        </div>
        <div class="card-body">
          <div v-if="alumniList.length === 0" class="empty-state">
            <p class="empty-icon">📭</p>
            <p>Belum ada data alumni. Mulai dengan menambah alumni baru!</p>
          </div>
          <div v-else class="alumni-grid">
            <div
              v-for="alumni in alumniList"
              :key="alumni.id"
              class="alumni-card"
            >
              <div class="alumni-header">
                <div class="alumni-name-info">
                  <h3>{{ alumni.nama_asli }}</h3>
                  <p class="alumni-id">ID: #{{ alumni.id }}</p>
                </div>
                <span :class="['status-badge', alumni.status.toLowerCase()]">
                  {{ alumni.status }}
                </span>
              </div>

              <div class="alumni-details">
                <div class="detail-row">
                  <span class="label">Keywords:</span>
                  <span class="value">{{ alumni.keywords }}</span>
                </div>
                <div class="detail-row">
                  <span class="label">Score:</span>
                  <div class="score-bar">
                    <div
                      class="score-fill"
                      :style="{ width: alumni.confidence_score * 100 + '%' }"
                    ></div>
                    <span class="score-text">{{
                      alumni.confidence_score
                    }}</span>
                  </div>
                </div>
              </div>

              <div class="alumni-actions">
                <button
                  @click="trackAlumni(alumni.id)"
                  :disabled="loading"
                  class="btn btn-track"
                >
                  🚀 Track
                </button>
                <button
                  @click="fetchEvidence(alumni.id)"
                  class="btn btn-detail"
                >
                  🔎 Details
                </button>
                <button @click="startEditAlumni(alumni)" class="btn btn-edit">
                  ✏️ Edit
                </button>
                <button @click="deleteAlumni(alumni.id)" class="btn btn-delete">
                  🗑️ Hapus
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Results Tab -->
    <div v-if="activeTab === 'hasil'" class="tab-content">
      <!-- Latest Results Section -->
      <div class="card">
        <div class="card-header">
          <h2>📈 Hasil Pelacakan Terakhir</h2>
        </div>
        <div class="card-body">
          <div v-if="!trackingResult" class="empty-state">
            <p class="empty-icon">🎯</p>
            <p>
              Mulai tracking pada menu Tracking untuk melihat hasil langsung
            </p>
          </div>
          <div v-else class="tracking-result">
            <div class="result-header">
              <div class="result-info">
                <p>
                  <strong>Target ID:</strong>
                  <span class="highlight">#{{ trackingResult.targetId }}</span>
                </p>
                <p>
                  <strong>Status:</strong>
                  <span class="highlight">{{ trackingResult.status }}</span>
                </p>
                <p>
                  <strong>Confidence Score:</strong>
                  <span class="highlight"
                    >{{ (trackingResult.score * 100).toFixed(1) }}%</span
                  >
                </p>
              </div>
            </div>

            <div class="results-section">
              <h3>🏆 Top Hasil</h3>
              <div
                v-if="trackingResult.top_results.length === 0"
                class="no-results"
              >
                <p>Tidak ada hasil relevan ditemukan</p>
              </div>
              <div
                v-for="(item, idx) in trackingResult.top_results"
                :key="idx"
                class="result-item"
              >
                <div class="result-rank">{{ idx + 1 }}</div>
                <div class="result-content">
                  <h4>{{ item.title }}</h4>
                  <p class="snippet">{{ item.snippet }}</p>
                  <a :href="item.link" target="_blank" class="result-link"
                    >Lihat Link →</a
                  >
                </div>
                <div class="result-score">
                  <span class="score-badge"
                    >{{ (item.score * 100).toFixed(0) }}%</span
                  >
                </div>
              </div>
            </div>

            <div v-if="trackingResult.best_match" class="results-section">
              <h3>⭐ Best Match</h3>
              <div class="best-match-card">
                <h4>{{ trackingResult.best_match.title }}</h4>
                <p class="snippet">{{ trackingResult.best_match.snippet }}</p>
                <a
                  :href="trackingResult.best_match.link"
                  target="_blank"
                  class="result-link"
                  >Lihat Link →</a
                >
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Evidence Section -->
      <div v-if="selectedEvidence" class="card evidence-card">
        <div class="card-header">
          <h2>🔐 Detail Evidence - Alumni #{{ viewingAlumniId }}</h2>
          <button @click="selectedEvidence = null" class="btn btn-close">
            ✕ Tutup
          </button>
        </div>

        <div class="card-body">
          <!-- Manual Evidence Form -->
          <div class="manual-evidence-form">
            <h3>➕ Tambah Bukti Manual</h3>
            <div class="form-group">
              <div class="input-wrapper">
                <input
                  v-model="newEvidence.source_name"
                  placeholder="Source Name (LinkedIn, etc)"
                  class="input-field"
                />
              </div>
              <div class="input-wrapper">
                <input
                  v-model="newEvidence.raw_data_url"
                  placeholder="URL Bukti"
                  class="input-field"
                />
              </div>
              <textarea
                v-model="newEvidence.snippet_content"
                placeholder="Konten Snippet / Bukti"
                class="input-field textarea-field"
              ></textarea>
              <div class="score-group">
                <label>Confidence Score (0-1):</label>
                <input
                  type="number"
                  step="0.1"
                  min="0"
                  max="1"
                  v-model="newEvidence.extracted_score"
                  class="score-input"
                />
              </div>
              <button
                @click="submitManualEvidence"
                :disabled="loading"
                class="btn btn-primary btn-block"
              >
                ✅ Tambah Bukti
              </button>
            </div>
          </div>

          <!-- Evidence List -->
          <div class="evidence-list">
            <h3>📋 Bukti Pelacakan</h3>
            <div v-if="selectedEvidence.length === 0" class="empty-state">
              <p>Belum ada bukti pelacakan untuk alumni ini</p>
            </div>
            <div
              v-for="item in selectedEvidence"
              :key="item.id"
              class="evidence-item"
            >
              <div class="evidence-header">
                <h4>{{ item.source_name }}</h4>
                <div class="evidence-badge-group">
                  <span class="score-badge"
                    >{{ (item.extracted_score * 100).toFixed(0) }}%</span
                  >
                  <button
                    @click="deleteEvidence(item.id)"
                    class="btn-delete-mini"
                    title="Hapus Bukti"
                  >
                    🗑️
                  </button>
                </div>
              </div>
              <p class="evidence-content">{{ item.snippet_content }}</p>
              <a :href="item.raw_data_url" target="_blank" class="evidence-link"
                >🔗 Lihat Sumber Data</a
              >
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.alumni-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 40px 20px;
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
}

/* Header Section */
.header-section {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 60px 40px;
  text-align: center;
  margin: -40px -20px 40px -20px;
  color: white;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
}

.header-content h1.main-title {
  font-size: 3em;
  font-weight: 700;
  margin-bottom: 10px;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.subtitle {
  font-size: 1.1em;
  opacity: 0.95;
  font-weight: 300;
}

/* Tab Navigation */
.tab-navigation {
  display: flex;
  gap: 15px;
  margin-bottom: 40px;
  justify-content: center;
}

.tab-btn {
  padding: 12px 28px;
  background: white;
  border: 2px solid white;
  border-radius: 50px;
  font-size: 1em;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
  color: #667eea;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.tab-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.15);
}

.tab-btn.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.tab-icon {
  font-size: 1.2em;
}

/* Alert Messages */
.alert {
  padding: 16px 20px;
  border-radius: 12px;
  margin-bottom: 30px;
  font-weight: 500;
  max-width: 1000px;
  margin-left: auto;
  margin-right: auto;
  animation: slideIn 0.3s ease;
}

.alert-success {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  border-left: 5px solid #047857;
}

.alert-error {
  background: linear-gradient(135deg, #ef4444 0%, #dc2626 100%);
  color: white;
  border-left: 5px solid #991b1b;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Tab Content */
.tab-content {
  max-width: 1000px;
  margin: 0 auto;
}

/* Card Component */
.card {
  background: white;
  border-radius: 16px;
  margin-bottom: 30px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15);
  overflow: hidden;
  transition: all 0.3s ease;
}

.card:hover {
  box-shadow: 0 15px 50px rgba(0, 0, 0, 0.2);
}

.add-card {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.add-card .card-header {
  color: white;
}

.add-card .card-subtitle {
  color: rgba(255, 255, 255, 0.9);
}

.add-card input,
.add-card textarea {
  background: rgba(255, 255, 255, 0.95);
}

.card-header {
  padding: 30px;
  border-bottom: 1px solid #f3f4f6;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card-header h2 {
  font-size: 1.5em;
  color: #1f2937;
  margin-bottom: 5px;
}

.card-subtitle {
  color: #6b7280;
  font-size: 0.95em;
  font-weight: 400;
}

.card-body {
  padding: 30px;
}

.btn-close {
  background: #ef4444 !important;
  padding: 8px 16px !important;
  font-size: 0.9em !important;
}

/* Form Elements */
.form-group {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.input-wrapper {
  position: relative;
}

.input-field {
  width: 100%;
  padding: 12px 16px 12px 45px;
  border: 2px solid #e5e7eb;
  border-radius: 10px;
  font-size: 1em;
  transition: all 0.3s ease;
  font-family: inherit;
  color: #1f2937; /* ini fixnya */
}

.input-field::placeholder {
  color: #9ca3af;
}

.input-field:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.textarea-field {
  min-height: 100px;
  resize: vertical;
  padding: 12px 16px;
}

.input-icon {
  position: absolute;
  left: 15px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 1.2em;
  pointer-events: none;
}

.score-group {
  display: flex;
  align-items: center;
  gap: 16px;
}

.score-group label {
  font-weight: 600;
  color: #374151;
  min-width: 180px;
}

.score-input {
  flex: 1;
  max-width: 120px;
  padding: 8px 12px;
  border: 2px solid #e5e7eb;
  border-radius: 8px;
  font-size: 0.95em;
}

/* Button Styles */
.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 10px;
  font-size: 0.95em;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.btn-secondary {
  background: #e5e7eb;
  color: #111827;
  border: 1px solid #cbd5e1;
}

.btn-edit {
  background: #f59e0b;
  color: white;
  padding: 8px 16px;
  font-size: 0.9em;
}

.btn-delete {
  background: #ef4444;
  color: white;
  padding: 8px 16px;
  font-size: 0.9em;
}

.btn-block {
  width: 100%;
}

.btn-track {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  padding: 8px 16px;
  font-size: 0.9em;
}

.btn-detail {
  background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
  color: white;
  padding: 8px 16px;
  font-size: 0.9em;
}

/* Alumni Grid */
.alumni-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 24px;
}

.alumni-card {
  background: #f9fafb;
  border: 2px solid #e5e7eb;
  border-radius: 12px;
  padding: 20px;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.alumni-card:hover {
  border-color: #667eea;
  box-shadow: 0 10px 30px rgba(102, 126, 234, 0.15);
}

.alumni-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 10px;
}

.alumni-name-info h3 {
  font-size: 1.3em;
  color: #1f2937;
  margin-bottom: 5px;
}

.alumni-id {
  color: #6b7280;
  font-size: 0.9em;
  font-weight: 500;
}

.alumni-details {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
}

.detail-row .label {
  font-weight: 600;
  color: #6b7280;
  font-size: 0.9em;
}

.detail-row .value {
  color: #1f2937;
  font-weight: 500;
  flex: 1;
  text-align: right;
}

.score-bar {
  flex: 1;
  height: 24px;
  background: #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  position: relative;
}

.score-fill {
  height: 100%;
  background: linear-gradient(90deg, #10b981 0%, #059669 100%);
  transition: width 0.3s ease;
}

.score-text {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  color: white;
  font-weight: 600;
  font-size: 0.8em;
}

.alumni-actions {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.alumni-actions .btn {
  flex: 1;
  justify-content: center;
}

/* Status Badges */
.status-badge {
  padding: 8px 16px;
  border-radius: 20px;
  font-size: 0.85em;
  font-weight: 700;
  white-space: nowrap;
}

.status-badge.untracked {
  background: #f3f4f6;
  color: #6b7280;
}

.status-badge.tracking {
  background: linear-gradient(135deg, #fbbf24 0%, #f59e0b 100%);
  color: white;
}

.status-badge.found {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
}

.status-badge.not_found {
  background: linear-gradient(135deg, #ef4444 0%, #dc2626 100%);
  color: white;
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 60px 40px;
  color: #6b7280;
}

.empty-icon {
  font-size: 4em;
  margin-bottom: 20px;
  display: block;
}

.empty-state p {
  font-size: 1.1em;
  line-height: 1.6;
}

/* Tracking Result */
.tracking-result {
  display: flex;
  flex-direction: column;
  gap: 30px;
}

.result-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
  border-radius: 12px;
  color: white;
}

.result-info p {
  margin: 8px 0;
  font-size: 1em;
}

.highlight {
  background: rgba(255, 255, 255, 0.2);
  padding: 4px 8px;
  border-radius: 6px;
  font-weight: 600;
}

.results-section {
  border-top: 2px solid #f3f4f6;
  padding-top: 24px;
}

.results-section h3 {
  font-size: 1.2em;
  color: #1f2937;
  margin-bottom: 20px;
}

.result-item {
  display: flex;
  gap: 16px;
  padding: 20px;
  background: #f9fafb;
  border-radius: 12px;
  border-left: 4px solid #667eea;
  margin-bottom: 16px;
  transition: all 0.3s ease;
}

.result-item:hover {
  background: #f3f4f6;
  transform: translateX(4px);
}

.result-rank {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 50%;
  font-weight: 700;
  flex-shrink: 0;
}

.result-content {
  flex: 1;
}

.result-content h4 {
  font-size: 1.1em;
  color: #1f2937;
  margin-bottom: 8px;
}

.snippet {
  color: #6b7280;
  font-size: 0.95em;
  line-height: 1.5;
  margin-bottom: 12px;
  font-style: italic;
}

.result-link {
  color: #667eea;
  text-decoration: none;
  font-weight: 600;
  transition: all 0.3s ease;
}

.result-link:hover {
  color: #764ba2;
  text-decoration: underline;
}

.result-score {
  display: flex;
  align-items: center;
}

.score-badge {
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  padding: 8px 16px;
  border-radius: 20px;
  font-weight: 700;
  font-size: 0.9em;
}

.best-match-card {
  background: linear-gradient(135deg, #fbbf24 0%, #f59e0b 100%);
  padding: 24px;
  border-radius: 12px;
  color: white;
}

.best-match-card h4 {
  font-size: 1.2em;
  margin-bottom: 12px;
}

.best-match-card .snippet {
  color: rgba(255, 255, 255, 0.9);
  margin-bottom: 16px;
}

.best-match-card .result-link {
  color: white;
  font-weight: 700;
}

/* Evidence Section */
.evidence-card {
  border-top: 5px solid #667eea;
}

.manual-evidence-form {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 24px;
  border-radius: 12px;
  margin-bottom: 30px;
}

.manual-evidence-form h3 {
  color: white;
  margin-bottom: 20px;
  font-size: 1.2em;
}

.manual-evidence-form .input-field {
  background: white;
  border-color: #e5e7eb;
}

.manual-evidence-form .btn-primary {
  margin-top: 16px;
}

.evidence-list {
  border-top: 2px solid #f3f4f6;
  padding-top: 24px;
}

.evidence-list h3 {
  font-size: 1.2em;
  color: #1f2937;
  margin-bottom: 20px;
}

.evidence-item {
  background: #f9fafb;
  padding: 20px;
  border-radius: 12px;
  border-left: 4px solid #3b82f6;
  margin-bottom: 16px;
  transition: all 0.3s ease;
}

.evidence-item:hover {
  background: #f3f4f6;
  transform: translateX(4px);
}

.evidence-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.evidence-header h4 {
  color: #1f2937;
  font-size: 1.05em;
}

.evidence-content {
  color: #6b7280;
  line-height: 1.6;
  margin-bottom: 12px;
  font-style: italic;
}

.evidence-link {
  color: #3b82f6;
  text-decoration: none;
  font-weight: 600;
  transition: all 0.3s ease;
}

.evidence-link:hover {
  color: #1d4ed8;
  text-decoration: underline;
}

.no-results {
  text-align: center;
  padding: 30px;
  color: #6b7280;
}

/* Responsive Design */
@media (max-width: 768px) {
  .alumni-container {
    padding: 20px 10px;
  }

  .header-section {
    padding: 40px 20px;
    margin: -20px -10px 30px -10px;
  }

  .header-content h1.main-title {
    font-size: 2em;
  }

  .tab-navigation {
    flex-direction: column;
  }

  .tab-btn {
    width: 100%;
    justify-content: center;
  }

  .card-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }

  .alumni-grid {
    grid-template-columns: 1fr;
  }

  .alumni-actions {
    flex-direction: column;
  }

  .score-group {
    flex-direction: column;
    align-items: flex-start;
  }

  .score-group label {
    min-width: auto;
  }

  .result-item {
    flex-direction: column;
  }

  .result-rank {
    align-self: flex-start;
  }
}

/* Transitions */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
