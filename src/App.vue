<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from "vue";

let jsPDFCtor = null;
let html2canvasFn = null;

const getJsPDFConstructor = async () => {
  if (jsPDFCtor) return jsPDFCtor;

  const module = await import("jspdf");
  jsPDFCtor = module.jsPDF;
  return jsPDFCtor;
};

const getHtml2Canvas = async () => {
  if (html2canvasFn) return html2canvasFn;

  const module = await import("html2canvas");
  html2canvasFn = module.default;
  return html2canvasFn;
};

const STORAGE_KEYS = {
  autoSortMenu: "split-bill:auto-sort-menu",
  taxPercent: "split-bill:tax-percent",
  serviceType: "split-bill:service-type",
  serviceValue: "split-bill:service-value",
  taxServiceScope: "split-bill:tax-service-scope",
  roundingDistribution: "split-bill:rounding-distribution",
  roundingTargetName: "split-bill:rounding-target-name",
  autoSyncQty: "split-bill:auto-sync-qty",
  roundingEnabled: "split-bill:rounding-enabled",
  roundingMode: "split-bill:rounding-mode",
  roundingUnit: "split-bill:rounding-unit",
  draftState: "split-bill:draft-state",
  historyList: "split-bill:history-list",
};

const ROUNDING_MODES = ["nearest", "up", "down"];
const SERVICE_TYPES = ["percent", "fixed"];
const TAX_SERVICE_SCOPES = ["global", "per-item"];
const ROUNDING_DISTRIBUTIONS = ["highest", "equal", "selected"];
const HISTORY_LIMIT = 20;
const DEFAULT_SETTINGS = {
  taxPercent: 10,
  serviceType: "percent",
  serviceValue: 0,
  taxServiceScope: "global",
  autoSyncQty: true,
  roundingEnabled: false,
  roundingMode: "nearest",
  roundingDistribution: "highest",
  roundingTargetName: "",
  roundingUnit: 100,
  autoSortMenu: true,
};

let shouldSkipStoragePersist = false;

const getStoredBoolean = (key, fallbackValue) => {
  if (typeof window === "undefined") return fallbackValue;

  try {
    const rawValue = window.localStorage.getItem(key);
    if (rawValue === null) return fallbackValue;
    return rawValue === "true";
  } catch {
    return fallbackValue;
  }
};

const getStoredString = (key, fallbackValue) => {
  if (typeof window === "undefined") return fallbackValue;

  try {
    const rawValue = window.localStorage.getItem(key);
    return rawValue === null ? fallbackValue : rawValue;
  } catch {
    return fallbackValue;
  }
};

const getStoredNumber = (key, fallbackValue, minValue = -Infinity) => {
  if (typeof window === "undefined") return fallbackValue;

  try {
    const rawValue = window.localStorage.getItem(key);
    if (rawValue === null) return fallbackValue;

    const parsed = parseInt(rawValue, 10);
    if (Number.isNaN(parsed) || parsed < minValue) return fallbackValue;

    return parsed;
  } catch {
    return fallbackValue;
  }
};

const getStoredFloat = (
  key,
  fallbackValue,
  minValue = -Infinity,
  maxValue = Infinity,
) => {
  if (typeof window === "undefined") return fallbackValue;

  try {
    const rawValue = window.localStorage.getItem(key);
    if (rawValue === null) return fallbackValue;

    const parsed = parseFloat(rawValue);
    if (Number.isNaN(parsed) || parsed < minValue || parsed > maxValue) {
      return fallbackValue;
    }

    return parsed;
  } catch {
    return fallbackValue;
  }
};

const getStoredJson = (key, fallbackValue) => {
  if (typeof window === "undefined") return fallbackValue;

  try {
    const rawValue = window.localStorage.getItem(key);
    if (!rawValue) return fallbackValue;
    return JSON.parse(rawValue);
  } catch {
    return fallbackValue;
  }
};

const deepClone = (value) => {
  return JSON.parse(JSON.stringify(value));
};

const setStoredValue = (key, value) => {
  if (typeof window === "undefined") return;
  if (shouldSkipStoragePersist) return;

  try {
    window.localStorage.setItem(key, String(value));
  } catch {
    // Ignore localStorage write errors.
  }
};

const setStoredJson = (key, value) => {
  if (typeof window === "undefined") return;
  if (shouldSkipStoragePersist) return;

  try {
    window.localStorage.setItem(key, JSON.stringify(value));
  } catch {
    // Ignore localStorage write errors.
  }
};

const removeStoredValue = (key) => {
  if (typeof window === "undefined") return;

  try {
    window.localStorage.removeItem(key);
  } catch {
    // Ignore localStorage remove errors.
  }
};

// State
const items = ref([]);
const participants = ref([]);
const newItem = ref({ name: "", price: "", qty: 1 });
const displayPrice = ref("");
const newParticipant = ref("");
const taxPercent = ref(
  getStoredFloat(STORAGE_KEYS.taxPercent, DEFAULT_SETTINGS.taxPercent, 0, 100),
);
const serviceValue = ref(
  getStoredFloat(STORAGE_KEYS.serviceValue, DEFAULT_SETTINGS.serviceValue, 0),
);
const serviceType = ref(
  getStoredString(STORAGE_KEYS.serviceType, DEFAULT_SETTINGS.serviceType),
); // 'percent' or 'fixed'
const taxServiceScope = ref(
  getStoredString(
    STORAGE_KEYS.taxServiceScope,
    DEFAULT_SETTINGS.taxServiceScope,
  ),
);
const autoSyncQty = ref(
  getStoredBoolean(STORAGE_KEYS.autoSyncQty, DEFAULT_SETTINGS.autoSyncQty),
);
const roundingEnabled = ref(
  getStoredBoolean(
    STORAGE_KEYS.roundingEnabled,
    DEFAULT_SETTINGS.roundingEnabled,
  ),
);
const roundingMode = ref(
  getStoredString(STORAGE_KEYS.roundingMode, DEFAULT_SETTINGS.roundingMode),
); // 'nearest' | 'up' | 'down'
const roundingDistribution = ref(
  getStoredString(
    STORAGE_KEYS.roundingDistribution,
    DEFAULT_SETTINGS.roundingDistribution,
  ),
);
const roundingTargetName = ref(
  getStoredString(
    STORAGE_KEYS.roundingTargetName,
    DEFAULT_SETTINGS.roundingTargetName,
  ),
);
const roundingUnit = ref(
  getStoredNumber(STORAGE_KEYS.roundingUnit, DEFAULT_SETTINGS.roundingUnit, 1),
);
const selectedItemId = ref(null);
const participantSearch = ref("");
const autoSortMenu = ref(
  getStoredBoolean(STORAGE_KEYS.autoSortMenu, DEFAULT_SETTINGS.autoSortMenu),
);
const resultRef = ref(null);
const showGuide = ref(false);
const showResetConfirm = ref(false);
const historyEntries = ref(getStoredJson(STORAGE_KEYS.historyList, []));
const undoStack = ref([]);
const redoStack = ref([]);
const shareLink = ref("");

