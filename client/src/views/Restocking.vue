<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Items below reorder point. Set a budget and select items to restock.</p>
    </div>

    <div v-if="loading" class="loading-state">Loading recommendations...</div>
    <div v-else-if="error" class="error-state">{{ error }}</div>

    <template v-else>
      <!-- Success banner shown after a successful order submission -->
      <div class="success-banner" v-if="orderSuccess">
        <span>
          Order <strong>{{ orderSuccess.order_number }}</strong> submitted —
          {{ orderSuccess.item_count }} item(s), ${{ orderSuccess.total_cost.toLocaleString() }}.
          Expected delivery: <strong>{{ formatDate(orderSuccess.expected_delivery) }}</strong>.
        </span>
        <button class="dismiss-btn" @click="orderSuccess = null">Dismiss</button>
      </div>

      <!-- Budget panel -->
      <div class="card budget-panel">
        <div class="budget-header">
          <div>
            <h3 class="card-title">Available Budget</h3>
            <p class="budget-hint">Slide to set your restocking budget</p>
          </div>
          <span class="budget-value">${{ budget.toLocaleString() }}</span>
        </div>
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="50000"
          step="1000"
          class="budget-slider"
        />
        <div class="budget-bar-wrap">
          <div class="budget-bar-track">
            <div
              class="budget-bar-fill"
              :class="{ 'over-budget': spentAmount > budget }"
              :style="{ width: budget > 0 ? Math.min((spentAmount / budget) * 100, 100) + '%' : '0%' }"
            ></div>
          </div>
        </div>
        <div class="budget-legend">
          <span>Selected: <strong>${{ spentAmount.toLocaleString() }}</strong></span>
          <span
            class="remaining"
            :class="{ 'over': remainingBudget < 0 }"
          >
            Remaining: <strong>${{ remainingBudget.toLocaleString() }}</strong>
          </span>
        </div>
      </div>

      <!-- Empty state when no items are below reorder point -->
      <div class="card empty-card" v-if="recommendations.length === 0">
        <p class="empty-state">All inventory items are at or above their reorder point.</p>
      </div>

      <!-- Recommendations table -->
      <div class="card" v-else>
        <div class="card-header">
          <h3 class="card-title">
            Items Below Reorder Point
            <span class="count-badge">{{ recommendations.length }}</span>
          </h3>
          <span class="selected-label" v-if="selectedCount > 0">
            {{ selectedCount }} selected
          </span>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>SKU</th>
                <th>Name</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th class="col-num">On Hand</th>
                <th class="col-num">Reorder Point</th>
                <th class="col-num">Units Needed</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.id"
                class="rec-row"
                :class="{
                  'row-selected': selectedIds.has(item.id),
                  'row-unaffordable': !isAffordable(item)
                }"
                @click="toggleSelection(item)"
              >
                <td class="col-check" @click.stop>
                  <input
                    type="checkbox"
                    :checked="selectedIds.has(item.id)"
                    :disabled="!isAffordable(item)"
                    @change="toggleSelection(item)"
                  />
                </td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td class="col-num">{{ item.quantity_on_hand }}</td>
                <td class="col-num">{{ item.reorder_point }}</td>
                <td class="col-num">
                  <span class="badge danger">{{ item.units_needed }}</span>
                </td>
                <td class="col-num">${{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num"><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>

    <!-- Sticky place-order bar — visible when items are selected -->
    <div class="place-order-bar" v-if="selectedCount > 0 || orderError">
      <div class="order-bar-summary">
        <span>{{ selectedCount }} item(s) selected</span>
        <span class="order-total">Total: <strong>${{ spentAmount.toLocaleString() }}</strong></span>
      </div>
      <div class="order-error" v-if="orderError">{{ orderError }}</div>
      <button
        class="btn-primary"
        :disabled="!canPlaceOrder"
        @click="placeOrder"
      >
        {{ submitting ? 'Placing Order...' : 'Place Order' }}
      </button>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentLocale } = useI18n()

    const budget = ref(10000)
    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const selectedIds = ref(new Set())
    const submitting = ref(false)
    const orderSuccess = ref(null)
    const orderError = ref(null)

    const spentAmount = computed(() =>
      recommendations.value
        .filter(r => selectedIds.value.has(r.id))
        .reduce((sum, r) => sum + r.total_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - spentAmount.value)

    // An item is unaffordable only when it is NOT yet selected and exceeds remaining budget.
    // Already-selected items remain selectable so the user can deselect them.
    const isAffordable = (item) =>
      selectedIds.value.has(item.id) || item.total_cost <= remainingBudget.value

    const selectedCount = computed(() => selectedIds.value.size)
    const canPlaceOrder = computed(() => selectedCount.value > 0 && !submitting.value)

    // Reassign Set to trigger Vue reactivity (Sets are not tracked by mutation)
    const toggleSelection = (item) => {
      const next = new Set(selectedIds.value)
      if (next.has(item.id)) {
        next.delete(item.id)
      } else if (isAffordable(item)) {
        next.add(item.id)
      }
      selectedIds.value = next
    }

    const formatDate = (dateStr) => {
      if (!dateStr) return ''
      const date = new Date(dateStr)
      if (isNaN(date.getTime())) return dateStr
      return date.toLocaleDateString(currentLocale.value === 'ja' ? 'ja-JP' : 'en-US', {
        year: 'numeric', month: 'short', day: 'numeric'
      })
    }

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load recommendations. Is the backend running?'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return
      try {
        submitting.value = true
        orderError.value = null
        const selectedItems = recommendations.value.filter(r => selectedIds.value.has(r.id))
        const payload = {
          items: selectedItems.map(r => ({
            inventory_item_id: r.id,
            sku: r.sku,
            name: r.name,
            units_needed: r.units_needed,
            unit_cost: r.unit_cost,
            total_cost: r.total_cost,
          }))
        }
        orderSuccess.value = await api.createRestockingOrder(payload)
        selectedIds.value = new Set()
        await loadRecommendations()
      } catch (err) {
        orderError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      budget,
      loading,
      error,
      recommendations,
      selectedIds,
      submitting,
      orderSuccess,
      orderError,
      spentAmount,
      remainingBudget,
      isAffordable,
      selectedCount,
      canPlaceOrder,
      toggleSelection,
      formatDate,
      placeOrder,
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 1.5rem;
  padding-bottom: 6rem; /* room for sticky place-order bar */
}

.page-header {
  margin-bottom: 1.5rem;
}
.page-header h2 {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.25rem;
}
.page-header p {
  color: #64748b;
  font-size: 0.9rem;
}

/* ── States ── */
.loading-state,
.error-state {
  padding: 2rem;
  color: #64748b;
  font-size: 0.9rem;
}
.error-state { color: #dc2626; }

/* ── Success banner ── */
.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.9rem;
  gap: 1rem;
}
.dismiss-btn {
  background: none;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  white-space: nowrap;
  flex-shrink: 0;
}
.dismiss-btn:hover { background: #a7f3d0; }

/* ── Card base ── */
.card {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  margin-bottom: 1.25rem;
  overflow: hidden;
}

/* ── Budget panel ── */
.budget-panel { padding: 1.5rem; }

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1rem;
}
.card-title {
  font-size: 0.95rem;
  font-weight: 700;
  color: #0f172a;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.budget-hint {
  font-size: 0.78rem;
  color: #94a3b8;
  margin-top: 0.15rem;
}
.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.75rem;
}

.budget-bar-wrap { margin-bottom: 0.5rem; }
.budget-bar-track {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}
.budget-bar-fill {
  height: 100%;
  background: #10b981; /* green to distinguish from the blue slider */
  border-radius: 4px;
  transition: width 0.3s ease, background 0.2s ease;
}
.budget-bar-fill.over-budget { background: #dc2626; }

.budget-legend {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: #64748b;
}
.remaining.over { color: #dc2626; }

/* ── Card header ── */
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}
.count-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  background: #f1f5f9;
  color: #475569;
  font-size: 0.72rem;
  font-weight: 700;
  padding: 0.1rem 0.5rem;
  border-radius: 999px;
  margin-left: 0.4rem;
}
.selected-label {
  font-size: 0.82rem;
  color: #2563eb;
  font-weight: 600;
}

/* ── Table ── */
.table-container { overflow-x: auto; }
table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}
thead th {
  background: #f8fafc;
  color: #64748b;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #e2e8f0;
  white-space: nowrap;
}
tbody td {
  padding: 0.75rem 1rem;
  border-bottom: 1px solid #f1f5f9;
  color: #334155;
  vertical-align: middle;
}
.col-check {
  width: 2.5rem;
  text-align: center;
}
.col-num { text-align: right; }

