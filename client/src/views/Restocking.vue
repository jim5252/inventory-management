<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set an available budget and let the system recommend which items to reorder</p>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-control">
        <input
          type="range"
          class="budget-slider"
          min="0"
          max="50000"
          step="500"
          v-model.number="budget"
        />
        <div class="budget-value">{{ formatCurrency(budget) }}</div>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="submitSuccess" class="success-banner">
        Order <strong>{{ submittedOrder.order_number }}</strong> placed successfully.
        Expected delivery: {{ formatDate(submittedOrder.expected_delivery) }}
        (lead time: {{ submittedOrder.lead_time_days }} days).
      </div>

      <div v-if="submitError" class="error">{{ submitError }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Order ({{ recommendations.length }} items)</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within this budget. Try increasing the budget.
        </div>
        <template v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>SKU</th>
                  <th>Item</th>
                  <th>Category</th>
                  <th>Warehouse</th>
                  <th>Trend</th>
                  <th>Quantity</th>
                  <th>Unit Cost</th>
                  <th>Line Total</th>
                  <th>Lead Time</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendations" :key="item.sku">
                  <td><strong>{{ item.sku }}</strong></td>
                  <td>{{ item.item_name }}</td>
                  <td>{{ item.category }}</td>
                  <td>{{ item.warehouse }}</td>
                  <td>
                    <span :class="['badge', trendBadgeClass(item.trend)]">{{ item.trend }}</span>
                  </td>
                  <td>{{ item.quantity }}</td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td><strong>{{ formatCurrency(item.line_total) }}</strong></td>
                  <td>{{ item.lead_time_days }} days</td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="summary-line">
            <div class="summary-item">
              <span class="summary-label">Total Cost</span>
              <span class="summary-value">{{ formatCurrency(totalCost) }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">Remaining Budget</span>
              <span class="summary-value">{{ formatCurrency(remainingBudget) }}</span>
            </div>
          </div>

          <button
            class="place-order-btn"
            :disabled="loading || recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import { api } from '../api'

const budget = ref(25000)

const loading = ref(false)
const error = ref(null)
const recommendationsData = ref(null)

const submitting = ref(false)
const submitError = ref(null)
const submitSuccess = ref(false)
const submittedOrder = ref(null)

const recommendations = computed(() => recommendationsData.value?.recommendations || [])
const totalCost = computed(() => recommendationsData.value?.total_cost || 0)
const remainingBudget = computed(() => recommendationsData.value?.remaining_budget || 0)

const formatCurrency = (value) => {
  return `$${Number(value || 0).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
}

const formatDate = (dateString) => {
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
}

const trendBadgeClass = (trend) => {
  if (trend === 'increasing') return 'increasing'
  if (trend === 'decreasing') return 'decreasing'
  return 'stable'
}

const loadRecommendations = async () => {
  loading.value = true
  error.value = null
  try {
    recommendationsData.value = await api.getRestockingRecommendations(budget.value)
  } catch (err) {
    error.value = 'Failed to load recommendations: ' + err.message
  } finally {
    loading.value = false
  }
}

let debounceTimer = null
watch(budget, () => {
  submitSuccess.value = false
  submittedOrder.value = null
  submitError.value = null

  if (debounceTimer) clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    loadRecommendations()
  }, 300)
})

onBeforeUnmount(() => {
  if (debounceTimer) clearTimeout(debounceTimer)
})

const placeOrder = async () => {
  submitting.value = true
  submitError.value = null
  submitSuccess.value = false

  try {
    const payload = {
      budget: budget.value,
      items: recommendations.value.map(item => ({
        sku: item.sku,
        item_name: item.item_name,
        quantity: item.quantity,
        unit_cost: item.unit_cost,
        lead_time_days: item.lead_time_days
      }))
    }

    const order = await api.createRestockingOrder(payload)
    submittedOrder.value = order
    submitSuccess.value = true
    recommendationsData.value = null
  } catch (err) {
    submitError.value = 'Failed to place order: ' + err.message
  } finally {
    submitting.value = false
  }
}

onMounted(() => loadRecommendations())
</script>

<style scoped>
.budget-control {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
}

.budget-value {
  min-width: 140px;
  text-align: right;
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.empty-state {
  text-align: center;
  padding: 2.5rem;
  color: #64748b;
  font-size: 0.938rem;
}

.summary-line {
  display: flex;
  gap: 2rem;
  padding: 1rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.place-order-btn {
  margin-top: 1rem;
  padding: 0.75rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
