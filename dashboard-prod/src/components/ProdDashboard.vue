<template>
    <div class="dashboard">
        <header class="dashboard-header">
            <h1 class="dashboard-title">Dashboard de Produção Amway/Nutrilite</h1>
            <p class="dashboard-description">
                Acompanhe o desempenho da produção com dados atualizados sobre matéria-prima, concentrado e rendimento.
                Selecione as ordens de produção para analisar e comparar os resultados.
            </p>
        </header>

        <div class="content">
            <!-- Barra de Navegação Lateral -->
            <div class="sidebar">
                <h3>Ordens de Produção</h3>
                <!-- Filtro de Data -->
                <div class="data-filters">
                        <label for="year">Ano</label>
                        <select v-model="selectedYear" @change="applyDateFilter">
                            <option v-for="year in availableYears" :key="year" :value="year">{{ year }}</option>
                        </select>

                        <label for="month">Mês</label>
                        <select v-model="selectedMonth" @change="applyDateFilter">
                            <option v-for="(month, index) in months" :key="index" :value="index + 1">
                                {{ month }}
                            </option>
                        </select>
                    </div>
                    <button @click="clearFilters" class="sidebar-button">
                        Limpar Filtros
                    </button>
                <details>
                    <summary>Mostrar Ordens de Produção</summary>
                    <p>Selecione para comparar</p>
                    <div class="order-list">
                        <label v-for="indicador in filteredOrders" :key="indicador.ordem" class="order-item">
                            <input type="checkbox" :value="indicador.ordem" v-model="filtroOrdemSelecionadas" />
                            Ordem {{ indicador.ordem }}
                        </label>
                    </div>
                </details>
            </div>

            <!-- Conteúdo principal -->
            <div class="main-content">
                <!-- KPIs -->
                <div class="kpi-section">
                    <div class="kpi-card">
                        <h3>Ordens de Produção Selecionadas</h3>
                        <p>{{ filtroOrdemSelecionadas.length }} de {{ filteredOrders.length }}</p>
                    </div>
                    <div class="kpi-card">
                        <h3>Rendimento Médio (%)</h3>
                        <p>{{ averageRendimento.toFixed(2) }}%</p>
                    </div>
                    <div class="kpi-card">
                        <h3>Total de Matéria-prima (Kg)</h3>
                        <p>{{ totalMateriaPrima.toFixed(2) }} Kg</p>
                    </div>
                    <div class="kpi-card">
                        <h3>Total de Concentrado (Kg)</h3>
                        <p>{{ totalConcentrado.toFixed(2) }} Kg</p>
                    </div>
                </div>

                <!-- Navegação entre abas -->
                <div class="tabs">
                    <button :class="{ active: activeTab === 'rendimento' }" @click="activeTab = 'rendimento'"
                        class="tab-button">
                        Comparação de Rendimento (%)
                    </button>
                    <button :class="{ active: activeTab === 'relation' }" @click="activeTab = 'relation'"
                        class="tab-button">
                        Matéria-prima vs Concentrado (Kg)
                    </button>
                    <button :class="{ active: activeTab === 'rawData' }" @click="activeTab = 'rawData'"
                        class="tab-button">
                        Dados Brutos
                    </button>
                </div>

                <!-- Conteúdo das abas -->
                <div v-if="activeTab === 'rendimento'" class="chart-container">
                    <div class="chart-toggle">
                        <button @click="chartType = 'line'" :class="{ active: chartType === 'line' }">Gráfico de
                            Linha</button>
                        <button @click="chartType = 'bar'" :class="{ active: chartType === 'bar' }">Gráfico de
                            Barras</button>
                    </div>
                    <div v-if="chartType === 'line'">
                        <line-chart :data="chartData" />
                    </div>
                    <div v-else>
                        <bar-chart :data="barChartData" />
                    </div>
                </div>

                <div v-if="activeTab === 'relation'" class="chart-container">
                    <div class="chart-toggle">
                        <button @click="relationChartType = 'line'"
                            :class="{ active: relationChartType === 'line' }">Gráfico de Linha</button>
                        <button @click="relationChartType = 'bar'"
                            :class="{ active: relationChartType === 'bar' }">Gráfico de Barras</button>
                    </div>
                    <div v-if="relationChartType === 'line'">
                        <line-chart :data="relationChartData" />
                    </div>
                    <div v-else>
                        <bar-chart :data="relationBarChartData" />
                    </div>
                </div>

                <div v-if="activeTab === 'rawData'" class="raw-data-container">
                    <table class="data-table">
                        <thead>
                            <tr>
                                <th>Ordem de Produção</th>
                                <th>Data</th>
                                <th>Matéria-prima (Kg)</th>
                                <th>Concentrado (Kg)</th>
                                <th>Rendimento (%)</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="indicador in indicadoresFiltrados" :key="indicador.ordem">
                                <td>{{ indicador.ordem }}</td>
                                <td>{{ formatDate(indicador.data) }}</td>
                                <td>{{ indicador.materiaPrima }}</td>
                                <td>{{ indicador.concentrado }}</td>
                                <td>{{ indicador.rendimento.toFixed(2) }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import axios from "axios";
import LineChart from "./LineChart.vue";
import BarChart from "./BarChart.vue"; // Componente para gráfico de barras

export default {
    components: { LineChart, BarChart },
    data() {
        return {
            indicadores: [], // Lista de dados combinados
            monthlyData: [],
            chartData: null, // Dados para o gráfico de linha (rendimento)
            barChartData: null, // Dados para o gráfico de barras (rendimento)
            relationChartData: null, // Dados para o gráfico de linha de relação (matéria-prima vs concentrado)
            relationBarChartData: null, // Dados para o gráfico de barras de relação (matéria-prima vs concentrado)
            filtroOrdemSelecionadas: [], // Filtro de ordens selecionadas
            loading: false, // Indicador de carregamento
            error: null, // Mensagem de erro
            activeTab: "rendimento", // Aba ativa ("rendimento" ou "relation")
            chartType: "line", // Tipo de gráfico de rendimento (linha ou barra)
            relationChartType: "line", // Tipo de gráfico de relação (linha ou barra)
            selectedMonth: null,
            selectedYear: null,
            availableYears: [], // Anos disponíveis para o seletor
            availableMonths: [], // Meses disponíveis para o seletor
            months: [
                "Janeiro", "Fevereiro", "Março", "Abril", "Maio", "Junho",
                "Julho", "Agosto", "Setembro", "Outubro", "Novembro", "Dezembro"
            ],
        };
    },
    computed: {
        filteredOrders() {
            let filtered = this.indicadores;

            // Filtro por ano
            if (this.selectedYear) {
                filtered = filtered.filter(indicador =>
                    new Date(indicador.data).getFullYear() === this.selectedYear
                );
            }

            // Filtro por mês
            if (this.selectedMonth) {
                filtered = filtered.filter(indicador =>
                    new Date(indicador.data).getMonth() + 1 === this.selectedMonth
                );
            }

            return filtered;
        },
        indicadoresFiltrados() {
            let indicadoresFiltrados = this.indicadores;

            // Filtro de ordens selecionadas
            if (this.filtroOrdemSelecionadas.length > 0) {
                indicadoresFiltrados = indicadoresFiltrados.filter(indicador =>
                    this.filtroOrdemSelecionadas.includes(indicador.ordem)
                );
            }

            // Filtro de ano e mês
            if (this.selectedYear) {
                indicadoresFiltrados = indicadoresFiltrados.filter(indicador =>
                    new Date(indicador.data).getFullYear() === this.selectedYear
                );
            }

            if (this.selectedMonth) {
                indicadoresFiltrados = indicadoresFiltrados.filter(indicador =>
                    new Date(indicador.data).getMonth() + 1 === this.selectedMonth
                );
            }

            return indicadoresFiltrados;
        },
        // KPI - Rendimento Médio
        averageRendimento() {
            const totalRendimento = this.indicadoresFiltrados.reduce((acc, indicador) => acc + indicador.rendimento, 0);
            return this.indicadoresFiltrados.length ? totalRendimento / this.indicadoresFiltrados.length : 0;
        },
        // KPI - Total de Matéria-prima
        totalMateriaPrima() {
            return this.indicadoresFiltrados.reduce((acc, indicador) => acc + indicador.materiaPrima, 0);
        },
        // KPI - Total de Concentrado
        totalConcentrado() {
            return this.indicadoresFiltrados.reduce((acc, indicador) => acc + indicador.concentrado, 0);
        },
    },
    methods: {
        formatDate(date) {
            const d = new Date(date);
            const day = String(d.getDate()).padStart(2, "0");
            const month = String(d.getMonth() + 1).padStart(2, "0");
            const year = d.getFullYear();
            return `${day}/${month}/${year}`;
        },
        clearFilters() {
            this.filtroOrdemSelecionadas = [];
            this.selectedYear = null;
            this.selectedMonth = null;
        },
        async fetchData() {
            this.loading = true;
            this.error = null;

            try {
                const [materiaPrimaResponse, concentradoResponse] = await Promise.all([
                    axios.get("http://localhost:3000/api/get_soma_materia_prima"),
                    axios.get("http://localhost:3000/api/get_soma_concentrado"),
                ]);

                const materiaPrima = materiaPrimaResponse.data.map(item => this.normalizeKeys(item));
                const concentrado = concentradoResponse.data.map(item => this.normalizeKeys(item));

                const ordemMap = {};

                materiaPrima.forEach(item => {
                    ordemMap[item["Ordem de Produção"]] = {
                        ordem: item["Ordem de Produção"],
                        data: `${item["Ano"]}-${item["Mês"]}-${item["Dia"]}`,
                        ano: item["Ano"],
                        mes: item["Mês"],
                        materiaPrima: item["Soma de Matéria-prima (Kg)"] || 0,
                        concentrado: 0,
                    };
                });

                concentrado.forEach(item => {
                    if (ordemMap[item["Ordem de Produção"]]) {
                        ordemMap[item["Ordem de Produção"]].concentrado =
                            item["Soma de Concentrado (Kg)"] || 0;
                    }
                });

                const indicadores = Object.values(ordemMap).map(indicador => ({
                    ...indicador,
                    rendimento:
                        (indicador.concentrado / indicador.materiaPrima) * 100 || 0,
                }));

                this.indicadores = indicadores;

                // Definir os anos e meses disponíveis para o filtro
                this.availableYears = [...new Set(indicadores.map(indicador => indicador.ano))].sort();

                // Atualiza os meses disponíveis com base nos dados filtrados por ano
                this.updateAvailableMonths();

                this.updateCharts();
            } catch (error) {
                this.error = "Erro ao carregar os dados, tente novamente mais tarde.";
                console.error(error);
            } finally {
                this.loading = false;
            }
        },
        updateAvailableMonths() {
            const monthsInYear = new Set();

            // Filtra meses disponíveis no ano selecionado
            this.indicadores.forEach(indicador => {
                if (this.selectedYear === null || indicador.ano === this.selectedYear) {
                    monthsInYear.add(indicador.mes);
                }
            });

            // Atualiza a lista de meses disponíveis com base nos dados
            this.availableMonths = [...monthsInYear].sort((a, b) => a - b); // Ordena numericamente
        },
        normalizeKeys(obj) {
            return Object.keys(obj).reduce((acc, key) => {
                const cleanKey = key.trim().replace(/^\uFEFF/, "");
                acc[cleanKey] = obj[key];
                return acc;
            }, {});
        },
        updateCharts() {
            const indicadores = this.indicadoresFiltrados;

            // Dados para o gráfico de linha (rendimento)
            this.chartData = {
                labels: indicadores.map(i => i.ordem),
                datasets: [
                    {
                        label: "Rendimento (%)",
                        data: indicadores.map(i => i.rendimento),
                        borderColor: "rgba(75, 192, 192, 1)",
                        backgroundColor: "rgba(75, 192, 192, 0.2)",
                        borderWidth: 3,
                        pointBackgroundColor: "rgba(75, 192, 192, 1)",
                        pointRadius: 5,
                        fill: true,
                        tension: 0.4,
                    },
                ],
            };

            // Dados para o gráfico de barras (rendimento)
            this.barChartData = {
                labels: indicadores.map(i => i.ordem),
                datasets: [
                    {
                        label: "Rendimento (%)",
                        data: indicadores.map(i => i.rendimento),
                        backgroundColor: "rgba(75, 192, 192, 0.7)",
                        borderColor: "rgba(75, 192, 192, 1)",
                        borderWidth: 2,
                        hoverBackgroundColor: "rgba(75, 192, 192, 1)",
                        hoverBorderColor: "rgba(75, 192, 192, 1)",
                        borderRadius: 5,
                        barThickness: 50,
                    },
                ],
            };

            // Dados para o gráfico de linha de Matéria-prima vs Concentrado
            this.relationChartData = {
                labels: indicadores.map(i => i.ordem),
                datasets: [
                    {
                        label: "Matéria-prima (Kg)",
                        data: indicadores.map(i => i.materiaPrima),
                        borderColor: "rgba(255, 99, 132, 1)",
                        borderWidth: 3,
                        pointRadius: 5,
                        fill: false,
                        tension: 0.4,
                    },
                    {
                        label: "Concentrado (Kg)",
                        data: indicadores.map(i => i.concentrado),
                        borderColor: "rgba(54, 162, 235, 1)",
                        borderWidth: 3,
                        pointRadius: 5,
                        fill: false,
                        tension: 0.4,
                    },
                ],
            };

            // Dados para o gráfico de barras de Matéria-prima vs Concentrado
            this.relationBarChartData = {
                labels: indicadores.map(i => i.ordem),
                datasets: [
                    {
                        label: "Matéria-prima (Kg)",
                        data: indicadores.map(i => i.materiaPrima),
                        backgroundColor: "rgba(255, 99, 132, 0.7)",
                        borderColor: "rgba(255, 99, 132, 1)",
                        borderWidth: 2,
                        hoverBackgroundColor: "rgba(255, 99, 132, 1)",
                        hoverBorderColor: "rgba(255, 99, 132, 1)",
                        borderRadius: 5,
                        barThickness: 30,
                    },
                    {
                        label: "Concentrado (Kg)",
                        data: indicadores.map(i => i.concentrado),
                        backgroundColor: "rgba(54, 162, 235, 0.7)",
                        borderColor: "rgba(54, 162, 235, 1)",
                        borderWidth: 2,
                        hoverBackgroundColor: "rgba(54, 162, 235, 1)",
                        hoverBorderColor: "rgba(54, 162, 235, 1)",
                        borderRadius: 5,
                        barThickness: 30,
                    },
                ],
            };
        },
        changeChartType(type) {
            this.chartType = type;
            this.updateCharts();
        },
        changeRelationChartType(type) {
            this.relationChartType = type;
            this.updateCharts();
        },
    },
    watch: {
        indicadoresFiltrados: "updateCharts", // Atualiza gráficos quando os indicadores filtrados mudam
        selectedYear: "updateAvailableMonths", // Atualiza meses disponíveis ao mudar o ano
    },
    mounted() {
        this.fetchData();
    },
};
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap');

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: 'Poppins', sans-serif;
}

