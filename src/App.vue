<script setup lang="ts">
import { computed, ref, watch } from 'vue'

type EntryType = 'income' | 'expense'
type PaymentFrequency = 'Monthly' | 'Weekly' | 'Biweekly' | 'Annual' | 'One-off'

interface BudgetEntry {
  id: string
  name: string
  amount: number
  category: string
  entryType: EntryType
  description: string
  paymentFrequency: PaymentFrequency
  recurring: boolean
  carriedOver: boolean
  createdAt: string
}

interface BudgetActivity {
  id: string
  title: string
  details: string
  createdAt: string
}

interface BudgetMonth {
  id: string
  label: string
  year: number
  monthIndex: number
  entries: BudgetEntry[]
  activities: BudgetActivity[]
}

const STORAGE_KEY = 'yhomoney-budget-data-v1'

const categoryOptions = [
  'Income',
  'Housing',
  'Groceries',
  'Transport',
  'Vehicle',
  'Insurance',
  'Debt repayments',
  'Family support',
  'Utilities',
  'Subscriptions',
  'Lifestyle',
  'Savings',
  'Other',
]

const monthFormatter = new Intl.DateTimeFormat('en-ZA', { month: 'long', year: 'numeric' })
const currencyFormatter = new Intl.NumberFormat('en-ZA', {
  style: 'currency',
  currency: 'ZAR',
})

const defaultForm = () => ({
  name: '',
  amount: 0,
  category: 'Housing',
  entryType: 'expense' as EntryType,
  description: '',
  paymentFrequency: 'Monthly' as PaymentFrequency,
  recurring: true,
  carriedOver: false,
})

const entryForm = ref(defaultForm())

function monthLabel(date: Date) {
  return monthFormatter.format(date)
}

function makeId(prefix: string) {
  return `${prefix}-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`
}

function seededEntriesForMonth(monthIndex: number, year: number): BudgetEntry[] {
  const isCurrentMonth = new Date().getMonth() === monthIndex && new Date().getFullYear() === year

  if (!isCurrentMonth) {
    return [
      {
        id: makeId('entry'),
        name: 'Salary',
        amount: 28000,
        category: 'Income',
        entryType: 'income',
        description: 'Primary salary deposit',
        paymentFrequency: 'Monthly',
        recurring: true,
        carriedOver: true,
        createdAt: new Date(year, monthIndex, 3).toISOString(),
      },
      {
        id: makeId('entry'),
        name: 'Rent',
        amount: 9500,
        category: 'Housing',
        entryType: 'expense',
        description: 'Apartment rental',
        paymentFrequency: 'Monthly',
        recurring: true,
        carriedOver: true,
        createdAt: new Date(year, monthIndex, 2).toISOString(),
      },
      {
        id: makeId('entry'),
        name: 'Groceries',
        amount: 3200,
        category: 'Groceries',
        entryType: 'expense',
        description: 'Food and household essentials',
        paymentFrequency: 'Monthly',
        recurring: false,
        carriedOver: true,
        createdAt: new Date(year, monthIndex, 4).toISOString(),
      },
    ]
  }

  return [
    {
      id: makeId('entry'),
      name: 'Salary',
      amount: 31000,
      category: 'Income',
      entryType: 'income',
      description: 'Primary salary deposit',
      paymentFrequency: 'Monthly',
      recurring: true,
      carriedOver: true,
      createdAt: new Date(year, monthIndex, 1).toISOString(),
    },
    {
      id: makeId('entry'),
      name: 'Rent',
      amount: 9500,
      category: 'Housing',
      entryType: 'expense',
      description: 'Apartment rental',
      paymentFrequency: 'Monthly',
      recurring: true,
      carriedOver: true,
      createdAt: new Date(year, monthIndex, 2).toISOString(),
    },
    {
      id: makeId('entry'),
      name: 'Car instalment',
      amount: 4800,
      category: 'Vehicle',
      entryType: 'expense',
      description: 'Vehicle finance payment',
      paymentFrequency: 'Monthly',
      recurring: true,
      carriedOver: true,
      createdAt: new Date(year, monthIndex, 3).toISOString(),
    },
    {
      id: makeId('entry'),
      name: 'Groceries',
      amount: 3800,
      category: 'Groceries',
      entryType: 'expense',
      description: 'Monthly groceries and household items',
      paymentFrequency: 'Monthly',
      recurring: false,
      carriedOver: false,
      createdAt: new Date(year, monthIndex, 4).toISOString(),
    },
    {
      id: makeId('entry'),
      name: 'Medical aid',
      amount: 2400,
      category: 'Insurance',
      entryType: 'expense',
      description: 'Health cover premium',
      paymentFrequency: 'Monthly',
      recurring: true,
      carriedOver: true,
      createdAt: new Date(year, monthIndex, 5).toISOString(),
    },
  ]
}

