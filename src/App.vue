<script setup>
import { ref, computed } from "vue";
import html2canvas from "html2canvas";

// State
const items = ref([]);
const participants = ref([]);
const newItem = ref({ name: "", price: "", qty: 1 });
const displayPrice = ref("");
const newParticipant = ref("");
const taxPercent = ref(10);
const serviceValue = ref(0);
const serviceType = ref("percent"); // 'percent' or 'fixed'
const resultRef = ref(null);
const showGuide = ref(false);

// Format number to Indonesian format (with dots as thousand separator)
const formatInputNumber = (value) => {
  // Remove all non-digit characters
  const numericValue = value.replace(/\D/g, "");
  // Format with dots as thousand separator
  return numericValue.replace(/\B(?=(\d{3})+(?!\d))/g, ".");
};

// Handle price input
const handlePriceInput = (e) => {
  const value = e.target.value;
  const formatted = formatInputNumber(value);
  displayPrice.value = formatted;
  // Store the raw numeric value
  newItem.value.price = formatted.replace(/\./g, "");
};

// Add new item
const addItem = () => {
  const rawPrice = newItem.value.price.toString().replace(/\./g, "");
  if (newItem.value.name && parseFloat(rawPrice) > 0) {
    items.value.push({
      id: Date.now(),
      name: newItem.value.name,
      price: parseFloat(rawPrice),
      qty: parseInt(newItem.value.qty) || 1,
      assignedTo: [],
    });
    newItem.value = { name: "", price: "", qty: 1 };
    displayPrice.value = "";
  }
};

// Remove item
const removeItem = (id) => {
  items.value = items.value.filter((item) => item.id !== id);
};

// Add participant
const addParticipant = () => {
  if (
    newParticipant.value.trim() &&
    !participants.value.includes(newParticipant.value.trim())
  ) {
    participants.value.push(newParticipant.value.trim());
    newParticipant.value = "";
  }
};

// Remove participant
const removeParticipant = (name) => {
  participants.value = participants.value.filter((p) => p !== name);
  // Remove from all item assignments
  items.value.forEach((item) => {
    item.assignedTo = item.assignedTo.filter((p) => p !== name);
  });
};

// Toggle item assignment
const toggleAssignment = (item, participant) => {
  const index = item.assignedTo.indexOf(participant);
  if (index === -1) {
    item.assignedTo.push(participant);
  } else {
    item.assignedTo.splice(index, 1);
  }
};

// Calculate subtotal per item
const getItemTotal = (item) => {
  return item.price * item.qty;
};

// Calculate per person share for an item
const getItemSharePerPerson = (item) => {
  if (item.assignedTo.length === 0) return 0;
  return getItemTotal(item) / item.assignedTo.length;
};

// Calculate subtotal (before tax & service)
const subtotal = computed(() => {
  return items.value.reduce((sum, item) => sum + getItemTotal(item), 0);
});

// Calculate tax amount
const taxAmount = computed(() => {
  return subtotal.value * (taxPercent.value / 100);
});

// Calculate service charge
const serviceAmount = computed(() => {
  if (serviceType.value === "fixed") {
    return parseFloat(serviceValue.value) || 0;
  }
  return subtotal.value * (serviceValue.value / 100);
});

// Calculate grand total
const grandTotal = computed(() => {
  return subtotal.value + taxAmount.value + serviceAmount.value;
});

// Get participants who have items assigned (for fixed service split)
const activeParticipants = computed(() => {
  const active = new Set();
  items.value.forEach((item) => {
    item.assignedTo.forEach((p) => active.add(p));
  });
  return Array.from(active);
});

