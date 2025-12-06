<template>
  <div class="dashboard-page">
    <!-- Header -->
    <header class="d-flex justify-content-between align-items-start flex-wrap gap-2 mb-4">
      <div>
        <h1 class="page-title mb-1">Dashboard</h1>
        <p class="page-subtitle mb-0">
          Overview of your income and expenses over time.
        </p>
      </div>

      <div class="d-flex flex-column align-items-end gap-2">
        <span v-if="lastUpdated" class="badge last-updated-badge">
          Updated: {{ lastUpdated }}
        </span>
        <span v-if="savingsRateDisplay !== null" class="badge savings-badge">
          Savings rate: {{ savingsRateDisplay }}%
        </span>
      </div>
    </header>

    <!-- Summary cards: running totals (wallet behavior) -->
    <section class="row g-3 mb-3">
      <div class="col-md-4">
        <div class="card summary-card h-100">
          <div class="card-body">
            <p class="summary-label mb-1">Total Income</p>
            <h3 class="summary-value text-income mb-0">
              ₱{{ totalIncome.toLocaleString() }}
            </h3>
          </div>
        </div>
      </div>

      <div class="col-md-4">
        <div class="card summary-card h-100">
          <div class="card-body">
            <p class="summary-label mb-1">Total Expenses</p>
            <h3 class="summary-value text-expense mb-0">
              ₱{{ totalExpense.toLocaleString() }}
            </h3>
          </div>
        </div>
      </div>

      <div class="col-md-4">
        <div class="card summary-card h-100">
          <div class="card-body">
            <p class="summary-label mb-1">Balance</p>
            <h3
              class="summary-value mb-1"
              :class="balance >= 0 ? 'text-balance-positive' : 'text-balance-negative'"
            >
              ₱{{ balance.toLocaleString() }}
            </h3>
            <small class="text-muted" v-if="savingsMessage">
              {{ savingsMessage }}
            </small>
          </div>
        </div>
      </div>
    </section>

    <!-- Range toggle + info (affects chart only) -->
    <section class="d-flex justify-content-between align-items-center flex-wrap gap-2 mb-3">
      <div class="small text-muted">
        Expense breakdown based on
        <span class="fw-semibold">{{ rangeLabel.toLowerCase() }}</span> range.
      </div>
      <div class="btn-group range-toggle" role="group">
        <button
          type="button"
          class="btn btn-sm"
          :class="range === 'daily' ? 'btn-primary' : 'btn-outline-primary'"
          @click="setRange('daily')"
        >
          Daily
        </button>
        <button
          type="button"
          class="btn btn-sm"
          :class="range === 'weekly' ? 'btn-primary' : 'btn-outline-primary'"
          @click="setRange('weekly')"
        >
          Weekly
        </button>
        <button
          type="button"
          class="btn btn-sm"
          :class="range === 'monthly' ? 'btn-primary' : 'btn-outline-primary'"
          @click="setRange('monthly')"
        >
          Monthly
        </button>
      </div>
    </section>

    <!-- Chart + recent list -->
    <section class="row g-4 align-items-stretch">
      <!-- Expense breakdown doughnut chart -->
      <div class="col-lg-8">
        <div class="card panel-card h-100">
          <div class="card-body d-flex flex-column">
            <div class="d-flex justify-content-between align-items-center flex-wrap gap-2 mb-2">
              <div>
                <h5 class="card-title mb-1">Expense Breakdown</h5>
                <small class="text-muted">
                  Each slice shows where your money went for the selected range.
                </small>
              </div>
            </div>

            <div v-if="loading" class="empty-state flex-grow-1 d-flex align-items-center justify-content-center">
              Loading chart…
            </div>

            <div
              v-else-if="!pieHasData"
              class="empty-state flex-grow-1 d-flex align-items-center justify-content-center"
            >
              No expenses yet for this range.
            </div>

            <div
              v-else
              class="d-flex flex-grow-1 align-items-center justify-content-center chart-wrapper"
            >
              <div class="position-relative chart-container">
                <canvas ref="chartCanvas"></canvas>
                <div class="chart-center-text">
                  <div class="label">Balance</div>
                  <div
                    class="value"
                    :class="balance >= 0 ? 'text-balance-positive' : 'text-balance-negative'"
                  >
                    ₱{{ balance.toLocaleString() }}
                  </div>
                </div>
              </div>
            </div>

            <div v-if="pieHasData" class="mt-3 small text-muted">
              <span class="fw-semibold">Tip:</span> Hover on a slice to see the amount and percentage.
            </div>
          </div>
        </div>
      </div>

      <!-- Recent transactions (latest 5) -->
      <div class="col-lg-4">
        <div class="card panel-card h-100">
          <div class="card-body d-flex flex-column">
            <h5 class="card-title mb-3">Recent Transactions</h5>

            <p
              v-if="recentTransactions.length === 0"
              class="text-muted mb-0 empty-state"
            >
              No transactions yet. Start adding income or expenses!
            </p>

            <ul
              v-else
              class="list-group list-group-flush recent-list flex-grow-1"
            >
              <li
                v-for="t in recentTransactions"
                :key="t.id"
                class="list-group-item d-flex justify-content-between align-items-center recent-item"
              >
                <div class="me-2">
                  <div class="recent-desc">
                    {{ t.description || t.category || "Transaction" }}
                  </div>
                  <small class="text-muted">
                    {{ formattedTxDate(t) }}
                  </small>
                </div>
                <span
                  class="fw-bold"
                  :class="(t.type || t.transaction_type) === 'income'
                    ? 'text-income'
                    : 'text-expense'"
                >
                  {{ (t.type || t.transaction_type) === "income" ? "+" : "-" }}
                  ₱{{ Number(t.amount).toLocaleString() }}
                </span>
              </li>
            </ul>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, onBeforeUnmount, watch, nextTick } from "vue";