if (!ROUNDING_MODES.includes(roundingMode.value)) {
  roundingMode.value = "nearest";
}

if (!SERVICE_TYPES.includes(serviceType.value)) {
  serviceType.value = "percent";
}

if (!TAX_SERVICE_SCOPES.includes(taxServiceScope.value)) {
  taxServiceScope.value = DEFAULT_SETTINGS.taxServiceScope;
}

if (!ROUNDING_DISTRIBUTIONS.includes(roundingDistribution.value)) {
  roundingDistribution.value = DEFAULT_SETTINGS.roundingDistribution;
}

const normalizeItemSettings = (item) => {
  return {
    ...item,
    taxPercent:
      typeof item.taxPercent === "number" ? item.taxPercent : taxPercent.value,
    serviceType: SERVICE_TYPES.includes(item.serviceType)
      ? item.serviceType
      : serviceType.value,
    serviceValue:
      typeof item.serviceValue === "number"
        ? item.serviceValue
        : Number(serviceValue.value) || 0,
    assignments: { ...(item.assignments || {}) },
  };
};

const createStateSnapshot = () => {
  return {
    items: items.value.map((item) => normalizeItemSettings(item)),
    participants: [...participants.value],
    selectedItemId: selectedItemId.value,
    settings: {
      taxPercent: Number(taxPercent.value) || DEFAULT_SETTINGS.taxPercent,
      serviceType: serviceType.value,
      serviceValue: Number(serviceValue.value) || 0,
      taxServiceScope: taxServiceScope.value,
      autoSyncQty: autoSyncQty.value,
      roundingEnabled: roundingEnabled.value,
      roundingMode: roundingMode.value,
      roundingDistribution: roundingDistribution.value,
      roundingTargetName: roundingTargetName.value,
      roundingUnit: Number(roundingUnit.value) || DEFAULT_SETTINGS.roundingUnit,
      autoSortMenu: autoSortMenu.value,
    },
  };
};

const applyStateSnapshot = (snapshot) => {
  if (!snapshot) return;

  shouldSkipStoragePersist = true;

  items.value = (snapshot.items || []).map((item) =>
    normalizeItemSettings(deepClone(item)),
  );
  participants.value = [...(snapshot.participants || [])];
  selectedItemId.value = snapshot.selectedItemId || null;

  const settings = snapshot.settings || {};
  taxPercent.value = Number(settings.taxPercent ?? DEFAULT_SETTINGS.taxPercent);
  serviceType.value = SERVICE_TYPES.includes(settings.serviceType)
    ? settings.serviceType
    : DEFAULT_SETTINGS.serviceType;
  serviceValue.value = Number(
    settings.serviceValue ?? DEFAULT_SETTINGS.serviceValue,
  );
  taxServiceScope.value = TAX_SERVICE_SCOPES.includes(settings.taxServiceScope)
    ? settings.taxServiceScope
    : DEFAULT_SETTINGS.taxServiceScope;
  autoSyncQty.value = Boolean(
    settings.autoSyncQty ?? DEFAULT_SETTINGS.autoSyncQty,
  );
  roundingEnabled.value = Boolean(
    settings.roundingEnabled ?? DEFAULT_SETTINGS.roundingEnabled,
  );
  roundingMode.value = ROUNDING_MODES.includes(settings.roundingMode)
    ? settings.roundingMode
    : DEFAULT_SETTINGS.roundingMode;
  roundingDistribution.value = ROUNDING_DISTRIBUTIONS.includes(
    settings.roundingDistribution,
  )
    ? settings.roundingDistribution
    : DEFAULT_SETTINGS.roundingDistribution;
  roundingTargetName.value = settings.roundingTargetName || "";
  roundingUnit.value = Math.max(
    Number(settings.roundingUnit ?? DEFAULT_SETTINGS.roundingUnit) ||
      DEFAULT_SETTINGS.roundingUnit,
    1,
  );
  autoSortMenu.value = Boolean(
    settings.autoSortMenu ?? DEFAULT_SETTINGS.autoSortMenu,
  );

  shouldSkipStoragePersist = false;

  if (autoSyncQty.value) {
    items.value.forEach((item) => {
      item.qty = Math.max(
        Object.values(item.assignments || {}).reduce(
          (sum, qty) => sum + (parseInt(qty) || 0),
          0,
        ),
        1,
      );
    });
  }
};

const pushUndoSnapshot = () => {
  undoStack.value.push(createStateSnapshot());
  if (undoStack.value.length > 50) {
    undoStack.value.shift();
  }
  redoStack.value = [];
};

const undoLastAction = () => {
  if (undoStack.value.length === 0) return;
  const previous = undoStack.value.pop();
  redoStack.value.push(createStateSnapshot());
  applyStateSnapshot(previous);
};

