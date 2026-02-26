<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Auto-prioritized restock recommendations based on forecasted demand</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <template v-if="!loading">
      <!-- Budget Section -->
      <div class="budget-section">
        <div class="budget-header">
          <span class="budget-label">Budget</span>
          <span class="budget-amount">${{ budget.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
        </div>
        <input
          v-model.number="budget"
          type="range"
          min="1000"
          max="50000"
          step="500"
          class="budget-slider"
        />
        <div class="budget-bar-track">
          <div
            class="budget-bar-fill"
            :style="{ width: budgetPercent + '%', background: budgetBarColor }"
          ></div>
        </div>
        <div class="budget-summary">
          {{ selected.size }} of {{ recommendations.length }} items selected
          &middot;
          ${{ totalCost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
          of
          ${{ budget.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
          budget used
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="successOrder" class="success-banner">
        <span>
          Order {{ successOrder.order_number }} placed successfully.
          Expected delivery: {{ successOrder.expected_delivery }}
        </span>
        <button class="dismiss-btn" @click="successOrder = null">&times;</button>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendations.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="selected.size === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="!error && recommendations.length === 0" class="loading">
          No recommendations available for the current filters.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th></th>
                <th>Item</th>
                <th>SKU</th>
                <th>Category</th>
                <th>Trend</th>
                <th>On Hand</th>
                <th>Reorder Pt</th>
                <th>Suggested Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-excluded': !selected.has(item.item_sku) }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="selected.has(item.item_sku)"
                    @change="toggleRow(item.item_sku)"
                  />
                </td>
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.category }}</td>
                <td>
                  <span :class="['badge', item.trend.toLowerCase()]">
                    {{ item.trend }}
                  </span>
                </td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.reorder_point }}</td>
                <td>
                  <input
                    type="number"
                    class="qty-input"
                    min="0"
                    :value="effectiveQty(item)"
                    @change="updateQty(item.item_sku, $event.target.value)"
                  />
                </td>
                <td>${{ item.unit_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td>
                  ${{ (effectiveQty(item) * item.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    // Core state
    const loading = ref(false)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(10000)
    const editedQty = ref({})           // { [sku]: qty } — user-edited quantities
    const manualSelected = ref(new Set())   // skus explicitly checked by user
    const manualDeselected = ref(new Set()) // skus explicitly unchecked by user
    const successOrder = ref(null)
    const placing = ref(false)

    // Effective quantity: user edit overrides the API-suggested value
    const effectiveQty = (item) => {
      const val = editedQty.value[item.item_sku]
      return val !== undefined ? val : item.suggested_quantity
    }

    // Auto-select items that fit within budget (highest demand first, sorted by API)
    const autoSelected = computed(() => {
      let spent = 0
      const result = new Set()
      for (const item of recommendations.value) {
        const qty = effectiveQty(item)
        const cost = qty * item.unit_cost
        if (spent + cost <= budget.value) {
          result.add(item.item_sku)
          spent += cost
        }
      }
      return result
    })

    // Final selection: autoSelected + manualSelected - manualDeselected
    const selected = computed(() => {
      const result = new Set()
      for (const sku of autoSelected.value) {
        if (!manualDeselected.value.has(sku)) result.add(sku)
      }
      for (const sku of manualSelected.value) result.add(sku)
      return result
    })

    const totalCost = computed(() => {
      let total = 0
      for (const item of recommendations.value) {
        if (selected.value.has(item.item_sku)) {
          total += effectiveQty(item) * item.unit_cost
        }
      }
      return total
    })

    const budgetPercent = computed(() => Math.min((totalCost.value / budget.value) * 100, 100))

    const budgetBarColor = computed(() => {
      if (budgetPercent.value >= 90) return '#dc2626'
      if (budgetPercent.value >= 60) return '#d97706'
      return '#059669'
    })

    // Toggle a row's selection state between manual and auto
    const toggleRow = (sku) => {
      if (selected.value.has(sku)) {
        manualDeselected.value = new Set(manualDeselected.value)
        manualDeselected.value.add(sku)
        const next = new Set(manualSelected.value)
        next.delete(sku)
        manualSelected.value = next
      } else {
        const next = new Set(manualSelected.value)
        next.add(sku)
        manualSelected.value = next
        const nextD = new Set(manualDeselected.value)
        nextD.delete(sku)
        manualDeselected.value = nextD
      }
    }

    // Update qty for a row; clamp to >= 0
    const updateQty = (sku, val) => {
      const next = { ...editedQty.value }
      next[sku] = Math.max(0, parseInt(val) || 0)
      editedQty.value = next
    }

    const placeOrder = async () => {
      if (selected.value.size === 0 || placing.value) return
      placing.value = true
      error.value = null
      try {
        const items = recommendations.value
          .filter(r => selected.value.has(r.item_sku))
          .map(r => ({
            sku: r.item_sku,
            name: r.item_name,
            quantity: effectiveQty(r),
            unit_cost: r.unit_cost
          }))
        const payload = {
          items,
          total_value: Math.round(totalCost.value * 100) / 100,
          warehouse: null
        }
        successOrder.value = await api.placeRestockingOrder(payload)
        // Reset all selection and edit state after a successful order
        manualSelected.value = new Set()
        manualDeselected.value = new Set()
        editedQty.value = {}
      } catch (err) {
        console.error('Failed to place order:', err)
        error.value = 'Failed to place order. Please try again.'
      } finally {
        placing.value = false
      }
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        recommendations.value = await api.getRestockingRecommendations({
          warehouse: filters.warehouse,
          category: filters.category
        })
        // Clear edits and manual selections when data reloads
        editedQty.value = {}
        manualSelected.value = new Set()
        manualDeselected.value = new Set()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations.'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Reload when warehouse or category filter changes
    watch([selectedLocation, selectedCategory], () => {
      loadRecommendations()
    })

    onMounted(loadRecommendations)

    return {
      t,
      currentCurrency,
      loading,
      error,
      recommendations,
      budget,
      editedQty,
      successOrder,
      placing,
      autoSelected,
      selected,
      totalCost,
      budgetPercent,
      budgetBarColor,
      effectiveQty,
      toggleRow,
      updateQty,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-section {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.budget-label {
  font-weight: 600;
  color: #0f172a;
}

.budget-amount {
  font-size: 1.25rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-slider {
  width: 100%;
  margin-bottom: 0.75rem;
  accent-color: #2563eb;
}

.budget-bar-track {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.budget-bar-fill {
  height: 100%;
  border-radius: 4px;
  transition: background 0.3s;
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #065f46;
}

.dismiss-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.25rem;
  color: #065f46;
}

/* Greyed-out rows for items not in selection */
.row-excluded :deep(td) {
  color: #94a3b8;
  background: #f8fafc;
}

.qty-input {
  width: 70px;
  padding: 0.25rem 0.375rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  text-align: center;
  font-size: 0.875rem;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}
</style>
