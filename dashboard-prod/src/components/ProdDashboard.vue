<template>
    <div class="dashboard">
        <header class="dashboard-header">
            <h1 class="dashboard-title">Dashboard de Produção Nutrilite</h1>
            <p class="dashboard-description">
                Acompanhe o desempenho da produção com dados atualizados sobre matéria-prima, concentrado e rendimento.
                Selecione as ordens de produção para analisar e comparar os resultados.
            </p>
        </header>

        <div class="content">
            <!-- Barra de Navegação Lateral -->
            <div class="sidebar">
                <h3>Ordens de Produção</h3>
                <p>Selecione para comparar</p>
                <div class="order-list">
                    <label v-for="indicador in indicadores" :key="indicador.ordem" class="order-item">
                        <input type="checkbox" :value="indicador.ordem" v-model="filtroOrdemSelecionadas" />
                        Ordem {{ indicador.ordem }}
                    </label>
                </div>
                <button v-if="filtroOrdemSelecionadas.length > 0" @click="clearFilters" class="clear-filters-btn">
                    Limpar Filtros
                </button>
            </div>

            <!-- Conteúdo principal -->
            <div class="main-content">
                <!-- KPIs -->
                <div class="kpi-section">
                    <div class="kpi-card">
                        <h3>Ordens de Produção Selecionadas</h3>
                        <p>{{ filtroOrdemSelecionadas.length }} de {{ indicadores.length }}</p>
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
                    <button :class="{ active: activeTab === 'line' }" @click="activeTab = 'line'" class="tab-button">
                        Gráfico de Linha
                    </button>
                    <button :class="{ active: activeTab === 'bar' }" @click="activeTab = 'bar'" class="tab-button">
                        Gráfico de Barras
                    </button>
                    <button :class="{ active: activeTab === 'relation' }" @click="activeTab = 'relation'"
                        class="tab-button">
                        Matéria-prima vs Concentrado
                    </button>
                    <button :class="{ active: activeTab === 'rawData' }" @click="activeTab = 'rawData'"
                        class="tab-button">
                        Dados Brutos
                    </button>
                </div>

                <!-- Conteúdo das abas -->
                <div v-if="activeTab === 'line'" class="chart-container">
                    <h2>Comparação de Rendimento (%)</h2>
                    <line-chart :data="chartData" />
                </div>

                <div v-else-if="activeTab === 'bar'" class="chart-container">
                    <h2>Comparação de Rendimento (%)</h2>
                    <bar-chart :data="barChartData" />
                </div>

                <div v-else-if="activeTab === 'relation'" class="chart-container">
                    <h2>Relação Matéria-prima vs Concentrado</h2>
                    <line-chart :data="relationChartData" />
                </div>

                <div v-else-if="activeTab === 'rawData'" class="raw-data-container">
                    <h2>Dados Brutos</h2>
                    <table class="data-table">
                        <thead>
                            <tr>
                                <th>Ordem de Produção</th>
                                <th>Matéria-prima (Kg)</th>
                                <th>Concentrado (Kg)</th>
                                <th>Rendimento (%)</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="indicador in indicadoresFiltrados" :key="indicador.ordem">
                                <td>{{ indicador.ordem }}</td>
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
            chartData: null, // Dados para o gráfico de linha (rendimento)
            barChartData: null, // Dados para o gráfico de barras
            relationChartData: null, // Dados para o gráfico de linha de relação (matéria-prima vs concentrado)
            filtroOrdemSelecionadas: [], // Filtro de ordens selecionadas
            loading: false, // Indicador de carregamento
            error: null, // Mensagem de erro
            activeTab: "line", // Aba ativa ("line", "bar", "relation", ou "rawData")
        };
    },
    computed: {
        indicadoresFiltrados() {
            if (this.filtroOrdemSelecionadas.length === 0) {
                return this.indicadores;
            }
            return this.indicadores.filter(indicador =>
                this.filtroOrdemSelecionadas.includes(indicador.ordem)
            );
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
        clearFilters() {
            this.filtroOrdemSelecionadas = [];
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
                this.updateCharts();
            } catch (error) {
                this.error = "Erro ao carregar os dados, tente novamente mais tarde.";
                console.error(error);
            } finally {
                this.loading = false;
            }
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
                        backgroundColor: "rgba(75, 192, 192, 0.2)", // Cor de fundo suave
                        borderWidth: 3,
                        pointBackgroundColor: "rgba(75, 192, 192, 1)", // Ponto de dados
                        pointRadius: 5,
                        fill: true, // Preenchimento suave abaixo da linha
                        tension: 0.4,
                    },
                ],
            };

            // Dados para o gráfico de barras
            this.barChartData = {
                labels: indicadores.map(i => `Ordem ${i.ordem}`),
                datasets: [
                    {
                        label: "Rendimento (%)",
                        data: indicadores.map(i => i.rendimento),
                        backgroundColor: "rgba(75, 192, 192, 0.7)",
                        borderColor: "rgba(75, 192, 192, 1)",
                        borderWidth: 2,
                        hoverBackgroundColor: "rgba(75, 192, 192, 1)", // Cor ao passar o mouse
                        hoverBorderColor: "rgba(75, 192, 192, 1)",
                        borderRadius: 5, // Barras arredondadas
                        barThickness: 20,
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
                        borderDash: [5, 5], // Linha pontilhada
                        fill: false,
                        tension: 0.4,
                    },
                    {
                        label: "Concentrado (Kg)",
                        data: indicadores.map(i => i.concentrado),
                        borderColor: "rgba(54, 162, 235, 1)",
                        borderWidth: 3,
                        fill: false,
                        tension: 0.4,
                    },
                ],
            };
        },
    },
    watch: {
        filtroOrdemSelecionadas() {
            this.updateCharts();
        },
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

.order-list {
    margin-bottom: 20px;
}

.order-item {
    display: block;
    margin: 8px 0;
    font-size: 1rem;
}

.clear-filters-btn {
    margin: 10px;
    padding: 10px;
    background-color: white;
    color: #0056b3;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 1rem;
}

.clear-filters-btn:hover {
    background-color: #4db8ff;
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
    padding: 10px 20px;
    margin-right: 10px;
    border: none;
    background: #ddd;
    cursor: pointer;
    border-radius: 5px;
    font-size: 1rem;
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
    gap: 20px;
    margin-bottom: 30px;
}

.kpi-card {
    background-color: #f4f4f4;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    width: 100%;
    height: auto;
    text-align: center;
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
</style>