const redoLastAction = () => {
  if (redoStack.value.length === 0) return;
  const next = redoStack.value.pop();
  undoStack.value.push(createStateSnapshot());
  applyStateSnapshot(next);
};

const persistDraftState = () => {
  setStoredJson(STORAGE_KEYS.draftState, createStateSnapshot());
};

const saveCurrentStateToHistory = () => {
  const entry = {
    id: Date.now(),
    label: new Date().toLocaleString("id-ID"),
    snapshot: createStateSnapshot(),
  };

  historyEntries.value = [entry, ...historyEntries.value].slice(
    0,
    HISTORY_LIMIT,
  );
  setStoredJson(STORAGE_KEYS.historyList, historyEntries.value);
};

const loadHistoryEntry = (entry) => {
  if (!entry?.snapshot) return;
  pushUndoSnapshot();
  applyStateSnapshot(entry.snapshot);
};

const exportShareLink = async () => {
  if (typeof window === "undefined") return;

  const encoded = btoa(
    unescape(encodeURIComponent(JSON.stringify(createStateSnapshot()))),
  );
  const url = `${window.location.origin}${window.location.pathname}?state=${encodeURIComponent(encoded)}`;
  shareLink.value = url;

  try {
    await navigator.clipboard.writeText(url);
    alert("Link state berhasil disalin.");
  } catch {
    alert("Gagal menyalin link. Link sudah ditampilkan di aplikasi.");
  }
};

const hydrateStateFromUrlOrDraft = () => {
  if (typeof window === "undefined") return;

  try {
    const params = new URLSearchParams(window.location.search);
    const encodedState = params.get("state");

    if (encodedState) {
      const decodedJson = decodeURIComponent(escape(atob(encodedState)));
      const snapshot = JSON.parse(decodedJson);
      applyStateSnapshot(snapshot);
      persistDraftState();
      params.delete("state");
      const nextUrl = `${window.location.pathname}${params.toString() ? `?${params.toString()}` : ""}`;
      window.history.replaceState({}, "", nextUrl);
      return;
    }
  } catch {
    // Ignore invalid shared state.
  }

  const draft = getStoredJson(STORAGE_KEYS.draftState, null);
  if (draft) {
    applyStateSnapshot(draft);
  }
};

const clearSelectedItemAssignments = () => {
  if (!selectedItemId.value) return;
  const target = items.value.find((item) => item.id === selectedItemId.value);
  if (!target) return;

  pushUndoSnapshot();
  target.assignments = {};
  if (autoSyncQty.value) {
    target.qty = 1;
  }
};

const zeroAllParticipantsForSelectedItem = () => {
  clearSelectedItemAssignments();
};

const applyResetAllStoredPreferences = () => {
  shouldSkipStoragePersist = true;

  taxPercent.value = DEFAULT_SETTINGS.taxPercent;
  serviceType.value = DEFAULT_SETTINGS.serviceType;
  serviceValue.value = DEFAULT_SETTINGS.serviceValue;
  taxServiceScope.value = DEFAULT_SETTINGS.taxServiceScope;
  autoSyncQty.value = DEFAULT_SETTINGS.autoSyncQty;
  roundingEnabled.value = DEFAULT_SETTINGS.roundingEnabled;
  roundingMode.value = DEFAULT_SETTINGS.roundingMode;
  roundingDistribution.value = DEFAULT_SETTINGS.roundingDistribution;
  roundingTargetName.value = DEFAULT_SETTINGS.roundingTargetName;
  roundingUnit.value = DEFAULT_SETTINGS.roundingUnit;
  autoSortMenu.value = DEFAULT_SETTINGS.autoSortMenu;

  shouldSkipStoragePersist = false;

  Object.values(STORAGE_KEYS).forEach((key) => removeStoredValue(key));

  if (autoSyncQty.value) {
    items.value.forEach((item) => syncItemQtyWithAssignments(item));
  }
};

const openResetConfirmationModal = () => {
  showResetConfirm.value = true;
};

const closeResetConfirmationModal = () => {
  showResetConfirm.value = false;
};

const confirmResetAllStoredPreferences = () => {
  applyResetAllStoredPreferences();
  closeResetConfirmationModal();
};

const handleWindowKeydown = (event) => {
  if (event.key === "Escape" && showResetConfirm.value) {
    closeResetConfirmationModal();
  }
};

onMounted(() => {
  if (typeof window === "undefined") return;
  window.addEventListener("keydown", handleWindowKeydown);
  hydrateStateFromUrlOrDraft();
});

onBeforeUnmount(() => {
  if (typeof window === "undefined") return;
  window.removeEventListener("keydown", handleWindowKeydown);
});

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
    pushUndoSnapshot();
    const item = {
      id: Date.now(),
      name: newItem.value.name,
      price: parseFloat(rawPrice),
      qty: parseInt(newItem.value.qty) || 1,
      assignments: {},
      taxPercent: Number(taxPercent.value) || DEFAULT_SETTINGS.taxPercent,
      serviceType: serviceType.value,
      serviceValue: Number(serviceValue.value) || 0,
    };
    items.value.push(item);
    selectedItemId.value = item.id;
    newItem.value = { name: "", price: "", qty: 1 };
    displayPrice.value = "";
  }
};

// Remove item
const removeItem = (id) => {
  pushUndoSnapshot();
  items.value = items.value.filter((item) => item.id !== id);
};

// Add participant
const addParticipant = () => {
  if (
    newParticipant.value.trim() &&
    !participants.value.includes(newParticipant.value.trim())
  ) {
    pushUndoSnapshot();
    participants.value.push(newParticipant.value.trim());
    newParticipant.value = "";
  }
};

// Remove participant
const removeParticipant = (name) => {
  pushUndoSnapshot();
  participants.value = participants.value.filter((p) => p !== name);
  // Remove from all item assignments
  items.value.forEach((item) => {
    if (item.assignments && item.assignments[name]) {
      delete item.assignments[name];
      syncItemQtyWithAssignments(item);
    }
  });
};