/* ── Row states ── */
.rec-row {
  cursor: pointer;
  transition: background 0.1s ease;
}
.rec-row:last-child td { border-bottom: none; }
.rec-row:hover td { background: #f8fafc; }
.row-selected td { background: #eff6ff !important; }
.row-unaffordable { pointer-events: none; }
.row-unaffordable td { opacity: 0.35; }

/* ── Badge ── */
.badge {
  display: inline-flex;
  align-items: center;
  padding: 0.2rem 0.55rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
}
.badge.danger { background: #fee2e2; color: #991b1b; }

/* ── Empty states ── */
.empty-card { padding: 0; }
.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9rem;
}

/* ── Place-order sticky bar ── */
.place-order-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  background: #fff;
  border-top: 1px solid #e2e8f0;
  box-shadow: 0 -4px 16px rgba(0, 0, 0, 0.06);
  padding: 1rem 2rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  z-index: 100;
}
.order-bar-summary {
  display: flex;
  gap: 1.5rem;
  align-items: center;
  font-size: 0.9rem;
  color: #334155;
}
.order-total { font-size: 1rem; }
.order-error {
  flex: 1;
  font-size: 0.82rem;
  color: #dc2626;
}
.btn-primary {
  background: #2563eb;
  color: #fff;
  padding: 0.625rem 1.75rem;
  border-radius: 8px;
  border: none;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
  white-space: nowrap;
  flex-shrink: 0;
}
.btn-primary:hover:not(:disabled) { background: #1d4ed8; }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
