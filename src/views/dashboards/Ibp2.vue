<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue';

const API_BASE = 'https://lubx-api.lubconsulta.com.br/bi/ipb';
const MONTHS = [
  { short: 'Jan', order: 1 },
  { short: 'Fev', order: 2 },
  { short: 'Mar', order: 3 },
  { short: 'Abr', order: 4 },
  { short: 'Mai', order: 5 },
  { short: 'Jun', order: 6 },
  { short: 'Jul', order: 7 },
  { short: 'Ago', order: 8 },
  { short: 'Set', order: 9 },
  { short: 'Out', order: 10 },
  { short: 'Nov', order: 11 },
  { short: 'Dez', order: 12 },
];
const SHARE_TICKS = [1, 2, 3, 4, 5, 10, 15, 20, 25, 100];
const SHARE_STOPS = [0, ...SHARE_TICKS];
const CHART_TOP = 24;
const CHART_BOTTOM = 280;
const CHART_HEIGHT = CHART_BOTTOM - CHART_TOP;
const EXCLUDED_EMPRESAS = ['CHEVRON', 'IPIRANGA'];

const marcasConfig = {
  TOTALENERGIES: { cor: '#f58220', logo: 'totalenergies' },
  YPF: { cor: '#00529b', logo: 'ypf' },
  VIBRA: { cor: '#004415', logo: 'vibra' },
  PETRONAS: { cor: '#00A79D', logo: 'petronas' },
  CASTROL: { cor: '#E10600', logo: 'castrol' },
  VALVOLINE: { cor: '#2FA4E7', logo: 'valvoline' },
  'RAIZEN LUBRIFICANTES': { cor: '#9B1BB3', logo: 'raizen' },
  RAIZEN: { cor: '#9B1BB3', logo: 'raizen' },
  MOOVE: { cor: '#00A3E0', logo: 'moove' },
  LWART: { cor: '#111111', logo: 'lwart' },
  ICONIC: { cor: '#F56600', logo: 'iconic' },
  CHEVRON: { cor: '#005da4', logo: 'chevron' },
  IPIRANGA: { cor: '#ffcb05', logo: 'ipiranga' },
};

const loading = ref(true);
const fetchError = ref('');
const filtrosData = ref({ empresas: [], anos: [], segmentos: [] });
const yearlyDashboards = ref({});
const visibleEmpresas = ref([]);
const hover = ref({ visible: false, x: 0, y: 0, title: '', value: '' });

const state = reactive({
  empresa: 'TODAS',
  segmentos: ['REVENDEDOR', 'CONSUMIDOR'],
  periodDe: '',
  periodAte: '',
  periodDirty: false,
});

const buildQuery = (params) => {
  const query = new URLSearchParams();
  Object.entries(params).forEach(([key, value]) => {
    if (value !== undefined && value !== null && value !== '') {
      query.append(key, value);
    }
  });
  return query.toString();
};

const comparePeriods = (a, b) => {
  if (!a || !b) return 0;
  return a.localeCompare(b);
};