function makeMonth(year: number, monthIndex: number, entries: BudgetEntry[] = []): BudgetMonth {
  const date = new Date(year, monthIndex, 1)

  return {
    id: `${year}-${monthIndex + 1}`,
    label: monthLabel(date),
    year,
    monthIndex,
    entries,
    activities: entries.length
      ? [
          {
            id: makeId('activity'),
            title: 'Budget opened',
            details: `${entries.length} entries were loaded for this month.`,
            createdAt: new Date(year, monthIndex, 1, 9).toISOString(),
          },
        ]
      : [],
  }
}

function buildInitialMonths(): BudgetMonth[] {
  const today = new Date()
  const currentYear = today.getFullYear()
  const currentMonth = today.getMonth()

  const previousMonthDate = new Date(currentYear, currentMonth - 1, 1)
  const previousYear = previousMonthDate.getFullYear()
  const previousMonthIndex = previousMonthDate.getMonth()

  return [
    makeMonth(previousYear, previousMonthIndex, seededEntriesForMonth(previousMonthIndex, previousYear)),
    makeMonth(currentYear, currentMonth, seededEntriesForMonth(currentMonth, currentYear)),
  ]
}

function loadInitialMonths(): BudgetMonth[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return buildInitialMonths()

    const parsed = JSON.parse(raw)
    if (!Array.isArray(parsed) || parsed.length === 0) return buildInitialMonths()
    return parsed as BudgetMonth[]
  } catch {
    return buildInitialMonths()
  }
}

const months = ref<BudgetMonth[]>(loadInitialMonths())

const activeMonthId = ref(months.value[months.value.length - 1]?.id ?? '')

const activeMonthIndex = computed(() => {
  const idx = months.value.findIndex((month) => month.id === activeMonthId.value)
  return idx >= 0 ? idx : 0
})

const activeMonth = computed(() => months.value[activeMonthIndex.value] ?? months.value[0])

const totalIncome = computed(() =>
  activeMonth.value.entries
    .filter((entry) => entry.entryType === 'income')
    .reduce((total, entry) => total + Number(entry.amount || 0), 0),
)

const totalExpenses = computed(() =>
  activeMonth.value.entries
    .filter((entry) => entry.entryType === 'expense')
    .reduce((total, entry) => total + Number(entry.amount || 0), 0),
)

const remainingBalance = computed(() => totalIncome.value - totalExpenses.value)
const allocationPct = computed(() => {
  if (totalIncome.value === 0) return 0
  return (totalExpenses.value / totalIncome.value) * 100
})

const previousMonth = computed(() => {
  const index = activeMonthIndex.value - 1
  return index >= 0 ? months.value[index] : null
})

const previousIncome = computed(() => {
  if (!previousMonth.value) return 0
  return previousMonth.value.entries
    .filter((entry) => entry.entryType === 'income')
    .reduce((sum, entry) => sum + Number(entry.amount || 0), 0)
})

const previousExpenses = computed(() => {
  if (!previousMonth.value) return 0
  return previousMonth.value.entries
    .filter((entry) => entry.entryType === 'expense')
    .reduce((sum, entry) => sum + Number(entry.amount || 0), 0)
})

const comparisonSummary = computed(() => ({
  incomeDelta: totalIncome.value - previousIncome.value,
  expensesDelta: totalExpenses.value - previousExpenses.value,
  remainingDelta: remainingBalance.value - (previousIncome.value - previousExpenses.value),
}))

const suggestions = computed(() => {
  const items: string[] = []

  if (remainingBalance.value > 0) {
    items.push(
      `You have ${formatCurrency(remainingBalance.value)} left after planned expenses. Consider building a small emergency buffer or paying down costly debt first.`,
    )
  } else if (remainingBalance.value < 0) {
    items.push(
      `Your current plan is ${formatCurrency(Math.abs(remainingBalance.value))} over budget. Review the biggest recurring expenses and trim flexible categories first.`,
    )
  } else {
    items.push('Your planned spending is exactly aligned with your income this month. Keep an eye on variable expenses to protect your buffer.')
  }

  const recurringCount = activeMonth.value.entries.filter((entry) => entry.recurring).length
  if (recurringCount > 0) {
    items.push(
      `${recurringCount} recurring commitments are active this month, which makes your fixed-cost picture clearer and easier to manage.`,
    )
  } else {
    items.push('No recurring entries are flagged yet. Add your fixed monthly commitments to keep the month realistic.')
  }

  const savingsEntries = activeMonth.value.entries.filter((entry) => entry.category === 'Savings').length
  if (savingsEntries === 0) {
    items.push('No savings contribution is budgeted yet. Even a small monthly transfer can strengthen your cash cushion.')
  }

  return items.slice(0, 4)
})