// Assignment helpers
const getAssignedQty = (item, participant) => {
  return item.assignments?.[participant] || 0;
};

const getTotalAssignedQty = (item) => {
  return Object.values(item.assignments || {}).reduce(
    (sum, qty) => sum + (parseInt(qty) || 0),
    0,
  );
};

const syncItemQtyWithAssignments = (item) => {
  if (!autoSyncQty.value) return;
  item.qty = Math.max(getTotalAssignedQty(item), 1);
};

const updateItemQty = (item, delta) => {
  if (autoSyncQty.value) return;

  const nextQty = item.qty + delta;
  if (nextQty < 1) return;

  const assignedQty = getTotalAssignedQty(item);
  if (nextQty < assignedQty) {
    alert(
      "Qty menu tidak boleh lebih kecil dari jumlah yang sudah dialokasikan.",
    );
    return;
  }

  pushUndoSnapshot();
  item.qty = nextQty;
};

const incrementParticipantQty = (item, participant) => {
  const assignedQty = getTotalAssignedQty(item);
  if (!autoSyncQty.value && assignedQty >= item.qty) return;

  pushUndoSnapshot();
  const currentQty = getAssignedQty(item, participant);
  item.assignments[participant] = currentQty + 1;
  syncItemQtyWithAssignments(item);
};

const decrementParticipantQty = (item, participant) => {
  const currentQty = getAssignedQty(item, participant);
  if (currentQty <= 0) return;

  pushUndoSnapshot();
  if (currentQty === 1) {
    delete item.assignments[participant];
    syncItemQtyWithAssignments(item);
    return;
  }

  item.assignments[participant] = currentQty - 1;
  syncItemQtyWithAssignments(item);
};

watch(autoSyncQty, (enabled) => {
  setStoredValue(STORAGE_KEYS.autoSyncQty, enabled);

  if (!enabled) return;
  items.value.forEach((item) => syncItemQtyWithAssignments(item));
});

watch(taxPercent, (value) => {
  const parsed = Number(value);
  if (!Number.isFinite(parsed)) {
    taxPercent.value = 10;
    return;
  }

  const clamped = Math.min(Math.max(parsed, 0), 100);
  if (clamped !== value) {
    taxPercent.value = clamped;
    return;
  }

  setStoredValue(STORAGE_KEYS.taxPercent, clamped);
});

watch(serviceType, (value) => {
  if (!SERVICE_TYPES.includes(value)) {
    serviceType.value = "percent";
    return;
  }

  setStoredValue(STORAGE_KEYS.serviceType, value);
});

watch(serviceValue, (value) => {
  const parsed = Number(value);
  if (!Number.isFinite(parsed) || parsed < 0) {
    serviceValue.value = 0;
    return;
  }

  if (parsed !== value) {
    serviceValue.value = parsed;
    return;
  }

  setStoredValue(STORAGE_KEYS.serviceValue, parsed);
});

watch(autoSortMenu, (enabled) => {
  setStoredValue(STORAGE_KEYS.autoSortMenu, enabled);
});

watch(roundingEnabled, (enabled) => {
  setStoredValue(STORAGE_KEYS.roundingEnabled, enabled);
});

watch(roundingMode, (mode) => {
  if (!ROUNDING_MODES.includes(mode)) {
    roundingMode.value = "nearest";
    return;
  }

  setStoredValue(STORAGE_KEYS.roundingMode, mode);
});

watch(roundingUnit, (value) => {
  const parsed = parseInt(value, 10);
  if (Number.isNaN(parsed) || parsed < 1) {
    roundingUnit.value = 100;
    return;
  }

  if (parsed !== value) {
    roundingUnit.value = parsed;
    return;
  }

  setStoredValue(STORAGE_KEYS.roundingUnit, parsed);
});

watch(taxServiceScope, (scope) => {
  if (!TAX_SERVICE_SCOPES.includes(scope)) {
    taxServiceScope.value = DEFAULT_SETTINGS.taxServiceScope;
    return;
  }

  setStoredValue(STORAGE_KEYS.taxServiceScope, scope);
});

watch(roundingDistribution, (distribution) => {
  if (!ROUNDING_DISTRIBUTIONS.includes(distribution)) {
    roundingDistribution.value = DEFAULT_SETTINGS.roundingDistribution;
    return;
  }

  setStoredValue(STORAGE_KEYS.roundingDistribution, distribution);
});

watch(roundingTargetName, (name) => {
  setStoredValue(STORAGE_KEYS.roundingTargetName, name || "");
});

watch(
  [
    items,
    participants,
    selectedItemId,
    taxPercent,
    serviceType,
    serviceValue,
    taxServiceScope,
    autoSyncQty,
    roundingEnabled,
    roundingMode,
    roundingDistribution,
    roundingTargetName,
    roundingUnit,
    autoSortMenu,
  ],
  () => {
    persistDraftState();
  },
  { deep: true },
);

watch(
  items,
  (nextItems) => {
    if (nextItems.length === 0) {
      selectedItemId.value = null;
      return;
    }

    const isSelectedStillExist = nextItems.some(
      (item) => item.id === selectedItemId.value,
    );

    if (!isSelectedStillExist) {
      selectedItemId.value = nextItems[0].id;
    }
  },
  { deep: true },
);

watch(participants, (nextParticipants) => {
  if (!roundingTargetName.value) return;
  if (!nextParticipants.includes(roundingTargetName.value)) {
    roundingTargetName.value = "";
  }
});

const selectedItem = computed(() => {
  if (!selectedItemId.value) return null;
  return items.value.find((item) => item.id === selectedItemId.value) || null;
});

const filteredParticipants = computed(() => {
  const keyword = participantSearch.value.trim().toLowerCase();
  if (!keyword) return participants.value;

  return participants.value.filter((name) =>
    name.toLowerCase().includes(keyword),
  );
});