import axios from "axios";
import Chart from "chart.js/auto";

const chartCanvas = ref(null);
let chartInstance = null;

const transactions = ref([]);
const loading = ref(true);
const lastUpdated = ref("");
const error = ref("");

const range = ref("daily"); // "daily" | "weekly" | "monthly"

const API_BASE = import.meta.env.VITE_API_BASE_URL || "http://localhost:3000";

const fetchTransactions = async () => {
  try {
    loading.value = true;
    const token = localStorage.getItem("auth_token");
    const res = await axios.get(`${API_BASE}/api/transactions`, {
      headers: token ? { Authorization: `Bearer ${token}` } : {},
    });
    transactions.value = Array.isArray(res.data) ? res.data : [];
    lastUpdated.value = new Date().toLocaleString();
    await nextTick();
    buildChart();
  } catch (e) {
    console.error(e);
    error.value = "Failed to load transactions.";
  } finally {
    loading.value = false;
  }
};

const setRange = (r) => {
  range.value = r;
};

// --- DATE HELPERS ---
const getRawDate = (t) =>
  t.transactionDate ||
  t.transaction_date ||
  t.date ||
  t.createdAt ||
  t.created_at ||
  null;

const parseTxDate = (t) => {
  const raw = getRawDate(t);
  if (!raw) return null;
  const d = new Date(raw);
  if (isNaN(d)) return null;
  return d;
};

// --- RUNNING TOTALS (wallet logic) ---
const totalIncome = computed(() =>
  transactions.value
    .filter((t) => (t.type || t.transaction_type) === "income")
    .reduce((sum, t) => sum + Number(t.amount || 0), 0)
);

const totalExpense = computed(() =>
  transactions.value
    .filter((t) => (t.type || t.transaction_type) !== "income")
    .reduce((sum, t) => sum + Number(t.amount || 0), 0)
);

const balance = computed(() => totalIncome.value - totalExpense.value);

// savings rate & message based on running totals
const savingsRate = computed(() => {
  if (totalIncome.value <= 0) return null;
  const rate = balance.value / totalIncome.value;
  return rate < 0 ? 0 : rate;
});

const savingsRateDisplay = computed(() => {
  if (savingsRate.value === null) return null;
  return Math.round(savingsRate.value * 100);
});