// Calculate each participant's total
const participantTotals = computed(() => {
  const totals = {};

  participants.value.forEach((p) => {
    totals[p] = {
      name: p,
      items: [],
      itemsTotal: 0,
      taxShare: 0,
      serviceShare: 0,
      total: 0,
    };
  });

  // Calculate items per person
  items.value.forEach((item) => {
    if (item.assignedTo.length > 0) {
      const sharePerPerson = getItemSharePerPerson(item);
      item.assignedTo.forEach((p) => {
        if (totals[p]) {
          totals[p].items.push({
            name: item.name,
            qty: item.qty,
            share: sharePerPerson,
            sharedWith: item.assignedTo.length,
          });
          totals[p].itemsTotal += sharePerPerson;
        }
      });
    }
  });

  // Calculate tax & service share
  const totalItemsAssigned = Object.values(totals).reduce(
    (sum, p) => sum + p.itemsTotal,
    0
  );

  if (totalItemsAssigned > 0) {
    const numActiveParticipants = activeParticipants.value.length;

    Object.keys(totals).forEach((p) => {
      const proportion = totals[p].itemsTotal / totalItemsAssigned;
      const hasItems = totals[p].itemsTotal > 0;

      // Tax is always proportional
      totals[p].taxShare = taxAmount.value * proportion;

      // Service: fixed = split equally, percent = proportional
      if (
        serviceType.value === "fixed" &&
        hasItems &&
        numActiveParticipants > 0
      ) {
        totals[p].serviceShare = serviceAmount.value / numActiveParticipants;
      } else {
        totals[p].serviceShare = serviceAmount.value * proportion;
      }

      totals[p].total =
        totals[p].itemsTotal + totals[p].taxShare + totals[p].serviceShare;
    });
  }

  return Object.values(totals).filter((p) => p.total > 0);
});

// Format currency
const formatCurrency = (amount) => {
  return new Intl.NumberFormat("id-ID", {
    style: "currency",
    currency: "IDR",
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
  }).format(amount);
};

// Get today's date
const today = new Date().toLocaleDateString("id-ID", {
  weekday: "long",
  year: "numeric",
  month: "long",
  day: "numeric",
});

// Download as image
const downloadAsImage = async () => {
  if (!resultRef.value) return;

  try {
    const canvas = await html2canvas(resultRef.value, {
      backgroundColor: "#1e293b",
      scale: 2,
      useCORS: true,
    });

    const link = document.createElement("a");
    link.download = `split-bill-${new Date().toISOString().split("T")[0]}.png`;
    link.href = canvas.toDataURL("image/png");
    link.click();
  } catch (error) {
    console.error("Error generating image:", error);
    alert("Gagal membuat gambar. Silakan coba lagi.");
  }
};

// Handle Enter key
const handleItemEnter = (e) => {
  if (e.key === "Enter") addItem();
};

const handleParticipantEnter = (e) => {
  if (e.key === "Enter") addParticipant();
};
</script>