const getAllocationStatus = (item) => {
  const allocatedQty = getTotalAssignedQty(item);

  if (allocatedQty > item.qty) {
    return { label: "Over", type: "over" };
  }

  if (allocatedQty === item.qty) {
    return { label: "Pas", type: "fit" };
  }

  return { label: "Belum penuh", type: "under" };
};

const getAllocationRank = (item) => {
  const allocatedQty = getTotalAssignedQty(item);

  if (allocatedQty > item.qty) {
    return 0;
  }

  if (allocatedQty < item.qty) {
    return 1;
  }

  return 2;
};

const getAllocationSeverity = (item) => {
  return Math.abs(getTotalAssignedQty(item) - item.qty);
};

const sortedMenuItems = computed(() => {
  return [...items.value].sort((a, b) => {
    const rankA = getAllocationRank(a);
    const rankB = getAllocationRank(b);

    if (rankA !== rankB) {
      return rankA - rankB;
    }

    const severityA = getAllocationSeverity(a);
    const severityB = getAllocationSeverity(b);

    if (severityA !== severityB) {
      return severityB - severityA;
    }

    return a.name.localeCompare(b.name, "id-ID");
  });
});

const displayedMenuItems = computed(() => {
  return autoSortMenu.value ? sortedMenuItems.value : items.value;
});

// Calculate subtotal per item
const getItemTotal = (item) => {
  return item.price * item.qty;
};

// Calculate subtotal (before tax & service)
const subtotal = computed(() => {
  return items.value.reduce((sum, item) => sum + getItemTotal(item), 0);
});

const getItemTaxCharge = (item) => {
  const itemTaxPercent =
    taxServiceScope.value === "per-item"
      ? Number(
          item.taxPercent ?? taxPercent.value ?? DEFAULT_SETTINGS.taxPercent,
        ) || 0
      : Number(taxPercent.value) || 0;

  return getItemTotal(item) * (itemTaxPercent / 100);
};

const getItemServiceCharge = (item) => {
  const itemServiceType =
    taxServiceScope.value === "per-item"
      ? item.serviceType || serviceType.value
      : serviceType.value;
  const itemServiceValue =
    taxServiceScope.value === "per-item"
      ? Number(item.serviceValue ?? serviceValue.value ?? 0) || 0
      : Number(serviceValue.value) || 0;

  if (itemServiceType === "fixed") {
    return itemServiceValue;
  }

  return getItemTotal(item) * (itemServiceValue / 100);
};

// Calculate tax amount
const taxAmount = computed(() => {
  if (taxServiceScope.value === "global") {
    return subtotal.value * ((Number(taxPercent.value) || 0) / 100);
  }

  return items.value.reduce((sum, item) => sum + getItemTaxCharge(item), 0);
});

// Calculate service charge
const serviceAmount = computed(() => {
  if (taxServiceScope.value === "global") {
    if (serviceType.value === "fixed") {
      return parseFloat(serviceValue.value) || 0;
    }
    return subtotal.value * ((Number(serviceValue.value) || 0) / 100);
  }

  return items.value.reduce((sum, item) => sum + getItemServiceCharge(item), 0);
});

// Calculate grand total
const grandTotalRaw = computed(() => {
  return subtotal.value + taxAmount.value + serviceAmount.value;
});

const roundedGrandTotal = computed(() => {
  if (!roundingEnabled.value) return grandTotalRaw.value;

  const unit = parseInt(roundingUnit.value) || 0;
  if (unit <= 0) return grandTotalRaw.value;

  if (roundingMode.value === "up") {
    return Math.ceil(grandTotalRaw.value / unit) * unit;
  }

  if (roundingMode.value === "down") {
    return Math.floor(grandTotalRaw.value / unit) * unit;
  }

  return Math.round(grandTotalRaw.value / unit) * unit;
});

const roundingAdjustment = computed(() => {
  return roundedGrandTotal.value - grandTotalRaw.value;
});

const grandTotal = computed(() => {
  return roundedGrandTotal.value;
});