const savingsMessage = computed(() => {
  if (savingsRate.value === null) return "";
  const r = savingsRate.value;
  if (r >= 0.5) return "Amazing! You're saving more than half of what you earn.";
  if (r >= 0.25) return "Nice! You're saving a healthy part of your income.";
  if (r > 0) return "You’re saving a bit — keep going.";
  if (balance.value === 0) return "You’re breaking even right now.";
  return "You’re spending more than you earn. Time to adjust a bit.";
});

// --- RANGE LABEL ---
const rangeLabel = computed(() => {
  if (range.value === "daily") return "Daily";
  if (range.value === "weekly") return "Weekly";
  return "Monthly";
});

// --- RECENT TRANSACTIONS (latest 5) ---
const recentTransactions = computed(() =>
  [...transactions.value]
    .sort((a, b) => {
      const da = parseTxDate(a) || new Date(0);
      const db = parseTxDate(b) || new Date(0);
      return db - da;
    })
    .slice(0, 5)
);

const formattedTxDate = (t) => {
  const d = parseTxDate(t);
  if (!d) return "";
  return d.toLocaleDateString(undefined, {
    month: "short",
    day: "numeric",
    year: "numeric",
  });
};

// --- EXPENSES FOR CHART ---
// Daily = latest day that has expenses
// Weekly = last 7 days
// Monthly = current month
const filteredExpensesForChart = computed(() => {
  const all = transactions.value || [];
  const expenses = all.filter(
    (t) => (t.type || t.transaction_type || "expense") !== "income"
  );
  if (!expenses.length) return [];

  const withDates = expenses
    .map((t) => ({ t, d: parseTxDate(t) }))
    .filter((x) => x.d);

  if (!withDates.length) return [];

  const now = new Date();

  // find latest expense day for "daily" so it's never visually empty
  const latest = new Date(
    Math.max(
      ...withDates.map((x) => x.d.getTime())
    )
  );

  return withDates
    .filter(({ d }) => {
      if (range.value === "daily") {
        return d.toDateString() === latest.toDateString();
      }

      if (range.value === "weekly") {
        const diffMs = now - d;
        const diffDays = diffMs / (1000 * 60 * 60 * 24);
        return diffDays <= 7;
      }

      // monthly → same calendar month
      return (
        d.getFullYear() === now.getFullYear() &&
        d.getMonth() === now.getMonth()
      );
    })
    .map((x) => x.t);
});

// group filtered expenses by description / category
const expenseBreakdown = computed(() => {
  const map = new Map();

  filteredExpensesForChart.value.forEach((t) => {
    const label =
      (t.description && t.description.trim()) ||
      (t.category && t.category.trim()) ||
      "Other";

    const amount = Number(t.amount) || 0;
    if (!amount) return;

    if (!map.has(label)) {
      map.set(label, 0);
    }
    map.set(label, map.get(label) + amount);
  });

  return Array.from(map.entries()).map(([label, amount]) => ({
    label,
    amount,
  }));
});

const pieHasData = computed(() => expenseBreakdown.value.length > 0);

