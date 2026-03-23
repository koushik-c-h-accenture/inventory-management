<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="budget-layout">
          <div class="budget-slider-section">
            <label class="budget-label" for="budget-slider">Available Budget</label>
            <input
              id="budget-slider"
              type="range"
              min="0"
              max="100000"
              step="500"
              v-model.number="budget"
              class="budget-slider"
            />
          </div>
          <div class="budget-display-section">
            <div class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
            <div class="budget-summary">
              {{ recommendedItems.length }} items recommended within budget
              &middot;
              Total cost: {{ currencySymbol }}{{ totalCost.toLocaleString() }}
            </div>
          </div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="placedOrder" class="success-banner">
        <div class="success-title">Order {{ placedOrder.order_number }} placed successfully</div>
        <div class="success-delivery">Expected delivery: {{ formatDate(placedOrder.expected_delivery) }}</div>
        <div class="success-note">Navigate to the Orders tab to view submitted orders.</div>
      </div>

      <!-- Recommendations Table -->
      <div v-if="!placedOrder" class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="allRecommendations.length === 0" class="empty-state">
          No items need restocking with current filters.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Category</th>
                <th>Demand Gap</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.id">
                <td><strong>{{ item.item_name }}</strong></td>
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.demand_gap.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.estimated_cost.toLocaleString() }}</td>
                <td>
                  <span :class="['badge', trendBadgeClass(item.trend)]">
                    {{ item.trend }}
                  </span>
                </td>
              </tr>
            </tbody>
            <tfoot v-if="recommendedItems.length > 0">
              <tr class="total-row">
                <td colspan="5"><strong>Total</strong></td>
                <td><strong>{{ currencySymbol }}{{ totalCost.toLocaleString() }}</strong></td>
                <td></td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
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
    const { selectedLocation } = useFilters()
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const initialized = ref(false)
    const error = ref(null)
    const allRecommendations = ref([])
    const budget = ref(50000)
    const placing = ref(false)
    const placedOrder = ref(null)

    const recommendedItems = computed(() => {
      let runningTotal = 0
      const selected = []
      for (const item of allRecommendations.value) {
        if (runningTotal + item.estimated_cost <= budget.value) {
          runningTotal += item.estimated_cost
          selected.push(item)
        }
      }
      return selected
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    })

    const loadRecommendations = async () => {
      error.value = null
      try {
        if (!initialized.value) loading.value = true
        const data = await api.getRestockingRecommendations(selectedLocation.value)
        allRecommendations.value = data
        initialized.value = true
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch(selectedLocation, () => {
      placedOrder.value = null
      loadRecommendations()
    })

    const placeOrder = async () => {
      placing.value = true
      error.value = null
      try {
        const items = recommendedItems.value.map(r => ({
          sku: r.item_sku,
          name: r.item_name,
          quantity: r.demand_gap,
          unit_price: r.unit_cost
        }))
        const warehouse = selectedLocation.value === 'all' ? 'All Warehouses' : selectedLocation.value
        const result = await api.createOrder({
          customer: 'Restocking Order',
          items,
          warehouse,
          category: 'Mixed'
        })
        placedOrder.value = result
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    const trendBadgeClass = (trend) => {
      if (trend === 'increasing') return 'success'
      if (trend === 'decreasing') return 'danger'
      return 'info'
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      allRecommendations,
      budget,
      placing,
      placedOrder,
      currencySymbol,
      recommendedItems,
      totalCost,
      placeOrder,
      formatDate,
      trendBadgeClass
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget Card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-layout {
  display: flex;
  align-items: center;
  gap: 2.5rem;
}

.budget-slider-section {
  flex: 0 0 68%;
  display: flex;
  flex-direction: column;
  gap: 0.625rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  border-radius: 4px;
}

.budget-display-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  line-height: 1.1;
}

.budget-summary {
  font-size: 0.813rem;
  color: #64748b;
}

/* Success Banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  padding: 1.25rem;
  margin-bottom: 1.25rem;
}

.success-title {
  font-size: 1rem;
  font-weight: 700;
  margin-bottom: 0.375rem;
}

.success-delivery {
  font-size: 0.875rem;
  margin-bottom: 0.25rem;
}

.success-note {
  font-size: 0.813rem;
  opacity: 0.85;
}

/* Table */
.sku-cell {
  font-family: monospace;
  font-size: 0.813rem;
  color: #64748b;
}

tfoot .total-row {
  background: #f8fafc;
}

tfoot .total-row td {
  font-weight: 600;
  color: #0f172a;
  border-top: 2px solid #e2e8f0;
}

/* Place Order Button */
.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Empty State */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