.dashboard-header {
    background: linear-gradient(135deg, #4db8ff, #2f4e7d);
    border-radius: 10px;
    color: white;
    padding: 40px 20px;
    text-align: center;
}

.dashboard-title {
    font-size: 3rem;
    font-weight: 600;
}

.dashboard-description {
    font-size: 1.2rem;
    color: #dfe7f0;
}

/* Barra lateral */
.sidebar {
    width: 250px;
    float: left;
    margin-top: 10px;
    background-color: #0056b3;
    border-radius: 10px;
    color: white;
}

h3 {
    font-size: 1.2rem;
    margin: 10px;
}

.data-filters {
    display: flex;
    flex-direction: column;
    align-items: center;
    border: none;
    border-radius: 5px;
    margin: 5px;
    font-size: 1rem;
}

.data-filters select {
    border: 1px solid #333;
    border-radius: 5px;
    font-size: 1rem;
}

.order-list {
    max-height: 300px;
    overflow-y: auto;
}

.sidebar-button {
    margin: 5px;
    padding: 5px;
    border-radius: 5px;
    border: none;
    cursor: pointer;
    background-color: #ddd;
}

.sidebar-button:hover {
    color: white;
    background-color: #003e7e;
}

.order-item {
    display: block;
    margin: 8px 0;
    font-size: 1rem;
}

/* Detalhes - para esconder/mostrar a lista */
details {
    margin-bottom: 15px;
    border-radius: 8px;
    padding: 10px;
}

summary {
    font-size: 1.1rem;
    font-weight: 600;
    cursor: pointer;
    color: white;
    padding: 5px;
    border-radius: 5px;
}

summary:hover {
    background-color: #003e7e;
}

/* Conteúdo principal */
.main-content {
    margin-left: 280px;
    padding: 10px;
}

.chart-container {
    margin-bottom: 30px;
}

h2 {
    font-size: 1.8rem;
    color: #333;
    margin-bottom: 15px;
}

.loading-spinner {
    text-align: center;
    font-size: 1.2rem;
    color: #666;
}

.tabs {
    display: flex;
    margin-bottom: 20px;
}

.tab-button {
    padding: 5px 10px;
    margin-right: 10px;
    border: none;
    background: #ddd;
    cursor: pointer;
    border-radius: 5px;
    font-size: 1.2rem;
}

.tab-button.active {
    background: #0056b3;
    color: white;
}

.raw-data-container {
    margin-top: 20px;
}

.data-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 20px;
}

th,
td {
    padding: 12px;
    text-align: center;
    border: 1px solid #ddd;
}

th {
    background-color: #0056b3;
    color: white;
}

tr:nth-child(even) {
    background-color: #f9f9f9;
}

tr:hover {
    background-color: #f1f1f1;
}

.kpi-section {
    display: flex;
    justify-content: space-between;
}

.kpi-card {
    background-color: #f4f4f4;
    padding: 10px;
    margin: 20px;
    border-radius: 8px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    width: 100%;
    height: auto;
    text-align: center;
    transition: transform 0.3s ease-in-out;
}

.kpi-card:hover {
    transform: scale(1.1);
}

.kpi-card h3 {
    font-size: 1.2rem;
    color: #333;
}

.kpi-card p {
    font-size: 1.5rem;
    font-weight: bold;
    color: #0056b3;
}

.chart-toggle {
    display: flex;
    justify-content: flex-start;
}

.chart-toggle button {
    padding: 5px 10px;
    margin: 0 5px;
    background-color: #f1f1f1;
    border: 1px solid #ddd;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
    font-size: 1rem;
}

.chart-toggle button.active {
    background-color: #0056b3;
    color: white;
}
</style>