// --- CHART BUILDING ---
const buildChart = () => {
  const breakdown = expenseBreakdown.value;
  if (!chartCanvas.value || !breakdown.length) {
    if (chartInstance) {
      chartInstance.destroy();
      chartInstance = null;
    }
    return;
  }

  const labels = breakdown.map((b) => b.label);
  const data = breakdown.map((b) => b.amount);
  const total = data.reduce((s, v) => s + v, 0);

  const baseColors = [
    "#10b981",
    "#22c55e",
    "#0ea5e9",
    "#6366f1",
    "#f97316",
    "#ec4899",
    "#a855f7",
    "#eab308",
  ];
  const backgroundColors = labels.map(
    (_, i) => baseColors[i % baseColors.length]
  );

  if (chartInstance) {
    chartInstance.destroy();
  }

  chartInstance = new Chart(chartCanvas.value, {
    type: "doughnut",
    data: {
      labels,
      datasets: [
        {
          data,
          backgroundColor: backgroundColors,
          borderColor: "#ffffff",
          borderWidth: 2,
          hoverOffset: 8,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      cutout: "65%",
      plugins: {
        legend: {
          position: "bottom",
          labels: {
            usePointStyle: true,
            boxWidth: 10,
          },
        },
        tooltip: {
          callbacks: {
            label: (ctx) => {
              const label = ctx.label || "";
              const value = ctx.parsed || 0;
              const percent = total
                ? ((value / total) * 100).toFixed(1)
                : 0;
              return `${label}: ₱${Number(value).toLocaleString()} (${percent}%)`;
            },
          },
        },
      },
      animation: {
        duration: 900,
        easing: "easeOutCubic",
      },
    },
  });
};

onMounted(async () => {
  await fetchTransactions();
});

// rebuild chart when range or data changes, and wait for DOM before drawing
watch([range, transactions], async () => {
  await nextTick();
  buildChart();
});

onBeforeUnmount(() => {
  if (chartInstance) chartInstance.destroy();
});
</script>

<style scoped>
.dashboard-page {
  padding-top: 0.5rem;
  padding-bottom: 2rem;
}

.page-title {
  font-weight: 700;
  font-size: 1.9rem;
  letter-spacing: 0.02em;
}

.page-subtitle {
  color: #6b7280;
  font-size: 0.95rem;
}

.last-updated-badge {
  background-color: #f9fafb;
  color: #6b7280;
  border: 1px solid #e5e7eb;
  font-size: 0.8rem;
  padding: 0.4rem 0.6rem;
  border-radius: 999px;
}

.savings-badge {
  background-color: #ecfdf5;
  color: #047857;
  border: 1px solid #bbf7d0;
  font-size: 0.8rem;
  padding: 0.4rem 0.6rem;
  border-radius: 999px;
}

/* Summary cards */
.summary-card {
  border-radius: 18px;
  border: 1px solid #e5e7eb;
  box-shadow: 0 4px 14px rgba(15, 23, 42, 0.06);
  background-color: #ffffff;
}

.summary-label {
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #9ca3af;
}

.summary-value {
  font-weight: 700;
  font-size: 1.6rem;
}

/* Custom text colors */
.text-income {
  color: #16a34a;
}

.text-expense {
  color: #dc2626;
}

.text-balance-positive {
  color: #0f766e;
}

.text-balance-negative {
  color: #b91c1c;
}

/* Shared panel card style (chart + recent box) */
.panel-card {
  border-radius: 20px;
  border: 1px solid #e5e7eb;
  box-shadow: 0 8px 24px rgba(15, 23, 42, 0.06);
  background-color: #ffffff;
}

/* Range toggle buttons */
.range-toggle .btn {
  min-width: 70px;
  border-radius: 999px !important;
}

/* Empty states */
.empty-state {
  font-size: 0.9rem;
  color: #9ca3af;
  text-align: center;
}

/* Recent list */
.recent-list {
  margin-top: -0.25rem;
}

.recent-item {
  border: none;
  padding-left: 0;
  padding-right: 0;
  border-bottom: 1px solid #e5e7eb;
}

.recent-item:last-child {
  border-bottom: none;
}

.recent-desc {
  font-weight: 500;
}

/* Doughnut chart wrapper */
.chart-wrapper {
  width: 100%;
}

.chart-container {
  position: relative;
  width: 100%;
  max-width: 460px;
  margin: 0 auto;
  height: 360px;
}

.chart-center-text {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.chart-center-text .label {
  font-size: 0.8rem;
  color: #9ca3af;
  text-transform: uppercase;
  letter-spacing: 0.12em;
}

.chart-center-text .value {
  font-weight: 700;
  font-size: 1.6rem;
}

/* Chart canvas interaction */
canvas {
  transition: transform 0.18s ease-out, box-shadow 0.18s ease-out;
  border-radius: 14px;
}

canvas:hover {
  transform: scale(1.01);
  box-shadow: 0 8px 22px rgba(15, 23, 42, 0.16);
}
</style>
