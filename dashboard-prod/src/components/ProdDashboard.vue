<template>
    <div class="dashboard">
        <header class="dashboard-header">
            <h1 class="dashboard-title">Dashboard de Produção</h1>
            <p class="dashboard-description">
                Acompanhe o desempenho da produção com dados atualizados sobre matéria-prima, concentrado e rendimento. Selecione as ordens de produção para analisar e comparar os resultados.
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
                <button v-if="filtroOrdemSelecionadas.length > 0" @click="clearFilters" class="clear-filters-btn">Limpar Filtros</button>
            </div>

            <!-- Conteúdo principal -->
            <div class="main-content">
                <!-- Gráfico de Rendimento -->
                <div class="chart-container">
                    <h2>Comparação de Rendimento</h2>
                    <line-chart :data="chartData" />
                </div>

                <!-- Tabela de Dados -->
                <div class="data-table">
                    <table v-if="!loading">
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
                                <td>{{ indicador.materiaPrima }} Kg</td>
                                <td>{{ indicador.concentrado }} Kg</td>
                                <td>{{ indicador.rendimento.toFixed(2) }}%</td>
                            </tr>
                        </tbody>
                    </table>
                    <div v-else class="loading-spinner">Carregando dados...</div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import axios from "axios";
import LineChart from "./LineChart.vue";

export default {
    components: { LineChart },
    data() {
        return {
            indicadores: [], // Lista de dados combinados
            chartData: null, // Dados para o gráfico
            filtroOrdemSelecionadas: [], // Filtro de ordens selecionadas
            loading: false, // Indicador de carregamento
            error: null, // Mensagem de erro
        };
    },
    computed: {
        // Filtra os indicadores com base nas ordens de produção selecionadas
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
        normalizeKeys(obj) {
            return Object.keys(obj).reduce((acc, key) => {
                const cleanKey = key.trim().replace(/^\uFEFF/, ""); // Remove caracteres invisíveis
                acc[cleanKey] = obj[key];
                return acc;
            }, {});
        },
        getNumericMonth(monthName) {
            const monthMap = {
                janeiro: 1,
                fevereiro: 2,
                março: 3,
                abril: 4,
                maio: 5,
                junho: 6,
                julho: 7,
                agosto: 8,
                setembro: 9,
                outubro: 10,
                novembro: 11,
                dezembro: 12,
            };
            return monthMap[monthName.toLowerCase()] || 0;
        },
        formatDate(year, month, day) {
            return `${year}-${String(month).padStart(2, "0")}-${String(day).padStart(2, "0")}`;
        },
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

                const materiaPrima = Array.isArray(materiaPrimaResponse.data)
                    ? materiaPrimaResponse.data.map(this.normalizeKeys)
                    : [this.normalizeKeys(materiaPrimaResponse.data)];

                const concentrado = Array.isArray(concentradoResponse.data)
                    ? concentradoResponse.data.map(this.normalizeKeys)
                    : [this.normalizeKeys(concentradoResponse.data)];

                const ordemMap = {};

                materiaPrima.forEach((item) => {
                    ordemMap[item["Ordem de Produção"]] = {
                        ordem: item["Ordem de Produção"],
                        materiaPrima: item["Soma de Matéria-prima (Kg)"] || 0,
                        concentrado: 0,
                        data: this.formatDate(
                            item.Ano,
                            this.getNumericMonth(item.Mês),
                            item.Dia
                        ),
                    };
                });

                concentrado.forEach((item) => {
                    if (ordemMap[item["Ordem de Produção"]]) {
                        ordemMap[item["Ordem de Produção"]].concentrado =
                            item["Soma de Concentrado (Kg)"] || 0;
                    }
                });

                const indicadores = Object.values(ordemMap).map((indicador) => ({
                    ...indicador,
                    rendimento:
                        (indicador.concentrado / indicador.materiaPrima) * 100 || 0,
                }));

                this.indicadores = indicadores;
                this.updateChart();
            } catch (error) {
                this.error = "Erro ao carregar os dados, tente novamente mais tarde.";
                console.error(error);
            } finally {
                this.loading = false;
            }
        },
        updateChart() {
            const indicadores = this.indicadoresFiltrados;

            if (indicadores.length === 1) {
                // Exibir gráfico de barra (coluna) para uma única ordem
                const indicador = indicadores[0];

                this.chartData = {
                    labels: ["Matéria-prima", "Concentrado"],
                    datasets: [
                        {
                            label: `Ordem ${indicador.ordem}`,
                            data: [indicador.materiaPrima, indicador.concentrado],
                            backgroundColor: ["#FF8C00", "#4CAF50"],
                            borderColor: ["#FF7F00", "#388E3C"],
                            borderWidth: 1,
                        },
                    ],
                };
            } else {
                // Exibir gráfico de linha para múltiplas ordens
                this.chartData = {
                    labels: indicadores.map((i) => i.ordem),
                    datasets: [
                        {
                            label: "Rendimento (%)",
                            data: indicadores.map((i) => i.rendimento),
                            borderColor: "rgba(75, 192, 192, 1)",
                            borderWidth: 2,
                            fill: false,
                        },
                    ],
                };
            }
        },
    },
    watch: {
        filtroOrdemSelecionadas() {
            this.updateChart();
        },
    },
    mounted() {
        this.fetchData();
    },
};
</script>

<style scoped>
/* Estilos gerais */
.dashboard {
    font-family: 'Roboto', sans-serif;
    color: #333;
    padding: 20px;
    max-width: 1200px;
    margin: 0 auto;
}

.dashboard-header {
    text-align: center;
    margin-bottom: 40px;
}

.dashboard-title {
    font-size: 2.5rem;
    color: #0056b3;
    margin-bottom: 10px;
}

.dashboard-description {
    font-size: 1rem;
    color: #666;
    max-width: 800px;
    margin: 0 auto;
}

/* Barra lateral */
.sidebar {
    width: 250px;
    float: left;
    padding: 20px;
    background-color: #f9f9f9;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

h3 {
    font-size: 1.2rem;
    color: #0056b3;
    margin-bottom: 15px;
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
    padding: 8px 15px;
    background-color: #f44336;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 1rem;
}

.clear-filters-btn:hover {
    background-color: #d32f2f;
}

/* Conteúdo principal */
.main-content {
    margin-left: 280px;
    padding: 20px;
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
    background-color: #fff;
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 20px;
}

th, td {
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
</style>