function formatCurrency(value: number) {
  return currencyFormatter.format(value)
}

function updatePersistedMonths() {
  if (typeof localStorage === 'undefined') return
  localStorage.setItem(STORAGE_KEY, JSON.stringify(months.value))
}

watch(
  months,
  () => {
    updatePersistedMonths()
  },
  { deep: true },
)

function addActivity(month: BudgetMonth, title: string, details: string) {
  month.activities.unshift({
    id: makeId('activity'),
    title,
    details,
    createdAt: new Date().toISOString(),
  })
  month.activities = month.activities.slice(0, 8)
}

function handleMonthSelect(monthId: string) {
  activeMonthId.value = monthId
}

function createMonthCard() {
  const lastMonth = months.value[months.value.length - 1]
  const nextDate = new Date(lastMonth.year, lastMonth.monthIndex + 1, 1)
  const newMonth = makeMonth(nextDate.getFullYear(), nextDate.getMonth())
  months.value.push(newMonth)
  activeMonthId.value = newMonth.id
}

function addEntry() {
  const month = activeMonth.value
  if (!entryForm.value.name.trim() || Number(entryForm.value.amount) <= 0) {
    return
  }

  const newEntry: BudgetEntry = {
    id: makeId('entry'),
    name: entryForm.value.name.trim(),
    amount: Number(entryForm.value.amount),
    category: entryForm.value.category,
    entryType: entryForm.value.entryType,
    description: entryForm.value.description.trim(),
    paymentFrequency: entryForm.value.paymentFrequency,
    recurring: entryForm.value.recurring,
    carriedOver: entryForm.value.carriedOver,
    createdAt: new Date().toISOString(),
  }

  month.entries.unshift(newEntry)
  addActivity(month, 'Added budget entry', `${newEntry.name} was added at ${formatCurrency(newEntry.amount)}.`)
  entryForm.value = defaultForm()
}

function deleteEntry(entryId: string) {
  const month = activeMonth.value
  const entry = month.entries.find((item: BudgetEntry) => item.id === entryId)
  if (!entry) return

  month.entries = month.entries.filter((item: BudgetEntry) => item.id !== entryId)
  addActivity(month, 'Removed entry', `${entry.name} was deleted from the budget.`)
}

