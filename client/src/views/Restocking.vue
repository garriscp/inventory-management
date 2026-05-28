<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetTitle') }}</h3>
        </div>
        <input
          type="range"
          min="0"
          max="500000"
          step="1000"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="budget-display">{{ formatCurrency(budget) }}</div>
        <div class="budget-summary">
          {{ t('restocking.selectedSummary', { count: selectedItems.length, cost: formatCurrency(totalSelectedCost) }) }}
        </div>
      </div>

      <div v-if="lastSubmittedOrder" class="success-banner">
        <h4>{{ t('restocking.orderSubmitted') }}</h4>
        <p>{{ t('restocking.orderNumber') }}: {{ lastSubmittedOrder.order_number }}</p>
        <p>{{ t('restocking.expectedDelivery') }}: {{ formatDate(lastSubmittedOrder.expected_delivery) }}</p>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendationsTitle') }}</h3>
        </div>
        <p v-if="recommendations.length === 0">{{ t('restocking.noRecommendations') }}</p>
        <div v-else class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.demandGap') }}</th>
                <th>{{ t('restocking.table.qtyToOrder') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
                <th>{{ t('restocking.table.select') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in recommendations"
                :key="rec.sku"
                :style="{ background: rec.isSelected ? '#eff6ff' : '' }"
              >
                <td><strong>{{ rec.sku }}</strong></td>
                <td>{{ rec.name }}</td>
                <td>{{ rec.demandGap }}</td>
                <td>{{ rec.qtyToOrder }}</td>
                <td>{{ formatCurrency(rec.unitCost) }}</td>
                <td>{{ formatCurrency(rec.totalCost) }}</td>
                <td>
                  <input
                    type="checkbox"
                    :checked="rec.isSelected"
                    @change="toggleItem(rec.sku)"
                    class="checkbox"
                  />
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-if="selectedItems.length > 0" class="action-row">
        <button
          @click="placeOrder"
          :disabled="submitting"
          class="btn-primary"
        >
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentLocale, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const inventoryMap = ref({})
    const budget = ref(25000)
    const submitting = ref(false)
    const lastSubmittedOrder = ref(null)
    const manuallyDeselected = ref(new Set())

    const candidateItems = computed(() => {
      const increasing = forecasts.value.filter(f => f.trend === 'increasing')

      const withCosts = increasing.map(f => {
        const invItem = inventoryMap.value[f.item_sku]
        const unitCost = invItem?.unit_cost || 0
        const demandGap = f.forecasted_demand - f.current_demand
        const qtyToOrder = Math.max(demandGap, 1)
        const totalCost = qtyToOrder * unitCost

        return {
          sku: f.item_sku,
          name: f.item_name,
          demandGap,
          qtyToOrder,
          unitCost,
          totalCost
        }
      })

      return withCosts.sort((a, b) => b.demandGap - a.demandGap)
    })

    const autoSelectedSkus = computed(() => {
      const selected = new Set()
      let runningTotal = 0

      for (const item of candidateItems.value) {
        if (manuallyDeselected.value.has(item.sku)) continue
        if (runningTotal + item.totalCost > budget.value) break
        selected.add(item.sku)
        runningTotal += item.totalCost
      }

      return selected
    })

    const recommendations = computed(() => {
      return candidateItems.value.map(item => ({
        ...item,
        isSelected: autoSelectedSkus.value.has(item.sku)
      }))
    })

    const selectedItems = computed(() => {
      return recommendations.value.filter(r => r.isSelected)
    })

    const totalSelectedCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.totalCost, 0)
    })

    const toggleItem = (sku) => {
      const newSet = new Set(manuallyDeselected.value)
      if (newSet.has(sku)) {
        newSet.delete(sku)
      } else {
        newSet.add(sku)
      }
      manuallyDeselected.value = newSet
    }

    const placeOrder = async () => {
      submitting.value = true
      try {
        const orderData = {
          customer: 'Restocking System',
          items: selectedItems.value.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.qtyToOrder,
            unit_price: i.unitCost
          }))
        }
        const result = await api.createOrder(orderData)
        lastSubmittedOrder.value = result

        // Reset selections after successful order
        manuallyDeselected.value = new Set()
      } catch (err) {
        alert(t('restocking.orderFailed'))
        console.error('Failed to place order:', err)
      } finally {
        submitting.value = false
      }
    }

    const formatDate = (dateString) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return new Date(dateString).toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const formatCurrency = (value) => {
      const currencySymbol = currentCurrency.value === 'JPY' ? '¥' : '$'
      return currencySymbol + value.toLocaleString()
    }

    watch(budget, () => {
      manuallyDeselected.value = new Set()
      lastSubmittedOrder.value = null
    })

    onMounted(async () => {
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])

        forecasts.value = forecastData

        inventoryData.forEach(item => {
          inventoryMap.value[item.sku] = item
        })
      } catch (err) {
        error.value = 'Failed to load data'
        console.error(err)
      } finally {
        loading.value = false
      }
    })

    return {
      t,
      loading,
      error,
      budget,
      recommendations,
      selectedItems,
      totalSelectedCost,
      toggleItem,
      placeOrder,
      submitting,
      lastSubmittedOrder,
      formatDate,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0.5rem 0;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  margin: 1rem 0;
}

.budget-summary {
  color: #64748b;
  font-size: 0.875rem;
  margin-top: 0.5rem;
}

.recommendations-table {
  width: 100%;
  border-collapse: collapse;
}

.recommendations-table tbody tr {
  transition: background-color 0.15s ease;
}

.checkbox {
  accent-color: #2563eb;
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.action-row {
  display: flex;
  gap: 1rem;
  align-items: center;
  margin-top: 1.25rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  border: none;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem;
  color: #065f46;
  margin-bottom: 1.25rem;
}

.success-banner h4 {
  font-weight: 700;
  margin-bottom: 0.5rem;
}

.success-banner p {
  font-size: 0.875rem;
  margin: 0.25rem 0;
}
</style>
