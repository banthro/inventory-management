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
            <div class="item-context">
              <div class="item-context-name">{{ backlogItem.item_name }}</div>
              <div class="item-context-meta">
                <span class="item-sku">{{ backlogItem.item_sku }}</span>
                <span class="item-order">Order: {{ backlogItem.order_id }}</span>
              </div>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" @submit.prevent="submitForm" class="po-form">
              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="supplier_name">Supplier Name</label>
                  <input
                    id="supplier_name"
                    v-model="form.supplier_name"
                    type="text"
                    class="form-input"
                    required
                    :disabled="submitting"
                  />
                </div>
              </div>

              <div class="form-row two-col">
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
                  <label class="form-label" for="unit_cost">Unit Cost ($)</label>
                  <input
                    id="unit_cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    min="0"
                    step="0.01"
                    class="form-input"
                    required
                    :disabled="submitting"
                  />
                </div>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="expected_delivery_date">Expected Delivery Date</label>
                  <input
                    id="expected_delivery_date"
                    v-model="form.expected_delivery_date"
                    type="date"
                    class="form-input"
                    required
                    :disabled="submitting"
                  />
                </div>
              </div>

              <div class="form-row" v-if="form.quantity && form.unit_cost">
                <div class="total-preview">
                  Total: <strong>{{ formatCurrency(form.quantity * form.unit_cost) }}</strong>
                </div>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="notes">Notes <span class="optional">(optional)</span></label>
                  <textarea
                    id="notes"
                    v-model="form.notes"
                    class="form-input form-textarea"
                    rows="3"
                    :disabled="submitting"
                  />
                </div>
              </div>

              <div v-if="submitError" class="form-error">{{ submitError }}</div>
            </form>

            <!-- View mode: read-only PO data -->
            <div v-else>
              <div v-if="viewLoading" class="state-message">Loading...</div>
              <div v-else-if="viewError" class="form-error">{{ viewError }}</div>
              <div v-else-if="poData" class="info-grid">
                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ poData.supplier_name }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="badge" :class="statusBadgeClass(poData.status)">{{ poData.status }}</span>
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ poData.quantity }} units</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">{{ formatCurrency(poData.unit_cost) }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value"><strong>{{ formatCurrency(poData.quantity * poData.unit_cost) }}</strong></div>
                </div>
                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(poData.expected_delivery_date) }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Created</div>
                  <div class="info-value">{{ formatDate(poData.created_at) }}</div>
                </div>
                <div v-if="poData.notes" class="info-item full-width">
                  <div class="info-label">Notes</div>
                  <div class="info-value">{{ poData.notes }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
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

<script>
import { ref, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',
  props: {
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
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const form = ref({
      supplier_name: '',
      quantity: 0,
      unit_cost: null,
      expected_delivery_date: '',
      notes: ''
    })
    const submitting = ref(false)
    const submitError = ref(null)

    const poData = ref(null)
    const viewLoading = ref(false)
    const viewError = ref(null)

    const resetForm = () => {
      if (!props.backlogItem) return
      form.value = {
        supplier_name: '',
        quantity: Math.max(0, props.backlogItem.quantity_needed - props.backlogItem.quantity_available),
        unit_cost: null,
        expected_delivery_date: '',
        notes: ''
      }
      submitting.value = false
      submitError.value = null
    }

    const fetchPO = async () => {
      viewLoading.value = true
      viewError.value = null
      poData.value = null
      try {
        const response = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
        poData.value = response
      } catch (err) {
        viewError.value = 'Failed to load purchase order'
        console.error(err)
      } finally {
        viewLoading.value = false
      }
    }

    watch(
      () => props.isOpen,
      (open) => {
        if (!open) return
        if (props.mode === 'create') {
          resetForm()
        } else {
          fetchPO()
        }
      }
    )

    const close = () => {
      emit('close')
    }

    const submitForm = async () => {
      if (submitting.value) return
      submitting.value = true
      submitError.value = null
      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name,
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes || undefined
        }
        const response = await api.createPurchaseOrder(payload)
        emit('po-created', response)
      } catch (err) {
        submitError.value = err?.response?.data?.detail || 'Failed to create purchase order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (value) => {
      if (value == null || isNaN(value)) return '-'
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return 'N/A'
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    const statusBadgeClass = (status) => {
      if (!status) return ''
      const s = status.toLowerCase()
      if (s === 'delivered' || s === 'completed') return 'success'
      if (s === 'pending') return 'warning'
      if (s === 'cancelled') return 'danger'
      return ''
    }

    return {
      form,
      submitting,
      submitError,
      poData,
      viewLoading,
      viewError,
      close,
      submitForm,
      formatCurrency,
      formatDate,
      statusBadgeClass
    }
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
  max-width: 560px;
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
  padding: 1.5rem;
}

.item-context {
  padding: 1rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  margin-bottom: 1.5rem;
}

.item-context-name {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.375rem;
}

.item-context-meta {
  display: flex;
  gap: 1rem;
  font-size: 0.813rem;
  color: #64748b;
}

.item-sku,
.item-order {
  font-family: 'Monaco', 'Courier New', monospace;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: flex;
  flex-direction: column;
}

.form-row.two-col {
  flex-direction: row;
  gap: 1rem;
}

.form-row.two-col .form-group {
  flex: 1;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #374151;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.optional {
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  color: #94a3b8;
}

.form-input {
  padding: 0.5rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  font-family: inherit;
  background: white;
  transition: border-color 0.15s ease;
}

.form-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-input:disabled {
  background: #f8fafc;
  color: #94a3b8;
  cursor: not-allowed;
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.total-preview {
  font-size: 0.875rem;
  color: #64748b;
  padding: 0.5rem 0.75rem;
  background: #f1f5f9;
  border-radius: 6px;
}

.total-preview strong {
  color: #0f172a;
}

.form-error {
  font-size: 0.875rem;
  color: #dc2626;
  padding: 0.625rem 0.875rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
}

.state-message {
  font-size: 0.875rem;
  color: #64748b;
  padding: 2rem;
  text-align: center;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-item.full-width {
  grid-column: 1 / -1;
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

.modal-footer {
  padding: 1.25rem 1.5rem;
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

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #0f172a;
  border: 1px solid #0f172a;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1e293b;
  border-color: #1e293b;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

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