const addMonths = (period, offset) => {
  const [yearRaw, monthRaw] = period.split('-');
  const date = new Date(Number(yearRaw), Number(monthRaw) - 1 + offset, 1);
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}`;
};

const monthKeyFromDate = (date) => (date ? String(date).slice(0, 7) : '');
const startOfMonth = (period) => (period ? `${period}-01` : '');
const endOfMonth = (period) => {
  if (!period) return '';
  const [year, month] = period.split('-');
  const lastDay = new Date(Number(year), Number(month), 0).getDate();
  return `${period}-${String(lastDay).padStart(2, '0')}`;
};

const listPeriods = (from, to) => {
  if (!from || !to || comparePeriods(from, to) > 0) return [];
  const periods = [];
  let cursor = from;
  while (comparePeriods(cursor, to) <= 0) {
    periods.push(cursor);
    cursor = addMonths(cursor, 1);
  }
  return periods;
};

const monthLabel = (period) => {
  if (!period) return '';
  const [year, month] = period.split('-');
  return `${MONTHS[Number(month) - 1]?.short || month}/${year}`;
};

const shareToY = (share) => {
  const capped = Math.max(0, Math.min(100, Number(share) || 0));
  for (let index = 0; index < SHARE_STOPS.length - 1; index += 1) {
    const start = SHARE_STOPS[index];
    const end = SHARE_STOPS[index + 1];
    if (capped <= end) {
      const ratio = end === start ? 0 : (capped - start) / (end - start);
      const yStart = CHART_BOTTOM - (index / (SHARE_STOPS.length - 1)) * CHART_HEIGHT;
      const yEnd = CHART_BOTTOM - ((index + 1) / (SHARE_STOPS.length - 1)) * CHART_HEIGHT;
      return yStart + (yEnd - yStart) * ratio;
    }
  }
  return CHART_TOP;
};

const tickToY = (tick) => shareToY(tick);
const getBrandColor = (empresa) => marcasConfig[empresa]?.cor || '#7d8794';
const formatVolume = (value) => `${Number(value || 0).toLocaleString('pt-BR', { maximumFractionDigits: 1 })} m3`;
const formatPercent = (value) => `${Number(value || 0).toLocaleString('pt-BR', { minimumFractionDigits: 1, maximumFractionDigits: 1 })}%`;
const formatAnnualNumber = (value) => {
  const numeric = Number(value || 0);
  const scaled = numeric >= 1000 ? numeric / 1000 : numeric;
  return scaled.toLocaleString('pt-BR', { maximumFractionDigits: 1 });
};
const formatCompactVolume = (value) => {
  const numeric = Number(value || 0);
  if (numeric >= 1000) {
    return `${(numeric / 1000).toLocaleString('pt-BR', { maximumFractionDigits: 1 })}k m3`;
  }
  return `${numeric.toLocaleString('pt-BR', { maximumFractionDigits: 1 })} m3`;
};

const companyOptions = computed(() => ['TODAS', ...filtrosData.value.empresas]);
const selectedEmpresa = computed(() => (state.empresa === 'TODAS' ? null : state.empresa));
const analysisLabel = computed(() => selectedEmpresa.value || 'Todas as Marcas');
const periodList = computed(() => listPeriods(monthKeyFromDate(state.periodDe), monthKeyFromDate(state.periodAte)));

const ensureVisibleCompanies = () => {
  const valid = new Set(filtrosData.value.empresas);
  const next = visibleEmpresas.value.filter((empresa) => valid.has(empresa));
  visibleEmpresas.value = next.length ? next : [...filtrosData.value.empresas];
};

const fetchFilters = async () => {
  const response = await fetch(`${API_BASE}/filtros`);
  const json = await response.json();
  if (json.status !== 'ok') throw new Error('Nao foi possivel carregar os filtros do IBP.');
  filtrosData.value = {
    ...json.data,
    empresas: (json.data.empresas || []).filter((empresa) => !EXCLUDED_EMPRESAS.includes(empresa)),
  };
  if (selectedEmpresa.value && !filtrosData.value.empresas.includes(selectedEmpresa.value)) {
    state.empresa = 'TODAS';
  }
  ensureVisibleCompanies();
};

const fetchYearSegmentDashboard = async (ano, segmento) => {
  const query = buildQuery({ ano, segmento, mesDe: 'Jan', mesAte: 'Dez' });
  const response = await fetch(`${API_BASE}/dashboard?${query}`);
  const json = await response.json();
  if (json.status !== 'ok') throw new Error(`Falha ao carregar ${segmento} ${ano}.`);
  return json.data;
};

const fetchAllDashboards = async () => {
  loading.value = true;
  fetchError.value = '';
  try {
    await fetchFilters();
    const tasks = [];
    (filtrosData.value.anos || []).forEach((ano) => {
      state.segmentos.forEach((segmento) => {
        tasks.push(fetchYearSegmentDashboard(ano, segmento).then((data) => ({ ano, segmento, data })));
      });
    });

    const results = await Promise.all(tasks);
    const next = {};
    results.forEach(({ ano, segmento, data }) => {
      if (!next[ano]) next[ano] = {};
      next[ano][segmento] = data;
    });
    yearlyDashboards.value = next;
    ensureVisibleCompanies();
  } catch (error) {
    console.error(error);
    fetchError.value = error.message || 'Erro ao carregar o dashboard.';
  } finally {
    loading.value = false;
  }
};

const extractYearMonthSeries = (ano, empresa = null) => {
  const dashboards = yearlyDashboards.value[ano];
  if (!dashboards) return Array.from({ length: 12 }, () => 0);
  const monthly = Array.from({ length: 12 }, () => 0);

  state.segmentos.forEach((segmento) => {
    const dashboard = dashboards[segmento];
    (dashboard?.graficos?.vendasPorMes?.series || []).forEach((serie) => {
      if (empresa && serie.empresa !== empresa) return;
      serie.valores.forEach((valor, index) => {
        monthly[index] += Number(valor || 0);
      });
    });
  });

  return monthly;
};

const allYearCompanyMap = computed(() => {
  const map = new Map();
  (filtrosData.value.anos || []).forEach((ano) => {
    state.segmentos.forEach((segmento) => {
      const dashboard = yearlyDashboards.value[ano]?.[segmento];
      (dashboard?.graficos?.vendasPorMes?.series || []).forEach((serie) => {
        if (!map.has(serie.empresa)) map.set(serie.empresa, new Map());
        serie.valores.forEach((valor, index) => {
          const period = `${ano}-${String(index + 1).padStart(2, '0')}`;
          const current = map.get(serie.empresa).get(period) || 0;
          map.get(serie.empresa).set(period, current + Number(valor || 0));
        });
      });
    });
  });
  return map;
});

const latestAvailablePeriod = computed(() => {
  const anos = [...(filtrosData.value.anos || [])].sort((a, b) => Number(b) - Number(a));
  for (const ano of anos) {
    const totals = extractYearMonthSeries(ano);
    for (let index = totals.length - 1; index >= 0; index -= 1) {
      if (totals[index] > 0) {
        return `${ano}-${String(index + 1).padStart(2, '0')}`;
      }
    }
  }
  return '';
});

const applyDefaultPeriod = () => {
  if (state.periodDirty || !latestAvailablePeriod.value) return;
  state.periodAte = endOfMonth(latestAvailablePeriod.value);
  state.periodDe = startOfMonth(addMonths(latestAvailablePeriod.value, -12));
};

const annualChart = computed(() => {
  const items = (filtrosData.value.anos || []).map((ano) => {
    const totalMercado = extractYearMonthSeries(ano).reduce((sum, value) => sum + value, 0);
    const totalMarca = selectedEmpresa.value
      ? extractYearMonthSeries(ano, selectedEmpresa.value).reduce((sum, value) => sum + value, 0)
      : 0;
    return {
      ano,
      valor: totalMercado,
      valorMarca: totalMarca,
    };
  });

  const startX = 44;
  const gap = items.length > 1 ? 300 / (items.length - 1) : 0;
  const maxValue = Math.max(...items.map((item) => item.valor), 1);
  const linePoints = selectedEmpresa.value
    ? items.map((item, index) => {
      const ratio = item.valorMarca / maxValue;
      return {
        key: `${item.ano}-${selectedEmpresa.value}`,
        x: startX + (index * gap),
        y: 164 - (Math.max(0, ratio) * 132),
        valor: item.valorMarca,
      };
    })
    : [];

  return {
    items,
    max: maxValue,
    linePoints,
  };
});

const currentEndYear = computed(() => (state.periodAte ? state.periodAte.slice(0, 4) : String(filtrosData.value.anos?.slice(-1)[0] || '')));
const currentEndMonth = computed(() => monthLabel(monthKeyFromDate(state.periodAte)));

const segmentChart = computed(() => {
  const items = (filtrosData.value.segmentos || []).map((segmento) => {
    const total = periodList.value.reduce((sum, period) => {
      const [ano] = period.split('-');
      const dashboard = yearlyDashboards.value[ano]?.[segmento.id];
      const index = Number(period.slice(5, 7)) - 1;
      const volume = (dashboard?.graficos?.vendasPorMes?.series || []).reduce(
        (acc, serie) => acc + Number(serie.valores?.[index] || 0),
        0,
      );
      return sum + volume;
    }, 0);
    return {
      ...segmento,
      valor: total,
      active: state.segmentos.includes(segmento.id),
      cor: segmento.id === 'REVENDEDOR' ? '#194f99' : '#5bc0de',
    };
  });
  return {
    items,
    max: Math.max(...items.map((item) => item.valor), 1),
  };
});

const totalForPeriod = (period, empresa = null) => {
  if (empresa) return Number(allYearCompanyMap.value.get(empresa)?.get(period) || 0);
  return Array.from(allYearCompanyMap.value.values()).reduce((sum, periods) => sum + Number(periods.get(period) || 0), 0);
};

const kpis = computed(() => {
  const currentYearMarket = extractYearMonthSeries(currentEndYear.value);
  const previousYearMarket = extractYearMonthSeries(String(Number(currentEndYear.value) - 1));
  const currentYearCompany = extractYearMonthSeries(currentEndYear.value, selectedEmpresa.value);
  const previousYearCompany = extractYearMonthSeries(String(Number(currentEndYear.value) - 1), selectedEmpresa.value);
  const currentMonthIndex = state.periodAte ? Number(state.periodAte.slice(5, 7)) - 1 : -1;
  const previousPeriodTotal = periodList.value.reduce((sum, period) => {
    const previousPeriod = `${String(Number(period.slice(0, 4)) - 1)}-${period.slice(5, 7)}`;
    return sum + totalForPeriod(previousPeriod, selectedEmpresa.value);
  }, 0);
  const marketPreviousPeriod = periodList.value.reduce((sum, period) => {
    const previousPeriod = `${String(Number(period.slice(0, 4)) - 1)}-${period.slice(5, 7)}`;
    return sum + totalForPeriod(previousPeriod);
  }, 0);
  const variation = (current, previous) => (previous > 0 ? ((current - previous) / previous) * 100 : 0);
  const marketPeriod = periodList.value.reduce((sum, period) => sum + totalForPeriod(period), 0);
  const companyPeriod = periodList.value.reduce((sum, period) => sum + totalForPeriod(period, selectedEmpresa.value), 0);

  return [
    {
      id: 'mercado-anual',
      titulo: `Mercado anual ${currentEndYear.value}`,
      valor: currentYearMarket.reduce((sum, value) => sum + value, 0),
      comparativo: previousYearMarket.reduce((sum, value) => sum + value, 0),
      variacaoPercentual: variation(
        currentYearMarket.reduce((sum, value) => sum + value, 0),
        previousYearMarket.reduce((sum, value) => sum + value, 0),
      ),
      destaque: false,
    },
    {
      id: 'mercado-mensal',
      titulo: `Mercado mensal ${currentEndMonth.value}`,
      valor: currentMonthIndex >= 0 ? currentYearMarket[currentMonthIndex] : 0,
      comparativo: currentMonthIndex >= 0 ? previousYearMarket[currentMonthIndex] : 0,
      variacaoPercentual: variation(
        currentMonthIndex >= 0 ? currentYearMarket[currentMonthIndex] : 0,
        currentMonthIndex >= 0 ? previousYearMarket[currentMonthIndex] : 0,
      ),
      destaque: false,
    },
    {
      id: 'mercado-periodo',
      titulo: 'Mercado no periodo',
      valor: marketPeriod,
      comparativo: marketPreviousPeriod,
      variacaoPercentual: variation(marketPeriod, marketPreviousPeriod),
      destaque: false,
    },
    {
      id: 'empresa-anual',
      titulo: `${analysisLabel.value} anual ${currentEndYear.value}`,
      valor: currentYearCompany.reduce((sum, value) => sum + value, 0),
      comparativo: previousYearCompany.reduce((sum, value) => sum + value, 0),
      variacaoPercentual: variation(
        currentYearCompany.reduce((sum, value) => sum + value, 0),
        previousYearCompany.reduce((sum, value) => sum + value, 0),
      ),
      destaque: true,
    },
    {
      id: 'empresa-mensal',
      titulo: `${analysisLabel.value} mensal ${currentEndMonth.value}`,
      valor: currentMonthIndex >= 0 ? currentYearCompany[currentMonthIndex] : 0,
      comparativo: currentMonthIndex >= 0 ? previousYearCompany[currentMonthIndex] : 0,
      variacaoPercentual: variation(
        currentMonthIndex >= 0 ? currentYearCompany[currentMonthIndex] : 0,
        currentMonthIndex >= 0 ? previousYearCompany[currentMonthIndex] : 0,
      ),
      destaque: true,
    },
    {
      id: 'empresa-periodo',
      titulo: `${analysisLabel.value} no periodo`,
      valor: companyPeriod,
      comparativo: previousPeriodTotal,
      variacaoPercentual: variation(companyPeriod, previousPeriodTotal),
      destaque: true,
    },
  ];
});

const chartMetrics = computed(() => {
  if (!periodList.value.length || !allYearCompanyMap.value.size) return null;
  const months = periodList.value.map((period) => {
    const [year, month] = period.split('-');
    const totalMarket = totalForPeriod(period);
    return {
      key: period,
      label: `${MONTHS[Number(month) - 1].short}/${year.slice(2)}`,
      fullLabel: `${MONTHS[Number(month) - 1].short}/${year}`,
      totalMarket,
    };
  });

  const startX = 80;
  const gap = months.length > 1 ? 840 / (months.length - 1) : 0;
  const maxMarketVolume = Math.max(...months.map((month) => month.totalMarket), 1);

  const bars = months.map((month, index) => {
    const height = (month.totalMarket / maxMarketVolume) * 210;
    return {
      key: month.key,
      x: startX + index * gap - 18,
      y: CHART_BOTTOM - height,
      h: height,
      tooltip: `${month.fullLabel}: ${formatVolume(month.totalMarket)}`,
      label: formatCompactVolume(month.totalMarket),
    };
  });

  const lines = filtrosData.value.empresas.map((empresa) => {
    const visible = visibleEmpresas.value.includes(empresa);
    const highlighted = selectedEmpresa.value === empresa;
    const points = months.map((month, index) => {
      const volume = totalForPeriod(month.key, empresa);
      const share = month.totalMarket > 0 ? (volume / month.totalMarket) * 100 : 0;
      return {
        key: `${empresa}-${month.key}`,
        x: startX + index * gap,
        y: shareToY(share),
        volume,
        share,
        label: `${empresa} - ${month.fullLabel}`,
      };
    });

    return {
      empresa,
      cor: getBrandColor(empresa),
      visible,
      strokeWidth: selectedEmpresa.value ? (highlighted ? 4 : 2) : 2,
      points,
    };
  });

  return { months, bars, lines, startX, gap };
});

const sideShares = computed(() => {
  if (!state.periodAte) return [];
  const currentPeriod = monthKeyFromDate(state.periodAte);
  const monthTotal = totalForPeriod(currentPeriod);
  return filtrosData.value.empresas
    .map((empresa) => {
      const volume = totalForPeriod(currentPeriod, empresa);
      return {
        nome: empresa,
        volume,
        share: monthTotal > 0 ? (volume / monthTotal) * 100 : 0,
        visible: visibleEmpresas.value.includes(empresa),
        selected: selectedEmpresa.value === empresa,
      };
    })
    .sort((a, b) => b.share - a.share);
});

const selectEmpresa = (empresa) => {
  state.empresa = empresa;
  if (empresa !== 'TODAS' && !visibleEmpresas.value.includes(empresa)) {
    visibleEmpresas.value = [...visibleEmpresas.value, empresa];
  }
};

const toggleVisibleEmpresa = (empresa) => {
  if (visibleEmpresas.value.includes(empresa)) {
    if (visibleEmpresas.value.length === 1) return;
    visibleEmpresas.value = visibleEmpresas.value.filter((item) => item !== empresa);
    return;
  }
  visibleEmpresas.value = [...visibleEmpresas.value, empresa];
};

const toggleSegmento = (segmento) => {
  if (state.segmentos.includes(segmento)) {
    if (state.segmentos.length === 1) return;
    state.segmentos.splice(state.segmentos.indexOf(segmento), 1);
  } else {
    state.segmentos.push(segmento);
  }
};

const onPeriodChange = () => {
  if (state.periodDe && state.periodAte && comparePeriods(state.periodDe, state.periodAte) > 0) {
    state.periodAte = state.periodDe;
  }
  state.periodDirty = true;
};

const clearFilters = () => {
  state.empresa = 'TODAS';
  state.segmentos.splice(0, state.segmentos.length, 'REVENDEDOR', 'CONSUMIDOR');
  state.periodDirty = false;
  visibleEmpresas.value = [...filtrosData.value.empresas];
  applyDefaultPeriod();
};

const showTooltip = (event, title, value) => {
  const bounds = event.currentTarget?.ownerSVGElement?.getBoundingClientRect?.()
    || event.currentTarget?.getBoundingClientRect?.();
  if (!bounds) return;
  hover.value = {
    visible: true,
    x: event.clientX - bounds.left + 12,
    y: event.clientY - bounds.top - 12,
    title,
    value,
  };
};

const hideTooltip = () => {
  hover.value.visible = false;
};

watch(
  () => state.segmentos.join('|'),
  async () => {
    await fetchAllDashboards();
    applyDefaultPeriod();
  },
);

watch(latestAvailablePeriod, applyDefaultPeriod);

onMounted(async () => {
  await fetchAllDashboards();
  applyDefaultPeriod();
});
</script>

<template>
  <div class="pbi-layout">
    <header class="pbi-filters">
      <div class="filter-left">
        <div class="filter-box">
          <div class="filter-label">Selecione sua marca</div>
          <div class="filter-dropdown">
            <button class="filter-trigger">{{ analysisLabel }}</button>
            <div class="filter-menu">
              <button
                v-for="empresa in companyOptions"
                :key="empresa"
                class="menu-item"
                :class="{ active: state.empresa === empresa }"
                @click="selectEmpresa(empresa)"
              >
                {{ empresa === 'TODAS' ? 'Todas' : empresa }}
              </button>
            </div>
          </div>
        </div>

        <div class="filter-box">
          <div class="filter-label">Consumidor / Revendedor</div>
          <div class="segment-switches">
            <button
              v-for="segmento in filtrosData.segmentos"
              :key="segmento.id"
              class="segment-btn"
              :class="{ active: state.segmentos.includes(segmento.id) }"
              @click="toggleSegmento(segmento.id)"
            >
              {{ segmento.label }}
            </button>
          </div>
        </div>

        <div class="filter-box">
          <div class="filter-label">Periodo do BI</div>
          <div class="period-picker">
            <div class="date-field">
              <label>De</label>
              <input type="date" v-model="state.periodDe" :max="state.periodAte || undefined" @change="onPeriodChange">
            </div>
            <div class="date-field">
              <label>Ate</label>
              <input type="date" v-model="state.periodAte" :min="state.periodDe || undefined" @change="onPeriodChange">
            </div>
          </div>
        </div>
      </div>

      <button class="btn-clear" @click="clearFilters">Limpar filtros</button>
    </header>

    <div v-if="fetchError" class="error-box">{{ fetchError }}</div>

    <div v-if="!fetchError" class="pbi-content" :class="{ 'loading-opacity': loading }">
      <section class="top-section">
        <div class="kpi-grid">
          <div
            v-for="card in kpis"
            :key="card.id"
            class="card kpi-card"
            :class="{ 'card-highlight': card.destaque }"
            :title="`${card.titulo}: ${formatVolume(card.valor)}`"
          >
            <div class="kpi-label">{{ card.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(card.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ formatVolume(card.comparativo) }}</span>
              <span class="pill" :class="{ neg: card.variacaoPercentual < 0 }">
                {{ formatPercent(card.variacaoPercentual) }} YoY
              </span>
            </div>
          </div>
        </div>

        <div class="card chart-annual">
          <div class="card-title">Vendas Anual (Mil por Metros Cubicos)</div>
          <div class="card-subtitle">{{ analysisLabel }}</div>
          <div class="annual-chart-wrap">
            <svg
              v-if="annualChart.linePoints.length"
              class="annual-line-overlay"
              viewBox="0 0 360 180"
              preserveAspectRatio="none"
            >
              <polyline
                :points="annualChart.linePoints.map((point) => `${point.x},${point.y}`).join(' ')"
                :stroke="getBrandColor(selectedEmpresa)"
                stroke-width="3"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              />
              <g v-for="point in annualChart.linePoints" :key="point.key">
                <circle :cx="point.x" :cy="point.y" r="4" :fill="getBrandColor(selectedEmpresa)" />
              </g>
            </svg>

            <div class="bars-container">
              <div
                v-for="item in annualChart.items"
                :key="item.ano"
                class="bar-col"
                :title="`${item.ano}: Mercado ${formatVolume(item.valor)}${selectedEmpresa ? ` | ${selectedEmpresa}: ${formatVolume(item.valorMarca)}` : ''}`"
              >
                <span class="bar-txt">{{ formatAnnualNumber(item.valor) }}</span>
                <div
                  class="bar"
                  :class="{ orange: item.ano === currentEndYear }"
                  :style="{ height: `${(item.valor / annualChart.max) * 100}%` }"
                />
                <span class="bar-label">{{ item.ano }}</span>
              </div>
            </div>
          </div>
        </div>

        <div class="card chart-segment">
          <div class="card-title">Vendas por segmento</div>
          <div class="card-subtitle">{{ monthLabel(monthKeyFromDate(state.periodDe)) }} a {{ monthLabel(monthKeyFromDate(state.periodAte)) }}</div>
          <div class="segment-container">
            <div
              v-for="segmento in segmentChart.items"
              :key="segmento.id"
              class="seg-col"
              :title="`${segmento.label}: ${formatVolume(segmento.valor)}`"
            >
              <span class="bar-txt">{{ formatCompactVolume(segmento.valor) }}</span>
              <div
                class="seg-bar"
                :class="{ faded: !segmento.active }"
                :style="{ background: segmento.cor, height: `${(segmento.valor / segmentChart.max) * 100}%` }"
              />
              <span class="bar-label bold">{{ segmento.label }}</span>
            </div>
          </div>
        </div>
      </section>

      <section class="card main-viz">
        <div class="viz-header">
          <div class="viz-info">
            <h3>Vendas de Lubrificantes por mes</h3>
            <p>{{ monthLabel(monthKeyFromDate(state.periodDe)) }} a {{ monthLabel(monthKeyFromDate(state.periodAte)) }} | share em {{ monthLabel(monthKeyFromDate(state.periodAte)) }}</p>
          </div>
        </div>

        <div class="viz-legend">
          <button
            v-for="empresa in filtrosData.empresas"
            :key="empresa"
            class="leg-item"
            :class="{ active: visibleEmpresas.includes(empresa), highlighted: state.empresa === empresa }"
            @click="toggleVisibleEmpresa(empresa)"
            :title="visibleEmpresas.includes(empresa) ? `Ocultar ${empresa}` : `Mostrar ${empresa}`"
          >
            <span class="dot" :style="{ background: getBrandColor(empresa) }"></span>
            <span>{{ empresa }}</span>
          </button>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg v-if="chartMetrics" viewBox="0 0 1000 320" preserveAspectRatio="none">
              <g class="axis-y">
                <line x1="55" :y1="CHART_BOTTOM" x2="950" :y2="CHART_BOTTOM" stroke="#dfe3e8" />
                <g v-for="tick in SHARE_TICKS" :key="`tick-${tick}`">
                  <text x="48" :y="tickToY(tick) + 4" text-anchor="end" class="svg-txt-axis">{{ tick }}%</text>
                  <line x1="55" :y1="tickToY(tick)" x2="950" :y2="tickToY(tick)" stroke="#eef1f4" />
                </g>
              </g>

              <g v-for="bar in chartMetrics.bars" :key="bar.key">
                <rect
                  :x="bar.x"
                  :y="bar.y"
                  width="36"
                  :height="bar.h"
                  fill="#e3e7eb"
                  rx="3"
                  @mousemove="showTooltip($event, bar.tooltip, bar.label)"
                  @mouseleave="hideTooltip"
                />
              </g>

              <g v-for="line in chartMetrics.lines" :key="line.empresa" v-show="line.visible">
                <polyline
                  :points="line.points.map((point) => `${point.x},${point.y}`).join(' ')"
                  fill="none"
                  :stroke="line.cor"
                  :stroke-width="line.strokeWidth"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
                <g v-for="point in line.points" :key="point.key">
                  <circle
                    :cx="point.x"
                    :cy="point.y"
                    r="4.5"
                    :fill="line.cor"
                    stroke="#ffffff"
                    stroke-width="1.5"
                    @mousemove="showTooltip($event, point.label, `${formatPercent(point.share)} | ${formatVolume(point.volume)}`)"
                    @mouseleave="hideTooltip"
                  />
                </g>
              </g>

              <text
                v-for="(month, index) in chartMetrics.months"
                :key="month.key"
                :x="chartMetrics.startX + index * chartMetrics.gap"
                y="308"
                text-anchor="middle"
                class="svg-txt"
              >
                {{ month.label }}
              </text>
            </svg>

            <div v-if="hover.visible" class="chart-tooltip" :style="{ left: `${hover.x}px`, top: `${hover.y}px` }">
              <strong>{{ hover.title }}</strong>
              <span>{{ hover.value }}</span>
            </div>
          </div>

          <aside class="sidebar">
            <div class="side-title">Share mercado</div>
            <div class="side-subtitle">{{ monthLabel(monthKeyFromDate(state.periodAte)) }}</div>
            <button
              v-for="marca in sideShares"
              :key="marca.nome"
              class="side-item"
              :class="{ active: marca.visible, selected: marca.selected }"
              @click="toggleVisibleEmpresa(marca.nome)"
              :title="`${marca.nome}: ${formatPercent(marca.share)} | ${formatVolume(marca.volume)}`"
            >
              <img :src="`/logos/lubrificantes/${marcasConfig[marca.nome]?.logo || 'generic'}.png`" class="brand-logo-img" />
              <span class="name">{{ marca.nome }}</span>
              <span class="badge">{{ formatPercent(marca.share) }}</span>
            </button>
          </aside>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
.pbi-layout {
  min-height: 100vh;
  padding: 16px;
  background: linear-gradient(180deg, #f4f5f7 0%, #eceff2 100%);
  font-family: 'Segoe UI', sans-serif;
}

.pbi-filters {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  padding: 16px 18px;
  border: 1px solid #dde3ea;
  border-radius: 16px;
  background: #ffffff;
  box-shadow: 0 10px 24px rgba(17, 24, 39, 0.06);
}

.filter-left {
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
  align-items: flex-start;
}

.filter-box {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.filter-label {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  color: #6b7280;
  letter-spacing: 0.04em;
}

.filter-dropdown {
  position: relative;
  width: 220px;
}

.filter-trigger {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #d6dce3;
  border-radius: 10px;
  background: #f8fafc;
  font-weight: 700;
  text-align: left;
  cursor: pointer;
}

.filter-menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  z-index: 20;
  display: none;
  width: 100%;
  max-height: 260px;
  overflow: auto;
  border: 1px solid #d6dce3;
  border-radius: 12px;
  background: #ffffff;
  box-shadow: 0 12px 24px rgba(15, 23, 42, 0.12);
}

.filter-dropdown:hover .filter-menu {
  display: block;
}

.menu-item {
  width: 100%;
  padding: 10px 12px;
  border: 0;
  background: transparent;
  text-align: left;
  cursor: pointer;
  font-size: 12px;
}

.menu-item.active,
.menu-item:hover {
  background: #fff3e7;
  color: #c45d10;
}

.segment-switches {
  display: flex;
  gap: 8px;
}

.segment-btn {
  padding: 10px 14px;
  border: 1px solid #d6dce3;
  border-radius: 10px;
  background: #f8fafc;
  color: #4b5563;
  font-weight: 700;
  cursor: pointer;
}

.segment-btn.active {
  background: #f58220;
  border-color: #f58220;
  color: #ffffff;
}

.period-picker {
  display: flex;
  gap: 8px;
  padding: 8px;
  border: 1px solid #d6dce3;
  border-radius: 12px;
  background: #f8fafc;
}

.date-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.date-field label {
  font-size: 10px;
  font-weight: 700;
  color: #6b7280;
}

.date-field input {
  min-width: 138px;
  padding: 8px 10px;
  border: 1px solid #d6dce3;
  border-radius: 8px;
  background: #ffffff;
}

.btn-clear {
  align-self: center;
  padding: 10px 16px;
  border: 1px solid #111827;
  border-radius: 10px;
  background: #ffffff;
  font-weight: 700;
  cursor: pointer;
}

.error-box {
  margin-top: 14px;
  padding: 14px 16px;
  border: 1px solid #fecaca;
  border-radius: 12px;
  background: #fff1f2;
  color: #b91c1c;
  font-weight: 600;
}

.pbi-content {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-width: 1200px;
  margin-top: 16px;
  transition: opacity 0.25s ease;
}

.loading-opacity {
  opacity: 0.55;
  pointer-events: none;
}

.top-section {
  display: grid;
  grid-template-columns: 3fr 1.45fr 1fr;
  gap: 16px;
}

.kpi-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.card {
  display: flex;
  flex-direction: column;
  padding: 14px;
  border: 1px solid #dde3ea;
  border-radius: 16px;
  background: #ffffff;
  box-shadow: 0 12px 24px rgba(17, 24, 39, 0.04);
}

.card-highlight {
  background: linear-gradient(180deg, #fffaf4 0%, #fff4e6 100%);
}

.kpi-label {
  font-size: 12px;
  color: #6b7280;
}

.kpi-val {
  margin: 8px 0;
  font-size: 21px;
  font-weight: 800;
  color: #111827;
}

.kpi-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  margin-top: auto;
}

.sub {
  font-size: 11px;
  color: #6b7280;
}

.pill {
  padding: 4px 8px;
  border-radius: 999px;
  background: #e7f6ec;
  color: #0f8b47;
  font-size: 10px;
  font-weight: 700;
}

.pill.neg {
  background: #fee2e2;
  color: #c2410c;
}

.card-title {
  font-size: 15px;
  font-weight: 800;
  color: #111827;
}

.card-subtitle {
  margin-top: 4px;
  font-size: 11px;
  color: #6b7280;
}

.annual-chart-wrap {
  position: relative;
  flex: 1;
}

.annual-line-overlay {
  position: absolute;
  inset: 6px 0 0;
  width: 100%;
  height: calc(100% - 10px);
  pointer-events: none;
  z-index: 2;
}

.bars-container,
.segment-container {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 10px;
  flex: 1;
  margin-top: 16px;
}

.bars-container {
  position: relative;
  z-index: 1;
}

.bar-col,
.seg-col {
  display: flex;
  flex: 1;
  flex-direction: column;
  align-items: center;
  justify-content: flex-end;
  min-height: 185px;
}

.bar-txt {
  margin-bottom: 6px;
  font-size: 10px;
  font-weight: 700;
  color: #374151;
}

.bar {
  width: 54%;
  min-height: 4px;
  border-radius: 6px 6px 0 0;
  background: #d5dbe1;
}

.bar.orange {
  background: #f58220;
}

.seg-bar {
  width: 42px;
  min-height: 4px;
  border-radius: 8px 8px 0 0;
}

.seg-bar.faded {
  opacity: 0.25;
}

.bar-label {
  margin-top: 8px;
  font-size: 11px;
  color: #6b7280;
}

.bar-label.bold {
  font-weight: 700;
}

.main-viz {
  min-height: 520px;
}

.viz-info h3 {
  margin: 0;
  font-size: 19px;
  color: #111827;
}

.viz-info p {
  margin: 6px 0 0;
  color: #6b7280;
  font-size: 12px;
}

.viz-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin: 16px 0 14px;
  padding-bottom: 12px;
  border-bottom: 1px solid #edf1f5;
}

.leg-item {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 7px 10px;
  border: 1px solid #dde3ea;
  border-radius: 999px;
  background: #ffffff;
  color: #6b7280;
  font-size: 11px;
  font-weight: 700;
  cursor: pointer;
}

.leg-item.active {
  color: #111827;
}

.leg-item.highlighted {
  border-color: #f58220;
  background: #fff7ef;
}

.dot {
  width: 18px;
  height: 4px;
  border-radius: 999px;
}

.viz-body {
  display: flex;
  gap: 18px;
  flex: 1;
}

.svg-area {
  position: relative;
  flex: 1;
  min-height: 360px;
}

.svg-txt-axis {
  font-size: 11px;
  fill: #8a94a3;
  font-weight: 700;
}

.svg-txt {
  font-size: 11px;
  fill: #8a94a3;
  font-weight: 600;
}

.chart-tooltip {
  position: absolute;
  z-index: 5;
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 10px 12px;
  border-radius: 10px;
  background: rgba(17, 24, 39, 0.92);
  color: #ffffff;
  font-size: 11px;
  pointer-events: none;
  box-shadow: 0 10px 24px rgba(17, 24, 39, 0.22);
}

.sidebar {
  width: 240px;
  padding-left: 18px;
  border-left: 1px solid #edf1f5;
}

.side-title {
  font-size: 15px;
  font-weight: 800;
  color: #111827;
}

.side-subtitle {
  margin-top: 4px;
  margin-bottom: 14px;
  font-size: 11px;
  color: #6b7280;
}

.side-item {
  display: grid;
  grid-template-columns: 24px 1fr auto;
  align-items: center;
  gap: 10px;
  width: 100%;
  margin-bottom: 8px;
  padding: 8px 10px;
  border: 1px solid transparent;
  border-radius: 12px;
  background: #ffffff;
  cursor: pointer;
  text-align: left;
}

.side-item.active {
  background: #f8fafc;
  border-color: #dde3ea;
}

.side-item.selected {
  border-color: #f58220;
  background: #fff7ef;
}

.brand-logo-img {
  width: 24px;
  height: 24px;
  object-fit: contain;
}

.name {
  font-size: 12px;
  font-weight: 700;
  color: #111827;
}

.badge {
  padding: 4px 8px;
  border-radius: 999px;
  background: #eef2f7;
  color: #374151;
  font-size: 10px;
  font-weight: 800;
}
</style>
