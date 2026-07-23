<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue';
import Chart from 'chart.js/auto';

const fmt = (n) => Number(n).toLocaleString('fr-FR');

/* --- État --- */
const products = ref(JSON.parse(localStorage.getItem('products')) || []);
const search = ref('');
const filter = ref('all');
const sort = ref('');
const volume = ref(1);
const toasts = ref([]);

/* --- Persistance --- */
watch(products, () => {
  localStorage.setItem('products', JSON.stringify(products.value));
  redraw();
}, { deep: true });

/* --- Calcul --- */
function calc(p) {
  const pa = Number(p.pa) || 0;
  const cost = pa + (Number(p.pub) || 0) + (Number(p.liv) || 0) + (Number(p.autre) || 0);
  const profit = (Number(p.pv) || 0) - cost;
  const roi = pa ? (profit / pa) * 100 : 0;
  const status = profit <= 0 ? 'bad' : roi < 30 ? 'warn' : 'good';
  return { profit, roi, status };
}

/* --- Lignes affichées (recherche + tri + filtre) --- */
const rows = computed(() => {
  let data = products.value.map((p, i) => ({ p, i }));
  data = data.filter(({ p }) => (p.name || '').toLowerCase().includes(search.value.toLowerCase()));
  if (sort.value === 'profit') data.sort((a, b) => calc(b.p).profit - calc(a.p).profit);
  if (sort.value === 'roi') data.sort((a, b) => calc(b.p).roi - calc(a.p).roi);
  return data.filter(({ p }) => filter.value === 'all' || calc(p).status === filter.value);
});

/* --- KPIs (sur les lignes affichées, comme l'original) --- */
const stats = computed(() => {
  let totalProfit = 0, roiSum = 0, good = 0;
  rows.value.forEach(({ p }) => {
    const c = calc(p);
    totalProfit += c.profit;
    roiSum += c.roi;
    if (c.status === 'good') good++;
  });
  const roi = products.value.length ? roiSum / products.value.length : 0;
  return { count: products.value.length, profit: totalProfit, roi, good };
});

/* --- Actions --- */
function addProduct() {
  products.value.push({ name: '', pa: 0, pv: 0, pub: 0, liv: 0, autre: 0, notes: '' });
  toast('Produit ajouté — remplis les champs');
}
function del(i) {
  if (confirm('Supprimer ce produit ?')) {
    products.value.splice(i, 1);
    toast('Produit supprimé', 'error');
  }
}
function toast(msg, type = 'success') {
  const id = `${performance.now()}-${Math.round(performance.now() % 1000)}`;
  toasts.value.push({ id, msg, type, exiting: false });
  setTimeout(() => { const t = toasts.value.find((x) => x.id === id); if (t) t.exiting = true; }, 2500);
  setTimeout(() => { toasts.value = toasts.value.filter((x) => x.id !== id); }, 2700);
}
function exportCSV() {
  if (products.value.length === 0) { toast('Aucun produit à exporter', 'error'); return; }
  let csv = 'Nom,PA,PV,Pub,Livraison,Autres,Bénéfice,ROI,Notes\n';
  products.value.forEach((p) => {
    const c = calc(p);
    csv += `"${p.name}",${p.pa},${p.pv},${p.pub},${p.liv},${p.autre},${c.profit},${c.roi.toFixed(1)},"${p.notes}"\n`;
  });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(new Blob([csv], { type: 'text/csv' }));
  a.download = 'business_' + new Date().toISOString().slice(0, 10) + '.csv';
  a.click();
  toast('CSV exporté !');
}

/* --- Charts --- */
const profitCanvas = ref(null);
const statusCanvas = ref(null);
let profitChart, statusChart;

function drawCharts() {
  if (!profitCanvas.value || !statusCanvas.value) return;
  const profits = products.value.map((p) => calc(p).profit);
  const names = products.value.map((p) => p.name || 'Sans nom');
  const statusCount = { good: 0, warn: 0, bad: 0 };
  products.value.forEach((p) => statusCount[calc(p).status]++);
  const chartText = '#94a3b8';
  const chartGrid = '#1e293b';

  if (profitChart) profitChart.destroy();
  profitChart = new Chart(profitCanvas.value, {
    type: 'bar',
    data: {
      labels: names,
      datasets: [{
        label: 'Bénéfice (F)',
        data: profits,
        backgroundColor: profits.map((p) => (p >= 0 ? 'rgba(16,185,129,0.6)' : 'rgba(239,68,68,0.6)')),
        borderRadius: 6,
      }],
    },
    options: {
      responsive: true,
      plugins: { legend: { display: false } },
      scales: {
        x: { ticks: { color: chartText }, grid: { display: false } },
        y: { ticks: { color: chartText }, grid: { color: chartGrid } },
      },
    },
  });

  if (statusChart) statusChart.destroy();
  statusChart = new Chart(statusCanvas.value, {
    type: 'doughnut',
    data: {
      labels: ['Rentables', 'ROI faible', 'En perte'],
      datasets: [{
        data: [statusCount.good, statusCount.warn, statusCount.bad],
        backgroundColor: ['#10b981', '#f59e0b', '#ef4444'],
        borderWidth: 0,
      }],
    },
    options: {
      responsive: true,
      cutout: '65%',
      plugins: { legend: { labels: { color: chartText, padding: 16 } } },
    },
  });
}

