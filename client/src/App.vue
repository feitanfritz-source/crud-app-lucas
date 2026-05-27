<script setup>
import { ref, onMounted, computed } from 'vue'

const API = 'http://localhost:4000/api/items'

const items = ref([])
const loading = ref(false)
const searchQuery = ref('')

const form = ref({
  name: '',
  description: '',
  price: ''
})

const editId = ref(null)
const deleteConfirmId = ref(null)
const toast = ref(null)

const filteredItems = computed(() => {
  if (!searchQuery.value.trim()) return items.value
  const q = searchQuery.value.toLowerCase()
  return items.value.filter(i =>
    i.name.toLowerCase().includes(q) ||
    (i.description || '').toLowerCase().includes(q)
  )
})

const totalValue = computed(() =>
  items.value.reduce((sum, item) => sum + (Number(item.price) || 0), 0)
)

const avgPrice = computed(() =>
  items.value.length ? totalValue.value / items.value.length : 0
)

const highestItem = computed(() =>
  items.value.reduce((max, item) =>
    Number(item.price) > Number(max?.price || 0) ? item : max, null)
)

function showToast(message, type = 'success') {
  toast.value = { message, type }
  setTimeout(() => toast.value = null, 3000)
}

async function load() {
  loading.value = true
  try {
    items.value = await fetch(API).then(r => r.json())
  } catch {
    showToast('Failed to connect to server.', 'error')
  }
  loading.value = false
}