<template>
  <div class="container">
    <!-- Header -->
    <header class="header">
      <h1>💰 Split Bill</h1>
      <p>Hitung patungan dengan mudah!</p>
      <button
        class="guide-btn"
        @click="showGuide = true"
        title="Panduan Penggunaan"
      >
        ❓ Cara Pakai
      </button>
    </header>

    <!-- Guide Modal -->
    <div class="modal-overlay" v-if="showGuide" @click.self="showGuide = false">
      <div class="modal-content">
        <div class="modal-header">
          <h2>📖 Panduan Penggunaan</h2>
          <button class="modal-close" @click="showGuide = false">×</button>
        </div>
        <div class="modal-body">
          <div class="guide-step">
            <span class="guide-number">1</span>
            <div class="guide-text">
              <strong>Tambahkan Peserta</strong>
              <p>
                Masukkan nama-nama orang yang ikut patungan di bagian Setup.
              </p>
            </div>
          </div>
          <div class="guide-step">
            <span class="guide-number">2</span>
            <div class="guide-text">
              <strong>Atur Pajak & Service</strong>
              <p>
                Masukkan persentase pajak dan service (bisa dalam % atau nominal
                Rp).
              </p>
            </div>
          </div>
          <div class="guide-step">
            <span class="guide-number">3</span>
            <div class="guide-text">
              <strong>Tambahkan Menu</strong>
              <p>
                Input nama menu, harga, dan jumlah pesanan. Tekan + atau Enter
                untuk menambah.
              </p>
            </div>
          </div>
          <div class="guide-step">
            <span class="guide-number">4</span>
            <div class="guide-text">
              <strong>Pilih Siapa yang Pesan</strong>
              <p>
                Klik nama peserta di bawah setiap menu untuk menandai siapa yang
                memesan.
              </p>
            </div>
          </div>
          <div class="guide-step">
            <span class="guide-number">5</span>
            <div class="guide-text">
              <strong>Lihat Ringkasan</strong>
              <p>
                Hasil pembagian akan muncul otomatis. Klik tombol download untuk
                menyimpan sebagai gambar.
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Step 1: Setup - Peserta & Pajak/Service -->
    <section class="section">
      <div class="section-header">
        <span class="step-badge">1</span>
        <h2 class="section-title">Setup</h2>
      </div>

      <div class="grid-2">
        <!-- Peserta Card -->
        <div class="card">
          <h3 class="card-title">
            <span class="icon">👥</span>
            Peserta
          </h3>

          <div class="form-row">
            <div class="form-group">
              <input
                type="text"
                class="form-input"
                v-model="newParticipant"
                placeholder="Masukkan nama peserta..."
                @keyup="handleParticipantEnter"
              />
            </div>
            <button class="btn btn-primary btn-icon" @click="addParticipant">
              +
            </button>
          </div>

          <div class="participant-tags" v-if="participants.length > 0">
            <span class="participant-tag" v-for="p in participants" :key="p">
              {{ p }}
              <span class="remove" @click="removeParticipant(p)">×</span>
            </span>
          </div>

          <div class="empty-state-mini" v-else>
            <p>Belum ada peserta</p>
          </div>
        </div>

        <!-- Pajak & Service Card -->
        <div class="card">
          <h3 class="card-title">
            <span class="icon">💵</span>
            Pajak & Service
          </h3>

          <div class="tax-service-grid">
            <div class="tax-service-item">
              <label class="form-label">Pajak</label>
              <div class="input-with-suffix">
                <input
                  type="number"
                  class="form-input"
                  v-model="taxPercent"
                  min="0"
                  max="100"
                  placeholder="10"
                />
                <span class="input-suffix">%</span>
              </div>
            </div>

            <div class="tax-service-item">
              <label class="form-label">Service</label>
              <div class="input-with-toggle">
                <input
                  type="number"
                  class="form-input"
                  v-model="serviceValue"
                  min="0"
                  :placeholder="serviceType === 'fixed' ? '10000' : '0'"
                />
                <div class="toggle-btns">
                  <button
                    class="toggle-btn"
                    :class="{ active: serviceType === 'percent' }"
                    @click="serviceType = 'percent'"
                  >
                    %
                  </button>
                  <button
                    class="toggle-btn"
                    :class="{ active: serviceType === 'fixed' }"
                    @click="serviceType = 'fixed'"
                  >
                    Rp
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </section>

    <!-- Step 2: Daftar Menu -->
    <section class="section">
      <div class="section-header">
        <span class="step-badge">2</span>
        <h2 class="section-title">Daftar Menu</h2>
      </div>

      <div class="card">
        <!-- Add Item Form -->
        <div class="form-row">
          <div class="form-group" style="flex: 2">
            <input
              type="text"
              class="form-input"
              v-model="newItem.name"
              placeholder="Nama menu..."
              @keyup="handleItemEnter"
            />
          </div>
          <div class="form-group" style="flex: 1">
            <input
              type="text"
              class="form-input"
              :value="displayPrice"
              @input="handlePriceInput"
              placeholder="Harga"
              @keyup="handleItemEnter"
            />
          </div>
          <div class="form-group" style="width: 70px">
            <input
              type="number"
              class="form-input"
              v-model="newItem.qty"
              min="1"
              placeholder="Qty"
              @keyup="handleItemEnter"
            />
          </div>
          <button
            class="btn btn-primary btn-icon"
            @click="addItem"
            title="Tambah Item"
          >
            +
          </button>
        </div>

        <!-- Item List with Inline Assignment -->
        <div class="menu-list" v-if="items.length > 0">
          <div class="menu-item" v-for="item in items" :key="item.id">
            <div class="menu-item-main">
              <div class="menu-item-info">
                <span class="menu-item-name">{{ item.name }}</span>
                <span class="menu-item-details">
                  x{{ item.qty }} • {{ formatCurrency(getItemTotal(item)) }}
                </span>
              </div>
              <button
                class="btn-remove"
                @click="removeItem(item.id)"
                title="Hapus"
              >
                ×
              </button>
            </div>

            <!-- Inline Assignment -->
            <div class="menu-item-assign" v-if="participants.length > 0">
              <span
                class="assign-tag"
                :class="{ active: item.assignedTo.includes(p) }"
                v-for="p in participants"
                :key="p"
                @click="toggleAssignment(item, p)"
              >
                {{ p }}
              </span>
            </div>
            <div class="menu-item-hint" v-else>
              <small>Tambahkan peserta terlebih dahulu</small>
            </div>
          </div>
        </div>

        <div class="empty-state-mini" v-else>
          <p>Belum ada menu. Tambahkan item yang dipesan!</p>
        </div>
      </div>
    </section>

    <!-- Step 3: Ringkasan -->
    <section class="section" v-if="participantTotals.length > 0">
      <div class="section-header">
        <span class="step-badge">3</span>
        <h2 class="section-title">Ringkasan Pembagian</h2>
      </div>

      <div class="result-card" ref="resultRef">
        <div class="result-date">{{ today }}</div>

        <div class="result-grid">
          <div
            class="result-person"
            v-for="p in participantTotals"
            :key="p.name"
          >
            <div class="result-person-header">
              <span class="result-person-name">{{ p.name }}</span>
              <span class="result-person-total">{{
                formatCurrency(p.total)
              }}</span>
            </div>
            <ul class="result-person-items">
              <li v-for="(item, i) in p.items" :key="i">
                <span>
                  {{ item.name }}
                  <small v-if="item.sharedWith > 1"
                    >(÷{{ item.sharedWith }})</small
                  >
                </span>
                <span>{{ formatCurrency(item.share) }}</span>
              </li>
              <li v-if="p.taxShare > 0" class="tax-service-line">
                <span>Pajak</span>
                <span>{{ formatCurrency(p.taxShare) }}</span>
              </li>
              <li v-if="p.serviceShare > 0" class="tax-service-line">
                <span>Service</span>
                <span>{{ formatCurrency(p.serviceShare) }}</span>
              </li>
            </ul>
          </div>
        </div>

        <div class="result-summary">
          <div class="summary-row">
            <span>Subtotal</span>
            <span>{{ formatCurrency(subtotal) }}</span>
          </div>
          <div class="summary-row">
            <span>Pajak ({{ taxPercent }}%)</span>
            <span>{{ formatCurrency(taxAmount) }}</span>
          </div>
          <div class="summary-row" v-if="serviceAmount > 0">
            <span
              >Service
              {{
                serviceType === "fixed" ? "(Fixed)" : `(${serviceValue}%)`
              }}</span
            >
            <span>{{ formatCurrency(serviceAmount) }}</span>
          </div>
          <div class="summary-row total">
            <span>Grand Total</span>
            <span>{{ formatCurrency(grandTotal) }}</span>
          </div>
        </div>
      </div>

      <div class="download-section">
        <button class="btn download-btn" @click="downloadAsImage">
          📥 Download Sebagai Gambar
        </button>
      </div>
    </section>
  </div>
</template>

<style scoped>
/* Component-specific styles if needed */
</style>
