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
      <p>Hitung dan bagikan tagihan dengan mudah!</p>
    </header>

    <div class="grid-2">
      <!-- Left Column: Items & Tax -->
      <div>
        <!-- Add Items Card -->
        <div class="card">
          <h2 class="card-title">
            <span class="icon">🍽️</span>
            Daftar Menu
          </h2>

          <div class="form-row">
            <div class="form-group" style="flex: 2">
              <label class="form-label">Nama Item</label>
              <input
                type="text"
                class="form-input"
                v-model="newItem.name"
                placeholder="Contoh: Nasi Goreng"
                @keyup="handleItemEnter"
              />
            </div>
            <div class="form-group">
              <label class="form-label">Harga</label>
              <input
                type="text"
                class="form-input"
                :value="displayPrice"
                @input="handlePriceInput"
                placeholder="Contoh: 25.000"
                @keyup="handleItemEnter"
              />
            </div>
            <div class="form-group" style="width: 80px">
              <label class="form-label">Qty</label>
              <input
                type="number"
                class="form-input"
                v-model="newItem.qty"
                min="1"
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

          <!-- Item List -->
          <div class="item-list mt-3" v-if="items.length > 0">
            <div class="item-row" v-for="item in items" :key="item.id">
              <span class="item-name">{{ item.name }}</span>
              <span class="item-qty">x{{ item.qty }}</span>
              <span class="item-price">{{
                formatCurrency(getItemTotal(item))
              }}</span>
              <button
                class="btn btn-danger btn-sm btn-icon"
                @click="removeItem(item.id)"
              >
                ×
              </button>
            </div>
          </div>

          <div class="empty-state" v-else>
            <div class="icon">📝</div>
            <p>Belum ada item. Tambahkan menu yang dipesan!</p>
          </div>
        </div>

        <!-- Tax & Service Card -->
        <div class="card">
          <h2 class="card-title">
            <span class="icon">💵</span>
            Pajak & Service
          </h2>

          <div class="tax-service-grid">
            <!-- Pajak Input -->
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

            <!-- Service Input -->
            <div class="tax-service-item">
              <label class="form-label">Service</label>
              <div class="input-with-toggle">
                <input
                  type="number"
                  class="form-input"
                  v-model="serviceValue"
                  min="0"
                  :placeholder="serviceType === 'fixed' ? '10000' : '5'"
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
              <small class="form-hint" v-if="serviceType === 'fixed'">
                Dibagi rata ke semua peserta
              </small>
            </div>
          </div>

          <div class="summary-row mt-2">
            <span>Subtotal</span>
            <span>{{ formatCurrency(subtotal) }}</span>
          </div>
          <div class="summary-row">
            <span>Pajak ({{ taxPercent }}%)</span>
            <span>{{ formatCurrency(taxAmount) }}</span>
          </div>
          <div class="summary-row">
            <span
              >Service
              {{
                serviceType === "fixed" ? "(Fixed)" : `(${serviceValue}%)`
              }}</span
            >
            <span>{{ formatCurrency(serviceAmount) }}</span>
          </div>
          <div class="summary-row total">
            <span>Total</span>
            <span style="color: var(--secondary-light)">{{
              formatCurrency(grandTotal)
            }}</span>
          </div>
        </div>
      </div>

      <!-- Right Column: Participants & Assignment -->
      <div>
        <!-- Participants Card -->
        <div class="card">
          <h2 class="card-title">
            <span class="icon">👥</span>
            Peserta
          </h2>

          <div class="form-row">
            <div class="form-group">
              <label class="form-label">Nama Peserta</label>
              <input
                type="text"
                class="form-input"
                v-model="newParticipant"
                placeholder="Nama peserta..."
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

          <div class="empty-state" v-else>
            <div class="icon">👤</div>
            <p>Tambahkan nama peserta yang ikut patungan</p>
          </div>
        </div>

        <!-- Assignment Card -->
        <div class="card" v-if="items.length > 0 && participants.length > 0">
          <h2 class="card-title">
            <span class="icon">✓</span>
            Siapa Pesan Apa?
          </h2>

          <div class="item-list">
            <div
              class="item-row"
              v-for="item in items"
              :key="item.id"
              style="flex-direction: column; align-items: stretch"
            >
              <div class="flex justify-between items-center mb-2">
                <span class="item-name">{{ item.name }}</span>
                <span class="item-price">{{
                  formatCurrency(getItemTotal(item))
                }}</span>
              </div>
              <div class="participant-tags">
                <span
                  class="participant-tag"
                  :class="{ inactive: !item.assignedTo.includes(p) }"
                  v-for="p in participants"
                  :key="p"
                  @click="toggleAssignment(item, p)"
                >
                  {{ p }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Results Section -->
    <div v-if="participantTotals.length > 0">
      <div class="result-card" ref="resultRef">
        <div class="result-header">
          <h2>📊 Ringkasan Pembagian</h2>
          <p class="date">{{ today }}</p>
        </div>

        <div class="result-items">
          <div
            class="result-person"
            v-for="p in participantTotals"
            :key="p.name"
          >
            <div class="result-person-header">
              <span class="result-person-name">👤 {{ p.name }}</span>
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
              <li v-if="p.taxShare > 0">
                <span>Pajak</span>
                <span>{{ formatCurrency(p.taxShare) }}</span>
              </li>
              <li v-if="p.serviceShare > 0">
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
            <span>Pajak + Service</span>
            <span>{{ formatCurrency(taxAmount + serviceAmount) }}</span>
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
    </div>
  </div>
</template>

<style scoped>
/* Component-specific styles if needed */
</style>
