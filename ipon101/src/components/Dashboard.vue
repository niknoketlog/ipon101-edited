<template>
  <div>
    <h1 class="h3 mb-4">Dashboard</h1>

    <div class="row">
      <div class="col-md-4 mb-3">
        <div class="card text-white bg-success shadow-sm">
          <div class="card-body">
            <h5 class="card-title">Total Income</h5>
            <p class="card-text fs-4">₱ {{ summary.income.toFixed(2) }}</p>
          </div>
        </div>
      </div>

      <div class="col-md-4 mb-3">
        <div class="card text-white bg-danger shadow-sm">
          <div class="card-body">
            <h5 class="card-title">Total Expense</h5>
            <p class="card-text fs-4">₱ {{ summary.expense.toFixed(2) }}</p>
          </div>
        </div>
      </div>

      <div class="col-md-4 mb-3">
        <div class="card shadow-sm" :class="summary.balance >= 0 ? 'bg-primary text-white' : 'bg-warning'">
          <div class="card-body">
            <h5 class="card-title">Balance</h5>
            <p class="card-text fs-4">₱ {{ summary.balance.toFixed(2) }}</p>
          </div>
        </div>
      </div>
    </div>
    <h3 class="h4 mt-4 mb-3">Recent Transactions (Last 8)</h3>
    
    <div class="card shadow-sm">
      <ul class="list-group list-group-flush">
        <li v-for="t in recent" :key="t.id" class="list-group-item d-flex justify-content-between align-items-center">
          <span>
            <span class="fw-bold">{{ t.transactionDate || t.createdAt.slice(0,10) }}</span> — {{ t.description }}
          </span>
          <span :class="t.type === 'income' ? 'text-success fw-bold' : 'text-danger fw-bold'">
            {{ t.type === 'income' ? '+' : '-' }} ₱ {{ t.amount }}
          </span>
        </li>
      </ul>
    </div>
    <div v-if="error" class="alert alert-danger mt-4">{{ errorMessage }}</div>

    </div>
</template>

<script setup>
import { ref, onMounted, computed } from "vue";
import transactionService from "../services/transactionService";
import { getToken, clearAuth } from "../utils/auth";
import { useRouter } from "vue-router";

const transactions = ref([]);
const error = ref(null);
const router = useRouter();

const errorMessage = computed(() => (error.value ? (error.value.message || String(error.value)) : ""));

async function load() {
  error.value = null;
  try {
    if (!getToken()) { router.push("/login"); return; }
    transactions.value = await transactionService.list();
  } catch (e) {
    error.value = e;
    if (e?.response?.status === 401) {
      clearAuth();
      router.push("/login");
    }
  }
}

onMounted(load);

const summary = computed(() => {
  const income = transactions.value.filter(i => i.type === "income").reduce((s,v)=>s+Number(v.amount||0),0);
  const expense = transactions.value.filter(i => i.type === "expense").reduce((s,v)=>s+Number(v.amount||0),0);
  return { income, expense, balance: income-expense };
});

const recent = computed(() => transactions.value.slice(0,8));

// Removed the redundant logout function
</script>