// Get participants who have items assigned (for fixed service split)
const activeParticipants = computed(() => {
  const active = new Set();
  items.value.forEach((item) => {
    Object.entries(item.assignments || {}).forEach(([name, qty]) => {
      if ((parseInt(qty) || 0) > 0) {
        active.add(name);
      }
    });
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
      roundingShare: 0,
      total: 0,
    };
  });

  // Calculate items per person
  items.value.forEach((item) => {
    Object.entries(item.assignments || {}).forEach(([name, qty]) => {
      const assignedQty = parseInt(qty) || 0;
      if (!totals[name] || assignedQty <= 0) return;

      const share = item.price * assignedQty;
      totals[name].items.push({
        name: item.name,
        qty: assignedQty,
        share,
      });
      totals[name].itemsTotal += share;
    });
  });

  // Calculate tax & service share
  const totalItemsAssigned = Object.values(totals).reduce(
    (sum, p) => sum + p.itemsTotal,
    0,
  );

  if (taxServiceScope.value === "global") {
    if (totalItemsAssigned > 0) {
      const numActiveParticipants = activeParticipants.value.length;

      Object.keys(totals).forEach((p) => {
        const proportion = totals[p].itemsTotal / totalItemsAssigned;
        const hasItems = totals[p].itemsTotal > 0;

        totals[p].taxShare = taxAmount.value * proportion;

        if (
          serviceType.value === "fixed" &&
          hasItems &&
          numActiveParticipants > 0
        ) {
          totals[p].serviceShare = serviceAmount.value / numActiveParticipants;
        } else {
          totals[p].serviceShare = serviceAmount.value * proportion;
        }
      });
    }
  } else {
    items.value.forEach((item) => {
      const assignedEntries = Object.entries(item.assignments || {})
        .map(([name, qty]) => ({ name, qty: parseInt(qty) || 0 }))
        .filter((entry) => entry.qty > 0 && totals[entry.name]);

      if (assignedEntries.length === 0) return;

      const itemAssignedTotal = assignedEntries.reduce(
        (sum, entry) => sum + item.price * entry.qty,
        0,
      );

      if (itemAssignedTotal <= 0) return;

      const itemTaxCharge = getItemTaxCharge(item);
      const itemServiceCharge = getItemServiceCharge(item);
      const itemServiceType = item.serviceType || serviceType.value;

      assignedEntries.forEach((entry) => {
        const personShare = item.price * entry.qty;
        const proportion = personShare / itemAssignedTotal;

        totals[entry.name].taxShare += itemTaxCharge * proportion;

        if (itemServiceType === "fixed") {
          totals[entry.name].serviceShare +=
            itemServiceCharge / assignedEntries.length;
        } else {
          totals[entry.name].serviceShare += itemServiceCharge * proportion;
        }
      });
    });
  }

  Object.keys(totals).forEach((p) => {
    totals[p].total =
      totals[p].itemsTotal + totals[p].taxShare + totals[p].serviceShare;
  });

  const activeTotals = Object.values(totals).filter((p) => p.total > 0);

  if (activeTotals.length > 0 && roundingAdjustment.value !== 0) {
    if (roundingDistribution.value === "equal") {
      const share = roundingAdjustment.value / activeTotals.length;
      activeTotals.forEach((entry) => {
        entry.roundingShare += share;
        entry.total += share;
      });
    } else {
      let targetEntry = activeTotals[0];

      if (roundingDistribution.value === "selected") {
        const selectedTarget = activeTotals.find(
          (entry) => entry.name === roundingTargetName.value,
        );
        if (selectedTarget) {
          targetEntry = selectedTarget;
        }
      } else {
        activeTotals.forEach((entry) => {
          if (entry.total > targetEntry.total) {
            targetEntry = entry;
          }
        });
      }

      targetEntry.roundingShare += roundingAdjustment.value;
      targetEntry.total += roundingAdjustment.value;
    }
  }

  return activeTotals;
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

const hasAllocationIssues = computed(() => {
  return items.value.some((item) => getAllocationStatus(item).type !== "fit");
});

const ensureReadyForExport = () => {
  if (!hasAllocationIssues.value) return true;

  if (typeof window === "undefined") return true;
  return window.confirm(
    "Masih ada item yang statusnya belum Pas (Belum penuh/Over). Lanjutkan export?",
  );
};

// Download as image
const downloadAsImage = async () => {
  if (!resultRef.value) return;
  if (!ensureReadyForExport()) return;

  try {
    const html2canvas = await getHtml2Canvas();
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

const downloadAsCSV = () => {
  if (!ensureReadyForExport()) return;

  const header = ["Peserta", "Item", "Qty", "Nominal"];
  const lines = [header.join(",")];

  participantTotals.value.forEach((person) => {
    person.items.forEach((item) => {
      lines.push(
        [person.name, item.name, item.qty, Math.round(item.share)]
          .map((cell) => `"${String(cell).replace(/"/g, '""')}"`)
          .join(","),
      );
    });

    if (person.taxShare) {
      lines.push(
        `"${person.name}","Pajak","","${Math.round(person.taxShare)}"`,
      );
    }
    if (person.serviceShare) {
      lines.push(
        `"${person.name}","Service","","${Math.round(person.serviceShare)}"`,
      );
    }
    if (person.roundingShare) {
      lines.push(
        `"${person.name}","Pembulatan","","${Math.round(person.roundingShare)}"`,
      );
    }

    lines.push(`"${person.name}","TOTAL","","${Math.round(person.total)}"`);
  });

  const blob = new Blob([lines.join("\n")], {
    type: "text/csv;charset=utf-8;",
  });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = `split-bill-${new Date().toISOString().split("T")[0]}.csv`;
  link.click();
  URL.revokeObjectURL(link.href);
};

const downloadAsPDF = async () => {
  if (!ensureReadyForExport()) return;

  const JsPDF = await getJsPDFConstructor();

  const doc = new JsPDF({ unit: "pt", format: "a4" });
  let cursorY = 44;

  doc.setFontSize(16);
  doc.text("Split Bill", 40, cursorY);
  cursorY += 16;
  doc.setFontSize(10);
  doc.text(today, 40, cursorY);
  cursorY += 18;

  participantTotals.value.forEach((person) => {
    if (cursorY > 760) {
      doc.addPage();
      cursorY = 44;
    }

    doc.setFontSize(12);
    doc.text(`${person.name} - ${formatCurrency(person.total)}`, 40, cursorY);
    cursorY += 14;

    doc.setFontSize(10);
    person.items.forEach((item) => {
      doc.text(
        `• ${item.name} x${item.qty}: ${formatCurrency(item.share)}`,
        52,
        cursorY,
      );
      cursorY += 12;
    });

    if (person.taxShare > 0) {
      doc.text(`Pajak: ${formatCurrency(person.taxShare)}`, 52, cursorY);
      cursorY += 12;
    }
    if (person.serviceShare > 0) {
      doc.text(`Service: ${formatCurrency(person.serviceShare)}`, 52, cursorY);
      cursorY += 12;
    }
    if (person.roundingShare !== 0) {
      doc.text(
        `Pembulatan: ${formatCurrency(person.roundingShare)}`,
        52,
        cursorY,
      );
      cursorY += 12;
    }

    cursorY += 8;
  });

  cursorY += 6;
  doc.setFontSize(11);
  doc.text(`Subtotal: ${formatCurrency(subtotal.value)}`, 40, cursorY);
  cursorY += 14;
  doc.text(`Pajak: ${formatCurrency(taxAmount.value)}`, 40, cursorY);
  cursorY += 14;
  doc.text(`Service: ${formatCurrency(serviceAmount.value)}`, 40, cursorY);
  cursorY += 14;
  if (roundingAdjustment.value !== 0) {
    doc.text(
      `Pembulatan: ${formatCurrency(roundingAdjustment.value)}`,
      40,
      cursorY,
    );
    cursorY += 14;
  }
  doc.setFontSize(13);
  doc.text(`Grand Total: ${formatCurrency(grandTotal.value)}`, 40, cursorY);

  doc.save(`split-bill-${new Date().toISOString().split("T")[0]}.pdf`);
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
                Rp), atur auto-sync qty, dan opsi pembulatan total.
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
                Pilih menu di panel kiri, lalu atur jumlah pesanan tiap peserta
                di panel kanan.
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

    <div
      class="modal-overlay"
      v-if="showResetConfirm"
      @click.self="closeResetConfirmationModal"
    >
      <div class="modal-content modal-sm">
        <div class="modal-header">
          <h2>⚠️ Konfirmasi Reset</h2>
          <button class="modal-close" @click="closeResetConfirmationModal">
            ×
          </button>
        </div>
        <div class="modal-body">
          <p class="confirm-text">
            Semua pengaturan tersimpan akan dikembalikan ke default. Lanjutkan?
          </p>
          <div class="modal-actions">
            <button class="btn btn-muted" @click="closeResetConfirmationModal">
              Batal
            </button>
            <button
              class="btn btn-danger"
              @click="confirmResetAllStoredPreferences"
            >
              Ya, Reset
            </button>
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

            <div class="tax-service-item span-2">
              <div class="settings-row settings-row-wrap">
                <label class="form-label" style="margin-bottom: 0"
                  >Scope Pajak & Service</label
                >
                <div class="toggle-btns">
                  <button
                    class="toggle-btn"
                    :class="{ active: taxServiceScope === 'global' }"
                    @click="taxServiceScope = 'global'"
                  >
                    Global
                  </button>
                  <button
                    class="toggle-btn"
                    :class="{ active: taxServiceScope === 'per-item' }"
                    @click="taxServiceScope = 'per-item'"
                  >
                    Per Item
                  </button>
                </div>
              </div>

              <div class="settings-row">
                <label class="checkbox-label">
                  <input type="checkbox" v-model="autoSyncQty" />
                  <span>Auto-sync Qty Menu ke Assignment Peserta</span>
                </label>
              </div>

              <div class="settings-row">
                <label class="checkbox-label">
                  <input type="checkbox" v-model="roundingEnabled" />
                  <span>Aktifkan Rounding Amount</span>
                </label>
              </div>

              <div class="rounding-controls" v-if="roundingEnabled">
                <div class="form-group">
                  <label class="form-label">Mode Rounding</label>
                  <select class="form-input" v-model="roundingMode">
                    <option value="nearest">Terdekat</option>
                    <option value="up">Ke Atas</option>
                    <option value="down">Ke Bawah</option>
                  </select>
                </div>

                <div class="form-group">
                  <label class="form-label">Satuan (Rp)</label>
                  <input
                    type="number"
                    class="form-input"
                    v-model="roundingUnit"
                    min="1"
                    step="100"
                    placeholder="100"
                  />
                </div>

                <div class="form-group">
                  <label class="form-label">Distribusi Pembulatan</label>
                  <select class="form-input" v-model="roundingDistribution">
                    <option value="highest">Pembayar Terbesar</option>
                    <option value="equal">Rata Semua Peserta</option>
                    <option value="selected">Peserta Tertentu</option>
                  </select>
                </div>

                <div
                  class="form-group"
                  v-if="roundingDistribution === 'selected'"
                >
                  <label class="form-label">Peserta Target</label>
                  <select class="form-input" v-model="roundingTargetName">
                    <option value="">Pilih Peserta</option>
                    <option v-for="p in participants" :key="p" :value="p">
                      {{ p }}
                    </option>
                  </select>
                </div>
              </div>

              <div class="settings-row settings-row-end">
                <button
                  class="reset-pref-btn"
                  @click="openResetConfirmationModal"
                  title="Reset semua preferensi tersimpan"
                >
                  Reset Semua Pengaturan
                </button>
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

        <div class="section-actions-row">
          <button
            class="btn btn-muted"
            @click="undoLastAction"
            :disabled="undoStack.length === 0"
          >
            Undo
          </button>
          <button
            class="btn btn-muted"
            @click="redoLastAction"
            :disabled="redoStack.length === 0"
          >
            Redo
          </button>
        </div>

        <div class="menu-workspace" v-if="items.length > 0">
          <div class="menu-master">
            <div class="menu-panel-header">
              <div class="menu-panel-toprow">
                <h4>Daftar Menu</h4>
                <label class="mini-toggle">
                  <input type="checkbox" v-model="autoSortMenu" />
                  <span>Urutkan Otomatis</span>
                </label>
              </div>
              <small>
                Pilih menu untuk atur assignment
                {{ autoSortMenu ? "(Prioritas)" : "(Urutan Input)" }}
              </small>
            </div>

            <div class="menu-master-list">
              <button
                class="menu-master-item"
                :class="{ active: selectedItemId === item.id }"
                v-for="item in displayedMenuItems"
                :key="item.id"
                @click="selectedItemId = item.id"
              >
                <div class="menu-master-top">
                  <span class="menu-master-name">{{ item.name }}</span>
                  <span class="menu-master-total">{{
                    formatCurrency(getItemTotal(item))
                  }}</span>
                </div>
                <div class="menu-master-status">
                  <span
                    class="status-badge"
                    :class="`status-${getAllocationStatus(item).type}`"
                  >
                    {{ getAllocationStatus(item).label }}
                  </span>
                </div>
                <div class="menu-master-meta">
                  <small>Qty {{ item.qty }}</small>
                  <small
                    >Alokasi {{ getTotalAssignedQty(item) }}/{{
                      item.qty
                    }}</small
                  >
                </div>
              </button>
            </div>
          </div>

          <div class="menu-detail" v-if="selectedItem">
            <div class="menu-panel-header">
              <h4>{{ selectedItem.name }}</h4>
              <small>
                Dialokasikan {{ getTotalAssignedQty(selectedItem) }}/{{
                  selectedItem.qty
                }}
              </small>
            </div>

            <div class="detail-quick-actions">
              <button
                class="btn btn-muted"
                @click="clearSelectedItemAssignments"
              >
                Reset Assignment Item
              </button>
              <button
                class="btn btn-muted"
                @click="zeroAllParticipantsForSelectedItem"
              >
                Set Qty Peserta 0
              </button>
            </div>

            <div
              class="item-scope-settings"
              v-if="taxServiceScope === 'per-item'"
            >
              <div class="form-group">
                <label class="form-label">Pajak Item (%)</label>
                <input
                  type="number"
                  class="form-input"
                  v-model="selectedItem.taxPercent"
                  min="0"
                  max="100"
                />
              </div>
              <div class="form-group">
                <label class="form-label">Service Item</label>
                <div class="input-with-toggle">
                  <input
                    type="number"
                    class="form-input"
                    v-model="selectedItem.serviceValue"
                    min="0"
                  />
                  <div class="toggle-btns">
                    <button
                      class="toggle-btn"
                      :class="{
                        active: selectedItem.serviceType === 'percent',
                      }"
                      @click="selectedItem.serviceType = 'percent'"
                    >
                      %
                    </button>
                    <button
                      class="toggle-btn"
                      :class="{ active: selectedItem.serviceType === 'fixed' }"
                      @click="selectedItem.serviceType = 'fixed'"
                    >
                      Rp
                    </button>
                  </div>
                </div>
              </div>
            </div>

            <div class="menu-item-main menu-detail-action-row">
              <div class="inline-qty-control">
                <button
                  class="qty-btn"
                  @click="updateItemQty(selectedItem, -1)"
                  :disabled="autoSyncQty || selectedItem.qty <= 1"
                  title="Kurangi qty menu"
                >
                  -
                </button>
                <span class="qty-value">{{ selectedItem.qty }}</span>
                <button
                  class="qty-btn"
                  @click="updateItemQty(selectedItem, 1)"
                  :disabled="autoSyncQty"
                  title="Tambah qty menu"
                >
                  +
                </button>
              </div>

              <button
                class="btn-remove"
                @click="removeItem(selectedItem.id)"
                title="Hapus menu"
              >
                ×
              </button>
            </div>

            <div class="participant-search" v-if="participants.length > 0">
              <input
                type="text"
                class="form-input"
                v-model="participantSearch"
                placeholder="Cari peserta..."
              />
            </div>

            <div class="menu-detail-list" v-if="participants.length > 0">
              <div
                class="assign-row"
                v-for="p in filteredParticipants"
                :key="p"
              >
                <span class="assign-name">{{ p }}</span>
                <div class="assign-qty-control">
                  <button
                    class="qty-btn"
                    @click="decrementParticipantQty(selectedItem, p)"
                    :disabled="getAssignedQty(selectedItem, p) === 0"
                    title="Kurangi qty peserta"
                  >
                    -
                  </button>
                  <span class="qty-value">{{
                    getAssignedQty(selectedItem, p)
                  }}</span>
                  <button
                    class="qty-btn"
                    @click="incrementParticipantQty(selectedItem, p)"
                    :disabled="
                      !autoSyncQty &&
                      getTotalAssignedQty(selectedItem) >= selectedItem.qty
                    "
                    title="Tambah qty peserta"
                  >
                    +
                  </button>
                </div>
              </div>

              <div
                class="empty-state-mini"
                v-if="filteredParticipants.length === 0"
              >
                <p>Peserta tidak ditemukan</p>
              </div>
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
                  {{ item.name }} <small>x{{ item.qty }}</small>
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
              <li v-if="p.roundingShare !== 0" class="tax-service-line">
                <span>Pembulatan</span>
                <span>{{ formatCurrency(p.roundingShare) }}</span>
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
            <span>
              Pajak
              {{
                taxServiceScope === "global" ? `(${taxPercent}%)` : "(Per Item)"
              }}
            </span>
            <span>{{ formatCurrency(taxAmount) }}</span>
          </div>
          <div class="summary-row" v-if="serviceAmount > 0">
            <span
              >Service
              {{
                taxServiceScope === "global"
                  ? serviceType === "fixed"
                    ? "(Fixed)"
                    : `(${serviceValue}%)`
                  : "(Per Item)"
              }}</span
            >
            <span>{{ formatCurrency(serviceAmount) }}</span>
          </div>
          <div class="summary-row" v-if="roundingAdjustment !== 0">
            <span>Pembulatan</span>
            <span>{{ formatCurrency(roundingAdjustment) }}</span>
          </div>
          <div class="summary-row total">
            <span>Grand Total</span>
            <span>{{ formatCurrency(grandTotal) }}</span>
          </div>
        </div>
      </div>

      <div class="download-section">
        <div class="download-actions-grid">
          <button class="btn download-btn" @click="downloadAsImage">
            📥 Gambar
          </button>
          <button class="btn btn-muted" @click="downloadAsPDF">📄 PDF</button>
          <button class="btn btn-muted" @click="downloadAsCSV">🧾 CSV</button>
          <button class="btn btn-muted" @click="saveCurrentStateToHistory">
            💾 Simpan Riwayat
          </button>
          <button class="btn btn-muted" @click="exportShareLink">
            🔗 Share Link
          </button>
        </div>
        <p class="share-link" v-if="shareLink">
          {{ shareLink }}
        </p>
      </div>

      <div class="card history-card" v-if="historyEntries.length > 0">
        <h3 class="card-title">
          <span class="icon">🕘</span>
          Riwayat Transaksi
        </h3>
        <div class="history-list">
          <div
            class="history-row"
            v-for="entry in historyEntries"
            :key="entry.id"
          >
            <span>{{ entry.label }}</span>
            <button class="btn btn-muted" @click="loadHistoryEntry(entry)">
              Buka
            </button>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<style scoped>
/* Component-specific styles if needed */
</style>
