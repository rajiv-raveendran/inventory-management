<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set your budget and we'll recommend items to restock based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Success banner -->
      <div v-if="submitted" class="success-banner">
        <div class="success-banner-content">
          <strong>Order submitted successfully!</strong>
          <span v-if="lastOrder">
            Order {{ lastOrder.id }} placed — expected delivery
            {{ formatDate(lastOrder.expected_delivery) }}
          </span>
        </div>
      </div>

      <!-- Budget slider card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-control">
          <input
            type="range"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <div class="budget-labels">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}50,000</span>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking ({{ recommendedItems.length }} items)</h3>
          <div class="summary-info" v-if="recommendedItems.length > 0">
            <span class="summary-item">
              Total Cost: <strong>{{ currencySymbol }}{{ totalCost.toLocaleString() }}</strong>
            </span>
            <span class="summary-divider">|</span>
            <span class="summary-item">
              Remaining: <strong>{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</strong>
            </span>
          </div>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          <p v-if="budget === 0">Set a budget above to see restocking recommendations.</p>
          <p v-else>No items need restocking within the current budget. All items are sufficiently stocked, or the budget is too low to cover any single item.</p>
        </div>

        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Category</th>
                <th class="col-num">Current Stock</th>
                <th class="col-num">Reorder Point</th>
                <th class="col-num">Order Qty</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td>{{ item.name }}</td>
                <td><code class="sku">{{ item.sku }}</code></td>
                <td>{{ item.category }}</td>
                <td class="col-num">{{ item.quantity_on_hand }}</td>
                <td class="col-num">{{ item.reorder_point }}</td>
                <td class="col-num"><strong>{{ item.quantityNeeded }}</strong></td>
                <td class="col-num">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-num"><strong>{{ currencySymbol }}{{ item.itemCost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="card-footer">
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting || submitted"
            @click="placeOrder"
          >
            <span v-if="submitting">Submitting...</span>
            <span v-else>Place Order</span>
          </button>
          <span v-if="submitted" class="submitted-note">Order has been submitted.</span>
        </div>
      </div>
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
    const { currentCurrency, currentLocale } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(10000)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submitted = ref(false)
    const lastOrder = ref(null)

    const recommendedItems = computed(() => {
      if (budget.value === 0 || demandForecasts.value.length === 0 || inventoryItems.value.length === 0) {
        return []
      }

      // Build a map of inventory by SKU for fast lookup
      const inventoryBySku = {}
      for (const inv of inventoryItems.value) {
        inventoryBySku[inv.sku] = inv
      }

      // Match demand forecasts to inventory and filter to items needing restock
      const candidates = []
      for (const forecast of demandForecasts.value) {
        const inv = inventoryBySku[forecast.item_sku]
        if (!inv) continue
        if (inv.quantity_on_hand >= inv.reorder_point) continue

        const quantityNeeded = inv.reorder_point - inv.quantity_on_hand
        const itemCost = quantityNeeded * inv.unit_cost

        candidates.push({
          sku: inv.sku,
          name: inv.name,
          category: inv.category,
          quantity_on_hand: inv.quantity_on_hand,
          reorder_point: inv.reorder_point,
          unit_cost: inv.unit_cost,
          forecasted_demand: forecast.forecasted_demand,
          quantityNeeded,
          itemCost
        })
      }

      // Sort by forecasted_demand descending
      candidates.sort((a, b) => b.forecasted_demand - a.forecasted_demand)

      // Greedy selection within budget
      let remaining = budget.value
      const selected = []
      for (const item of candidates) {
        if (item.itemCost <= remaining) {
          selected.push(item)
          remaining -= item.itemCost
        }
      }

      return selected
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.itemCost, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - totalCost.value
    })

    const formatDate = (dateString) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return

      submitting.value = true
      try {
        const items = recommendedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.quantityNeeded,
          unit_cost: item.unit_cost,
          total_cost: item.itemCost
        }))

        const result = await api.submitRestockingOrder({
          items,
          total_cost: totalCost.value
        })

        lastOrder.value = result
        submitted.value = true
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      currencySymbol,
      budget,
      loading,
      error,
      submitting,
      submitted,
      lastOrder,
      recommendedItems,
      totalCost,
      remainingBudget,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #065f46;
}

.success-banner-content {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  font-size: 0.938rem;
}

.budget-control {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.budget-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
  font-size: 0.813rem;
  color: #94a3b8;
}

.summary-info {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
}

.summary-item strong {
  color: #0f172a;
}

.summary-divider {
  color: #e2e8f0;
}

.empty-state {
  padding: 3rem 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-num {
  text-align: right;
  width: 110px;
}

.sku {
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 2px 6px;
  border-radius: 4px;
  color: #475569;
}

.card-footer {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-top: 1rem;
  margin-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.submitted-note {
  font-size: 0.875rem;
  color: #059669;
  font-weight: 500;
}
</style>