async function save() {
  const payload = {
    ...form.value,
    price: parseFloat(form.value.price) || 0
  }

  try {
    if (editId.value) {
      await fetch(`${API}/${editId.value}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      })
      showToast('Product updated successfully.')
    } else {
      await fetch(API, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      })
      showToast('Product added to inventory.')
    }
  } catch {
    showToast('Something went wrong.', 'error')
  }

  resetForm()
  load()
}

function editItem(item) {
  editId.value = item.id
  form.value = { name: item.name, description: item.description, price: item.price }
  document.querySelector('.sidebar')?.scrollTo({ top: 0, behavior: 'smooth' })
}

function resetForm() {
  editId.value = null
  form.value = { name: '', description: '', price: '' }
}

async function remove(id) {
  try {
    await fetch(`${API}/${id}`, { method: 'DELETE' })
    showToast('Product removed from inventory.')
  } catch {
    showToast('Could not delete product.', 'error')
  }
  deleteConfirmId.value = null
  load()
}

function tagColor(price) {
  const p = Number(price)
  if (p >= 1000) return 'tag-gold'
  if (p >= 500) return 'tag-teal'
  return 'tag-slate'
}

onMounted(load)
</script>

<template>
  <div class="shell">

    <!-- TOAST -->
    <Transition name="toast">
      <div v-if="toast" :class="['toast', toast.type]">
        <span class="toast-dot"></span>
        {{ toast.message }}
      </div>
    </Transition>

    <!-- SIDEBAR -->
    <aside class="sidebar">
      <div class="brand">
        <div class="brand-mark">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M10 2L18 6V14L10 18L2 14V6L10 2Z" stroke="currentColor" stroke-width="1.5" fill="none"/>
            <path d="M10 6L14 8V12L10 14L6 12V8L10 6Z" fill="currentColor" opacity="0.5"/>
          </svg>
        </div>
        <div class="brand-text">
          <span class="brand-name">StockFlow</span>
          <span class="brand-sub">Inventory Suite</span>
        </div>
      </div>

      <div class="divider"></div>

      <!-- STATS -->
      <div class="stat-row">
        <div class="stat-block">
          <span class="stat-label">Products</span>
          <span class="stat-value">{{ items.length }}</span>
        </div>
        <div class="stat-block">
          <span class="stat-label">Avg. Price</span>
          <span class="stat-value">₱{{ avgPrice.toFixed(0) }}</span>
        </div>
      </div>

      <div class="total-card">
        <div class="total-label">
          <span class="pulse-dot"></span>
          Portfolio Value
        </div>
        <div class="total-amount">₱{{ totalValue.toLocaleString('en-PH', { minimumFractionDigits: 2 }) }}</div>
        <div v-if="highestItem" class="total-note">
          Top: <strong>{{ highestItem.name }}</strong> @ ₱{{ Number(highestItem.price).toFixed(2) }}
        </div>
      </div>

      <div class="divider"></div>

      <!-- FORM -->
      <div class="form-section">
        <div class="form-heading">
          <span class="form-icon">{{ editId ? '✎' : '+' }}</span>
          <div>
            <h3>{{ editId ? 'Edit Product' : 'New Product' }}</h3>
            <p>{{ editId ? 'Modify existing record' : 'Add to your inventory' }}</p>
          </div>
        </div>

        <form @submit.prevent="save">
          <div class="field">
            <label>Product Name</label>
            <input v-model="form.name" type="text" placeholder="e.g. Mechanical Keyboard" required />
          </div>
          <div class="field">
            <label>Description</label>
            <textarea v-model="form.description" rows="3" placeholder="Optional details..."></textarea>
          </div>
          <div class="field">
            <label>Price <span class="currency">₱</span></label>
            <input v-model="form.price" type="number" step="0.01" min="0" placeholder="0.00" />
          </div>

          <div class="form-actions">
            <button v-if="editId" type="button" class="btn-ghost" @click="resetForm">
              Cancel
            </button>
            <button type="submit" class="btn-primary">
              {{ editId ? 'Save Changes' : 'Add Product' }}
            </button>
          </div>
        </form>
      </div>
    </aside>

    <!-- MAIN -->
    <main class="main">
      <div class="main-header">
        <div class="header-left">
          <h1>Inventory</h1>
          <p>{{ filteredItems.length }} of {{ items.length }} products</p>
        </div>
        <div class="header-right">
          <div class="search-wrap">
            <svg class="search-icon" width="16" height="16" viewBox="0 0 16 16" fill="none">
              <circle cx="7" cy="7" r="5" stroke="currentColor" stroke-width="1.5"/>
              <path d="M11 11L14 14" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
            </svg>
            <input v-model="searchQuery" type="text" placeholder="Search products..." class="search-input" />
          </div>
          <div class="live-tag">
            <span class="live-dot"></span> Live
          </div>
        </div>
      </div>

      <!-- LOADING -->
      <div v-if="loading" class="center-state">
        <div class="spinner"></div>
        <p>Fetching inventory...</p>
      </div>

      <!-- EMPTY -->
      <div v-else-if="items.length === 0" class="center-state">
        <div class="empty-icon">◎</div>
        <h3>No products yet</h3>
        <p>Use the form to add your first item.</p>
      </div>

      <!-- NO RESULTS -->
      <div v-else-if="filteredItems.length === 0" class="center-state">
        <div class="empty-icon">⊘</div>
        <h3>No results</h3>
        <p>Try a different search term.</p>
      </div>

      <!-- GRID -->
      <div v-else class="grid">
        <div
          class="card"
          v-for="item in filteredItems"
          :key="item.id"
        >
          <div class="card-inner">
            <div class="card-head">
              <div class="card-avatar">{{ item.name.charAt(0).toUpperCase() }}</div>
              <span :class="['price-tag', tagColor(item.price)]">
                ₱{{ Number(item.price).toLocaleString('en-PH', { minimumFractionDigits: 2 }) }}
              </span>
            </div>

            <div class="card-body">
              <h4>{{ item.name }}</h4>
              <p>{{ item.description || 'No description provided.' }}</p>
            </div>

            <!-- DELETE CONFIRM OVERLAY -->
            <Transition name="confirm">
              <div v-if="deleteConfirmId === item.id" class="confirm-overlay">
                <p>Remove <strong>{{ item.name }}</strong>?</p>
                <div class="confirm-btns">
                  <button class="btn-confirm-cancel" @click="deleteConfirmId = null">Keep</button>
                  <button class="btn-confirm-delete" @click="remove(item.id)">Remove</button>
                </div>
              </div>
            </Transition>

            <div class="card-foot">
              <button class="action-edit" @click="editItem(item)">
                <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
                  <path d="M9.5 2.5L11.5 4.5L4.5 11.5H2.5V9.5L9.5 2.5Z" stroke="currentColor" stroke-width="1.3" stroke-linejoin="round"/>
                </svg>
                Edit
              </button>
              <button class="action-delete" @click="deleteConfirmId = item.id">
                <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
                  <path d="M2 4H12M5 4V2.5H9V4M5.5 6.5V10.5M8.5 6.5V10.5M3 4L3.5 11.5H10.5L11 4" stroke="currentColor" stroke-width="1.3" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
                Delete
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;500;600;700;800&family=DM+Sans:wght@300;400;500&display=swap');

*, *::before, *::after {
  margin: 0; padding: 0;
  box-sizing: border-box;
}

:root {
  --bg:        #0a0d14;
  --surface:   #111520;
  --surface2:  #181d2e;
  --border:    rgba(255,255,255,0.07);
  --border2:   rgba(255,255,255,0.12);
  --text:      #f0f2f7;
  --muted:     #6b7594;
  --accent:    #e8b84b;
  --accent2:   #f0d080;
  --teal:      #2dd4bf;
  --danger:    #f87171;
  --radius:    18px;
  --font-head: 'Syne', sans-serif;
  --font-body: 'DM Sans', sans-serif;
}

html, body { height: 100%; background: var(--bg); }

body {
  font-family: var(--font-body);
  color: var(--text);
  -webkit-font-smoothing: antialiased;
}

/* ── SHELL ── */
.shell {
  display: grid;
  grid-template-columns: 300px 1fr;
  min-height: 100vh;
}

/* ── TOAST ── */
.toast {
  position: fixed;
  bottom: 28px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 999;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 22px;
  border-radius: 999px;
  font-family: var(--font-body);
  font-size: 13.5px;
  font-weight: 500;
  backdrop-filter: blur(20px);
  border: 1px solid var(--border2);
  background: rgba(24,29,46,0.95);
  color: var(--text);
  white-space: nowrap;
  box-shadow: 0 20px 60px rgba(0,0,0,0.5);
}
.toast.error .toast-dot { background: var(--danger); }
.toast-dot {
  width: 7px; height: 7px;
  border-radius: 50%;
  background: var(--accent);
  flex-shrink: 0;
}
.toast-enter-active, .toast-leave-active { transition: all 0.35s cubic-bezier(.4,0,.2,1); }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateX(-50%) translateY(16px); }

/* ── SIDEBAR ── */
.sidebar {
  background: var(--surface);
  border-right: 1px solid var(--border);
  padding: 28px 24px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  overflow-y: auto;
  position: sticky;
  top: 0;
  height: 100vh;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
}
.brand-mark {
  width: 44px; height: 44px;
  border-radius: 14px;
  background: linear-gradient(135deg, #1e2640, #2a3260);
  border: 1px solid rgba(232,184,75,0.25);
  display: grid;
  place-items: center;
  color: var(--accent);
  flex-shrink: 0;
}
.brand-name {
  display: block;
  font-family: var(--font-head);
  font-size: 18px;
  font-weight: 800;
  letter-spacing: -0.3px;
  color: var(--text);
}
.brand-sub {
  display: block;
  font-size: 11px;
  color: var(--muted);
  letter-spacing: 0.06em;
  text-transform: uppercase;
}

.divider {
  height: 1px;
  background: var(--border);
}

/* STATS */
.stat-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}
.stat-block {
  background: var(--surface2);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 14px;
}
.stat-label {
  display: block;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  color: var(--muted);
  margin-bottom: 6px;
}
.stat-value {
  font-family: var(--font-head);
  font-size: 22px;
  font-weight: 700;
  color: var(--text);
}

.total-card {
  background: linear-gradient(135deg, #1a1e30, #222840);
  border: 1px solid rgba(232,184,75,0.2);
  border-radius: var(--radius);
  padding: 20px;
  position: relative;
  overflow: hidden;
}
.total-card::before {
  content: '';
  position: absolute;
  top: -40px; right: -40px;
  width: 120px; height: 120px;
  background: radial-gradient(circle, rgba(232,184,75,0.12) 0%, transparent 70%);
}
.total-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  color: var(--muted);
  margin-bottom: 10px;
}
.pulse-dot {
  width: 7px; height: 7px;
  border-radius: 50%;
  background: var(--accent);
  animation: pulse 2s ease-in-out infinite;
}
@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(0.85); }
}
.total-amount {
  font-family: var(--font-head);
  font-size: 30px;
  font-weight: 800;
  color: var(--accent);
  letter-spacing: -1px;
  line-height: 1.1;
}
.total-note {
  margin-top: 10px;
  font-size: 12px;
  color: var(--muted);
}
.total-note strong { color: rgba(240,208,128,0.8); font-weight: 500; }

/* FORM */
.form-section {}
.form-heading {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  margin-bottom: 20px;
}
.form-icon {
  width: 34px; height: 34px;
  border-radius: 10px;
  background: var(--surface2);
  border: 1px solid var(--border2);
  display: grid;
  place-items: center;
  font-size: 16px;
  color: var(--accent);
  flex-shrink: 0;
  margin-top: 2px;
}
.form-heading h3 {
  font-family: var(--font-head);
  font-size: 16px;
  font-weight: 700;
  margin-bottom: 2px;
}
.form-heading p { font-size: 12px; color: var(--muted); }

.field { margin-bottom: 14px; }
.field label {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: var(--muted);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: 7px;
}
.currency { color: var(--accent); font-weight: 600; }
.field input, .field textarea {
  width: 100%;
  background: var(--surface2);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 12px 14px;
  color: var(--text);
  font-family: var(--font-body);
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
  resize: none;
}
.field input::placeholder, .field textarea::placeholder { color: #3d4460; }
.field input:focus, .field textarea:focus { border-color: rgba(232,184,75,0.4); }

.form-actions { display: flex; gap: 8px; margin-top: 6px; }
.btn-primary, .btn-ghost {
  flex: 1;
  border: none;
  border-radius: 12px;
  padding: 13px;
  font-family: var(--font-body);
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}
.btn-primary {
  background: var(--accent);
  color: #0a0d14;
  font-weight: 600;
}
.btn-primary:hover { background: var(--accent2); transform: translateY(-1px); }
.btn-ghost {
  background: var(--surface2);
  color: var(--muted);
  border: 1px solid var(--border);
}
.btn-ghost:hover { color: var(--text); border-color: var(--border2); }

/* ── MAIN ── */
.main {
  padding: 32px 36px;
  min-height: 100vh;
}

.main-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 32px;
}
.header-left h1 {
  font-family: var(--font-head);
  font-size: 36px;
  font-weight: 800;
  letter-spacing: -1.5px;
  line-height: 1;
  margin-bottom: 6px;
}
.header-left p { font-size: 13px; color: var(--muted); }

.header-right { display: flex; align-items: center; gap: 12px; }
.search-wrap {
  position: relative;
  display: flex;
  align-items: center;
}
.search-icon {
  position: absolute;
  left: 12px;
  color: var(--muted);
  pointer-events: none;
}
.search-input {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 10px 14px 10px 36px;
  color: var(--text);
  font-family: var(--font-body);
  font-size: 13.5px;
  outline: none;
  width: 220px;
  transition: border-color 0.2s, width 0.2s;
}
.search-input::placeholder { color: var(--muted); }
.search-input:focus { border-color: rgba(232,184,75,0.35); width: 260px; }

.live-tag {
  display: flex;
  align-items: center;
  gap: 7px;
  background: rgba(45,212,191,0.07);
  border: 1px solid rgba(45,212,191,0.2);
  color: var(--teal);
  font-size: 12px;
  font-weight: 500;
  padding: 9px 14px;
  border-radius: 999px;
  letter-spacing: 0.04em;
}
.live-dot {
  width: 6px; height: 6px;
  border-radius: 50%;
  background: var(--teal);
  animation: pulse 2s ease-in-out infinite;
}

/* CENTER STATES */
.center-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 60vh;
  color: var(--muted);
  gap: 12px;
}
.center-state h3 { font-family: var(--font-head); font-size: 20px; color: var(--text); }
.center-state p { font-size: 14px; }
.empty-icon { font-size: 48px; opacity: 0.2; }
.spinner {
  width: 36px; height: 36px;
  border: 2.5px solid var(--border);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── GRID ── */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(270px, 1fr));
  gap: 18px;
}

.card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  overflow: hidden;
  transition: border-color 0.2s, transform 0.2s, box-shadow 0.2s;
}
.card:hover {
  border-color: var(--border2);
  transform: translateY(-3px);
  box-shadow: 0 20px 50px rgba(0,0,0,0.35);
}
.card-inner {
  padding: 22px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  position: relative;
}

.card-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.card-avatar {
  width: 46px; height: 46px;
  border-radius: 14px;
  background: linear-gradient(135deg, #1e2640, #2a3260);
  border: 1px solid rgba(232,184,75,0.15);
  display: grid;
  place-items: center;
  font-family: var(--font-head);
  font-size: 20px;
  font-weight: 700;
  color: var(--accent);
}

.price-tag {
  font-size: 12px;
  font-weight: 600;
  padding: 6px 12px;
  border-radius: 999px;
  letter-spacing: 0.02em;
}
.tag-gold { background: rgba(232,184,75,0.12); color: var(--accent); border: 1px solid rgba(232,184,75,0.2); }
.tag-teal { background: rgba(45,212,191,0.1); color: var(--teal); border: 1px solid rgba(45,212,191,0.2); }
.tag-slate { background: var(--surface2); color: var(--muted); border: 1px solid var(--border); }

.card-body h4 {
  font-family: var(--font-head);
  font-size: 17px;
  font-weight: 700;
  margin-bottom: 6px;
  letter-spacing: -0.3px;
}
.card-body p {
  font-size: 13px;
  color: var(--muted);
  line-height: 1.65;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.card-foot {
  display: flex;
  gap: 8px;
  margin-top: 4px;
}
.action-edit, .action-delete {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  border: none;
  border-radius: 10px;
  padding: 10px;
  font-family: var(--font-body);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}
.action-edit {
  background: var(--surface2);
  color: var(--muted);
  border: 1px solid var(--border);
}
.action-edit:hover { color: var(--accent); border-color: rgba(232,184,75,0.3); background: rgba(232,184,75,0.06); }
.action-delete {
  background: transparent;
  color: #4a3030;
  border: 1px solid #2a1a1a;
}
.action-delete:hover { color: var(--danger); border-color: rgba(248,113,113,0.3); background: rgba(248,113,113,0.06); }

/* CONFIRM OVERLAY */
.confirm-overlay {
  position: absolute;
  inset: 0;
  background: rgba(10,13,20,0.95);
  backdrop-filter: blur(4px);
  border-radius: var(--radius);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 16px;
  padding: 24px;
}
.confirm-overlay p { font-size: 14px; color: var(--muted); text-align: center; }
.confirm-overlay p strong { color: var(--text); }
.confirm-btns { display: flex; gap: 8px; width: 100%; }
.btn-confirm-cancel, .btn-confirm-delete {
  flex: 1;
  border: none;
  border-radius: 10px;
  padding: 11px;
  font-family: var(--font-body);
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}
.btn-confirm-cancel { background: var(--surface2); color: var(--muted); border: 1px solid var(--border); }
.btn-confirm-cancel:hover { color: var(--text); }
.btn-confirm-delete { background: rgba(248,113,113,0.12); color: var(--danger); border: 1px solid rgba(248,113,113,0.25); }
.btn-confirm-delete:hover { background: rgba(248,113,113,0.2); }

.confirm-enter-active, .confirm-leave-active { transition: all 0.2s ease; }
.confirm-enter-from, .confirm-leave-to { opacity: 0; }

/* ── RESPONSIVE ── */
@media (max-width: 900px) {
  .shell { grid-template-columns: 1fr; }
  .sidebar { position: static; height: auto; }
  .main { padding: 24px 20px; }
  .main-header { flex-direction: column; gap: 16px; }
  .header-right { width: 100%; }
  .search-input { width: 100%; }
  .search-input:focus { width: 100%; }
}
</style>