function formatDate(dateString: string) {
  return new Intl.DateTimeFormat('en-ZA', {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
  }).format(new Date(dateString))
}
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div>
        <p class="eyebrow">Personal budgeting</p>
        <h1>YhoMoney</h1>
      </div>
      <button class="primary-button" @click="createMonthCard">+ New month</button>
    </header>

    <div class="layout">
      <aside class="sidebar panel">
        <div class="panel-header">
          <h2>Monthly budgets</h2>
        </div>
        <div class="month-list">
          <button
            v-for="month in months"
            :key="month.id"
            class="month-card"
            :class="{ active: month.id === activeMonth.id }"
            @click="handleMonthSelect(month.id)"
          >
            <span class="month-label">{{ month.label }}</span>
            <span class="month-meta"
              >{{ month.entries.length }} entries</span
            >
            <strong>{{ formatCurrency(month.entries.filter((entry) => entry.entryType === 'income').reduce((sum, entry) => sum + Number(entry.amount || 0), 0)) }}</strong>
          </button>
        </div>
      </aside>

      <main class="main-panel">
        <section class="stats-grid">
          <article class="stat-card">
            <span class="label">Income</span>
            <strong>{{ formatCurrency(totalIncome) }}</strong>
          </article>
          <article class="stat-card">
            <span class="label">Planned expenses</span>
            <strong>{{ formatCurrency(totalExpenses) }}</strong>
          </article>
          <article class="stat-card accent">
            <span class="label">Remaining</span>
            <strong>{{ formatCurrency(remainingBalance) }}</strong>
          </article>
          <article class="stat-card">
            <span class="label">Allocated</span>
            <strong>{{ allocationPct.toFixed(0) }}%</strong>
          </article>
        </section>

        <section class="main-grid">
          <div class="panel form-panel">
            <div class="panel-header">
              <h2>Add budget entry</h2>
            </div>

            <form class="budget-form" @submit.prevent="addEntry">
              <div class="field-row two-col">
                <label>
                  <span>Entry name</span>
                  <input v-model="entryForm.name" type="text" placeholder="Groceries" />
                </label>
                <label>
                  <span>Amount</span>
                  <input v-model.number="entryForm.amount" type="number" min="0" step="0.01" placeholder="5000" />
                </label>
              </div>

              <div class="field-row two-col">
                <label>
                  <span>Category</span>
                  <select v-model="entryForm.category">
                    <option v-for="category in categoryOptions" :key="category" :value="category">
                      {{ category }}
                    </option>
                  </select>
                </label>
                <label>
                  <span>Type</span>
                  <select v-model="entryForm.entryType">
                    <option value="income">Income</option>
                    <option value="expense">Expense</option>
                  </select>
                </label>
              </div>

              <div class="field-row two-col">
                <label>
                  <span>Payment frequency</span>
                  <select v-model="entryForm.paymentFrequency">
                    <option value="Monthly">Monthly</option>
                    <option value="Weekly">Weekly</option>
                    <option value="Biweekly">Biweekly</option>
                    <option value="Annual">Annual</option>
                    <option value="One-off">One-off</option>
                  </select>
                </label>
                <label class="toggle">
                  <input v-model="entryForm.recurring" type="checkbox" />
                  <span>Recurring payment</span>
                </label>
              </div>

              <label>
                <span>Description</span>
                <textarea v-model="entryForm.description" rows="3" placeholder="Notes, loan description, or reminders..."></textarea>
              </label>

              <label class="toggle inline-toggle">
                <input v-model="entryForm.carriedOver" type="checkbox" />
                <span>Carried over from previous month</span>
              </label>

              <button class="primary-button full" type="submit">Save entry</button>
            </form>
          </div>

          <div class="panel history-panel">
            <div class="panel-header">
              <h2>Budget activity</h2>
            </div>
            <ul class="history-list">
              <li v-for="activity in activeMonth.activities" :key="activity.id">
                <div class="dot"></div>
                <div>
                  <strong>{{ activity.title }}</strong>
                  <p>{{ activity.details }}</p>
                </div>
                <time>{{ formatDate(activity.createdAt) }}</time>
              </li>
            </ul>
          </div>
        </section>

        <section class="bottom-grid">
          <div class="panel entries-panel">
            <div class="panel-header">
              <h2>Budget entries</h2>
            </div>
            <ul class="entry-list">
              <li v-for="entry in activeMonth.entries" :key="entry.id" class="entry-item">
                <div class="entry-main">
                  <div class="entry-badges">
                    <span class="badge" :class="entry.entryType">{{ entry.entryType }}</span>
                    <span class="badge neutral">{{ entry.category }}</span>
                    <span v-if="entry.recurring" class="badge neutral">Recurring</span>
                  </div>
                  <h3>{{ entry.name }}</h3>
                  <p v-if="entry.description">{{ entry.description }}</p>
                </div>
                <div class="entry-meta">
                  <strong>{{ formatCurrency(entry.amount) }}</strong>
                  <small>{{ entry.paymentFrequency }}</small>
                  <button class="delete-button" @click="deleteEntry(entry.id)">Remove</button>
                </div>
              </li>
            </ul>
          </div>

          <div class="panel comparison-panel">
            <div class="panel-header">
              <h2>Month comparison</h2>
            </div>

            <div v-if="previousMonth" class="comparison-box">
              <div class="comparison-row">
                <span>Previous month</span>
                <strong>{{ previousMonth.label }}</strong>
              </div>
              <div class="comparison-row">
                <span>Income change</span>
                <strong :class="comparisonSummary.incomeDelta >= 0 ? 'positive' : 'negative'">
                  {{ comparisonSummary.incomeDelta >= 0 ? '+' : '' }}{{ formatCurrency(comparisonSummary.incomeDelta) }}
                </strong>
              </div>
              <div class="comparison-row">
                <span>Expense change</span>
                <strong :class="comparisonSummary.expensesDelta <= 0 ? 'positive' : 'negative'">
                  {{ comparisonSummary.expensesDelta >= 0 ? '+' : '' }}{{ formatCurrency(comparisonSummary.expensesDelta) }}
                </strong>
              </div>
              <div class="comparison-row">
                <span>Remaining change</span>
                <strong :class="comparisonSummary.remainingDelta >= 0 ? 'positive' : 'negative'">
                  {{ comparisonSummary.remainingDelta >= 0 ? '+' : '' }}{{ formatCurrency(comparisonSummary.remainingDelta) }}
                </strong>
              </div>
            </div>
            <div v-else class="empty-state">
              Create a previous month to compare your month-to-month progress.
            </div>
          </div>
        </section>

        <section class="panel suggestions-panel">
          <div class="panel-header">
            <h2>Helpful suggestions</h2>
          </div>
          <ul class="suggestions-list">
            <li v-for="(suggestion, index) in suggestions" :key="index">
              {{ suggestion }}
            </li>
          </ul>
        </section>
      </main>
    </div>
  </div>
</template>
