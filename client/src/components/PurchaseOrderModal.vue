<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <div class="shortage-header">
              <div class="shortage-title-section">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
            </div>

            <!-- Create Mode -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div class="form-error" v-if="submitError">{{ submitError }}</div>

              <div class="form-group">
                <label class="form-label" for="supplier-name">Supplier Name</label>
                <input
                  id="supplier-name"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  required
                  :disabled="submitting"
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    min="1"
                    class="form-input"
                    required
                    :disabled="submitting"
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit-cost">Unit Cost</label>
                  <input
                    id="unit-cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    step="0.01"
                    min="0"
                    class="form-input"
                    required
                    :disabled="submitting"
                  />
                </div>
              </div>

              <div class="form-group">
                <label class="form-label" for="expected-delivery-date">Expected Delivery Date</label>
                <input
                  id="expected-delivery-date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                  :disabled="submitting"
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="notes">Notes (optional)</label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  class="form-textarea"
                  rows="3"
                  :disabled="submitting"
                ></textarea>
              </div>
            </form>

            <!-- View Mode -->
            <div v-else>
              <div v-if="viewLoading" class="view-loading">Loading purchase order...</div>
              <div v-else-if="viewError" class="view-error">{{ viewError }}</div>
              <div v-else-if="purchaseOrder" class="po-details">
                <div class="info-grid">
                  <div class="info-item">
                    <div class="info-label">Supplier</div>
                    <div class="info-value">{{ purchaseOrder.supplier_name }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Status</div>
                    <div class="info-value">
                      <span class="badge status">{{ purchaseOrder.status }}</span>
                    </div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Quantity</div>
                    <div class="info-value">{{ purchaseOrder.quantity }} units</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Unit Cost</div>
                    <div class="info-value">{{ formatCurrency(purchaseOrder.unit_cost) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Total Cost</div>
                    <div class="info-value">{{ formatCurrency(totalCost) }}</div>
                  </div>

                  <div class="info-item">
                    <div class="info-label">Expected Delivery</div>
                    <div class="info-value">{{ formatDate(purchaseOrder.expected_delivery_date) }}</div>
                  </div>
                </div>

                <div class="info-item notes-item" v-if="purchaseOrder.notes">
                  <div class="info-label">Notes</div>
                  <div class="info-value">{{ purchaseOrder.notes }}</div>
                </div>
              </div>
              <div v-else class="view-error">No purchase order found for this item.</div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close" :disabled="submitting">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              type="submit"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create'
  }
})

const emit = defineEmits(['close', 'po-created'])

const defaultForm = () => ({
  supplier_name: '',
  quantity: null,
  unit_cost: null,
  expected_delivery_date: '',
  notes: ''
})

const form = ref(defaultForm())
const submitting = ref(false)
const submitError = ref(null)

const purchaseOrder = ref(null)
const viewLoading = ref(false)
const viewError = ref(null)

const totalCost = computed(() => {
  if (!purchaseOrder.value) return 0
  return purchaseOrder.value.quantity * purchaseOrder.value.unit_cost
})

const formatCurrency = (value) => {
  return (value || 0).toLocaleString('en-US', { style: 'currency', currency: 'USD' })
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

const resetState = () => {
  submitError.value = null
  submitting.value = false
  viewError.value = null
  viewLoading.value = false
  purchaseOrder.value = null

  if (props.backlogItem) {
    form.value = {
      supplier_name: '',
      quantity: props.backlogItem.quantity_needed - props.backlogItem.quantity_available,
      unit_cost: null,
      expected_delivery_date: '',
      notes: ''
    }
  } else {
    form.value = defaultForm()
  }
}

const loadPurchaseOrder = async () => {
  if (!props.backlogItem) return

  // Prefer the PO already attached client-side (e.g. right after creation)
  if (props.backlogItem.purchase_order) {
    purchaseOrder.value = props.backlogItem.purchase_order
    return
  }

  viewLoading.value = true
  viewError.value = null
  try {
    purchaseOrder.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
  } catch (err) {
    purchaseOrder.value = null
    viewError.value = 'No purchase order found for this item.'
  } finally {
    viewLoading.value = false
  }
}

// Reset internal state whenever the modal is (re)opened or the backlog item changes
watch(
  () => [props.isOpen, props.backlogItem, props.mode],
  () => {
    if (!props.isOpen) return
    resetState()
    if (props.mode === 'view') {
      loadPurchaseOrder()
    }
  },
  { immediate: true }
)

const close = () => {
  emit('close')
}

const submitForm = async () => {
  if (!props.backlogItem) return

  submitting.value = true
  submitError.value = null

  try {
    const response = await api.createPurchaseOrder({
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date,
      notes: form.value.notes || undefined
    })
    emit('po-created', response)
  } catch (err) {
    submitError.value = err.response?.data?.detail || 'Failed to create purchase order. Please try again.'
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

.shortage-header {
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.shortage-title-section {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.5rem 0;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-input,
.form-textarea {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  font-family: inherit;
  transition: border-color 0.15s ease;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #3b82f6;
}

.form-input:disabled,
.form-textarea:disabled {
  background: #f8fafc;
  cursor: not-allowed;
}

.form-textarea {
  resize: vertical;
}

.view-loading,
.view-error {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.view-error {
  color: #dc2626;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.notes-item {
  margin-top: 1.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.badge.status {
  padding: 0.25rem 0.625rem;
  border-radius: 6px;
  background: #dbeafe;
  color: #1e40af;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-secondary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #3b82f6;
  border: 1px solid #3b82f6;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
  border-color: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
