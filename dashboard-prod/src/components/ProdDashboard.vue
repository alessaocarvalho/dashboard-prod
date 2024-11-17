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
                <button v-if="filtroOrdemSelecionadas.length > 0" @click="clearFilters" class="clear-filters-btn">Limpar
                    Filtros</button>
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
    margin-top: 20px;
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
</style>