let redrawScheduled = false;
function redraw() {
  if (redrawScheduled) return;
  redrawScheduled = true;
  nextTick(() => { redrawScheduled = false; drawCharts(); });
}

onMounted(drawCharts);
</script>

<template>
  <header class="header">
    <h1>📊 Business Dashboard PRO</h1>
    <div class="header-actions">
      <button class="btn btn-ghost" @click="exportCSV">⬇ Export CSV</button>
      <button class="btn btn-primary" @click="addProduct">+ Produit</button>
    </div>
  </header>

  <main class="main">
    <!-- KPI -->
    <section class="kpi-grid fade-in">
      <div class="kpi">
        <div class="kpi-icon purple">📦</div>
        <div class="kpi-label">Produits</div>
        <div class="kpi-value">{{ stats.count }}</div>
      </div>
      <div class="kpi">
        <div class="kpi-icon green">💰</div>
        <div class="kpi-label">Bénéfice total</div>
        <div class="kpi-value">{{ fmt(stats.profit) }} F</div>
      </div>
      <div class="kpi">
        <div class="kpi-icon yellow">📈</div>
        <div class="kpi-label">ROI moyen</div>
        <div class="kpi-value">{{ stats.roi.toFixed(1) }}%</div>
      </div>
      <div class="kpi">
        <div class="kpi-icon red">✅</div>
        <div class="kpi-label">Rentables</div>
        <div class="kpi-value">{{ stats.good }}</div>
      </div>
    </section>

    <!-- Controls -->
    <section class="controls fade-in">
      <input class="input" placeholder="🔍 Rechercher un produit..." v-model="search">
      <select class="select" v-model="filter">
        <option value="all">Tous les statuts</option>
        <option value="good">Rentables</option>
        <option value="warn">ROI faible</option>
        <option value="bad">En perte</option>
      </select>
      <select class="select" v-model="sort">
        <option value="">Trier par...</option>
        <option value="profit">Bénéfice</option>
        <option value="roi">ROI</option>
      </select>
    </section>

    <!-- Table -->
    <section class="card fade-in">
      <div class="card-title">Liste des produits</div>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Produit</th><th>Prix achat</th><th>Prix vente</th><th>Pub</th>
              <th>Livraison</th><th>Autres</th><th>Bénéfice</th><th>ROI</th>
              <th>Notes</th><th>Statut</th><th></th>
            </tr>
          </thead>
          <tbody>
            <template v-if="rows.length">
              <tr v-for="item in rows" :key="item.i">
                <td><input class="cell-input name" v-model="item.p.name" placeholder="Nom..."></td>
                <td><input class="cell-input narrow" type="number" v-model.number="item.p.pa"></td>
                <td><input class="cell-input narrow" type="number" v-model.number="item.p.pv"></td>
                <td><input class="cell-input narrow" type="number" v-model.number="item.p.pub"></td>
                <td><input class="cell-input narrow" type="number" v-model.number="item.p.liv"></td>
                <td><input class="cell-input narrow" type="number" v-model.number="item.p.autre"></td>
                <td class="profit-cell" :class="calc(item.p).profit * (parseInt(volume) || 1) >= 0 ? 'positive' : 'negative'">
                  {{ fmt(calc(item.p).profit * (parseInt(volume) || 1)) }} F
                </td>
                <td>{{ calc(item.p).roi.toFixed(1) }}%</td>
                <td><textarea class="cell-notes" v-model="item.p.notes" placeholder="..."></textarea></td>
                <td>
                  <span class="badge" :class="calc(item.p).status === 'good' ? 'badge-good' : calc(item.p).status === 'warn' ? 'badge-warn' : 'badge-bad'">
                    {{ calc(item.p).status === 'good' ? 'Rentable' : calc(item.p).status === 'warn' ? 'Faible' : 'Perte' }}
                  </span>
                </td>
                <td><button class="btn btn-danger-sm" @click="del(item.i)">✕</button></td>
              </tr>
            </template>
            <tr v-else>
              <td colspan="11" class="empty">
                <div class="empty-icon">📭</div>
                {{ products.length === 0 ? 'Aucun produit — clique "+ Produit" pour commencer' : 'Aucun résultat' }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- Simulator -->
    <section class="card fade-in">
      <div class="card-title">Simulateur de volume</div>
      <div class="simulator">
        <label>Nombre de ventes :</label>
        <input class="input" type="number" placeholder="ex: 100" v-model="volume">
      </div>
    </section>

    <!-- Charts -->
    <section class="chart-grid fade-in">
      <div class="chart-card">
        <div class="card-title">Bénéfice par produit</div>
        <canvas ref="profitCanvas"></canvas>
      </div>
      <div class="chart-card">
        <div class="card-title">Répartition statuts</div>
        <canvas ref="statusCanvas"></canvas>
      </div>
    </section>
  </main>

  <div class="toast-container">
    <div v-for="t in toasts" :key="t.id" class="toast" :class="[t.type, { 'toast-exit': t.exiting }]">{{ t.msg }}</div>
  </div>
</template>
