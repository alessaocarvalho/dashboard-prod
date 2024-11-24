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
                <!-- Navegação entre abas -->
                <div class="tabs">
                    <button :class="{ active: activeTab === 'line' }" @click="activeTab = 'line'" class="tab-button">
                        Gráfico de Linha
                    </button>
                    <button :class="{ active: activeTab === 'treemap' }" @click="activeTab = 'treemap'"
                        class="tab-button">
                        Treemap
                    </button>
                </div>

                <!-- Conteúdo das abas -->
                <div v-if="activeTab === 'line'" class="chart-container">
                    <h2>Comparação de Rendimento</h2>
                    <line-chart :data="chartData" />
                </div>

                <div v-else-if="activeTab === 'treemap'" class="chart-container">
                    <h2>Visualização em Treemap</h2>
                    <treemap-chart :data="treemapData" />
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import axios from "axios";
import LineChart from "./LineChart.vue";
import TreemapChart from "./TreemapChart.vue";

export default {
    components: { LineChart, TreemapChart },
    data() {
        return {
            indicadores: [], // Lista de dados combinados
            chartData: null, // Dados para o gráfico de linha
            treemapData: null, // Dados para o gráfico de Treemap
            filtroOrdemSelecionadas: [], // Filtro de ordens selecionadas
            loading: false, // Indicador de carregamento
            error: null, // Mensagem de erro
            activeTab: "line", // Aba ativa ("line" ou "treemap")
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

            // Dados para o gráfico de linha
            this.chartData = {
                labels: indicadores.map(i => i.ordem),
                datasets: [
                    {
                        label: "Rendimento (%)",
                        data: indicadores.map(i => i.rendimento),
                        borderColor: "rgba(75, 192, 192, 1)",
                        borderWidth: 2,
                        fill: false,
                    },
                ],
            };

            // Dados para o Treemap
            this.treemapData = indicadores.map(indicador => ({
                name: `Ordem de Produção: ${indicador.ordem}`,  // Nome da ordem de produção
                value: indicador.rendimento,     // Valor que representa a área do retângulo
            }));
        }
        ,
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

.data-table {
    padding: 20px;
}

table {
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

.loading-spinner {
    text-align: center;
    font-size: 1.2rem;
    color: #666;
}

/* Estilos para layout */
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
</style>