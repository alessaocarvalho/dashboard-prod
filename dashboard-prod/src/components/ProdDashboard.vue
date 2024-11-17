<template>
    <div class="dashboard">
        <h1 class="dashboard-title">📊 Dashboard de Produção</h1>

        <!-- Filtros -->
        <div class="filters">
            <div class="filter-group">
                <label for="ordem">Filtrar por Ordem de Produção:</label>
                <select v-model="filtroOrdem" id="ordem">
                    <option value="">Todas</option>
                    <option v-for="indicador in indicadores" :key="indicador.ordem" :value="indicador.ordem">
                        {{ indicador.ordem }}
                    </option>
                </select>
            </div>

            <div class="filter-group">
                <label for="periodo">Filtrar por Período:</label>
                <select v-model="filtroPeriodo" id="periodo">
                    <option value="">Todos</option>
                    <option value="mes">Mês</option>
                    <option value="trimestre">Trimestre</option>
                    <option value="ano">Ano</option>
                </select>
            </div>
        </div>

        <!-- Indicadores -->
        <div class="indicadores">
            <div class="card" v-for="indicador in indicadoresFiltrados" :key="indicador.ordem">
                <h3>Ordem de Produção: {{ indicador.ordem }}</h3>
                <div class="indicator">
                    <span class="indicator-label">Matéria-prima:</span>
                    <span class="indicator-value">{{ indicador.materiaPrima }} Kg</span>
                </div>
                <div class="indicator">
                    <span class="indicator-label">Concentrado:</span>
                    <span class="indicator-value">{{ indicador.concentrado }} Kg</span>
                </div>
                <div class="indicator">
                    <span class="indicator-label">Rendimento:</span>
                    <span class="indicator-value">{{ indicador.rendimento.toFixed(2) }}%</span>
                </div>
            </div>
        </div>

        <!-- Comparação -->
        <div class="chart-container">
            <h2>Comparação de Rendimento</h2>
            <line-chart :data="chartData" />
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
            filtroOrdem: "", // Filtro de ordem
            filtroPeriodo: "", // Filtro de período
        };
    },
    computed: {
        // Filtra os indicadores com base nos filtros de Ordem de Produção e Período
        indicadoresFiltrados() {
            let filtrados = this.indicadores;

            // Filtra por Ordem de Produção
            if (this.filtroOrdem) {
                filtrados = filtrados.filter(
                    (indicador) => indicador.ordem === this.filtroOrdem
                );
            }

            // Filtra por Período
            if (this.filtroPeriodo) {
                const now = new Date();
                filtrados = filtrados.filter((indicador) => {
                    const data = new Date(indicador.data);
                    if (this.filtroPeriodo === "mes") {
                        return data.getMonth() === now.getMonth();
                    } else if (this.filtroPeriodo === "trimestre") {
                        const trimestreAtual = Math.floor(now.getMonth() / 3) + 1;
                        const trimestreIndicador = Math.floor(data.getMonth() / 3) + 1;
                        return trimestreAtual === trimestreIndicador;
                    } else if (this.filtroPeriodo === "ano") {
                        return data.getFullYear() === now.getFullYear();
                    }
                    return true;
                });
            }

            return filtrados;
        },
    },
    methods: {
        // Busca e processa os dados da API
        async fetchData() {
            try {
                const [materiaPrima, concentrado] = await Promise.all([
                    axios.get("http://localhost:3000/api/get_soma_materia_prima"),
                    axios.get("http://localhost:3000/api/get_soma_concentrado"),
                ]);

                // Combina os dados por Ordem de Produção
                const ordemMap = {};

                // Processa Matéria-prima
                materiaPrima.data.forEach((item) => {
                    ordemMap[item["Ordem de Produção"]] = {
                        ordem: item["Ordem de Produção"],
                        materiaPrima: item["﻿Soma de Matéria-prima (Kg)"],
                        concentrado: 0, // Inicializa concentrado com 0
                        data: `${item.Ano}-${item.Mês}-${item.Dia}`, // Adiciona a data
                    };
                });

                // Processa Concentrado
                concentrado.data.forEach((item) => {
                    if (ordemMap[item["﻿Ordem de Produção"]]) {
                        ordemMap[item["﻿Ordem de Produção"]].concentrado =
                            item["﻿Soma de Concentrado (Kg)"];
                    }
                });

                // Calcula rendimento e converte para array
                const indicadores = Object.values(ordemMap).map((indicador) => ({
                    ...indicador,
                    rendimento:
                        (indicador.concentrado / indicador.materiaPrima) * 100 || 0,
                }));

                this.indicadores = indicadores;

                // Configura os dados do gráfico
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
            } catch (error) {
                console.error("Erro ao buscar dados:", error);
            }
        },
    },
    mounted() {
        this.fetchData();
    },
};
</script>

<style scoped>
/* Estrutura básica */
.dashboard {
    font-family: 'Roboto', sans-serif;
    color: #333;
    padding: 20px;
    max-width: 1200px;
    margin: 0 auto;
}

.dashboard-title {
    font-size: 2rem;
    text-align: center;
    margin-bottom: 20px;
    color: #0056b3;
}

/* Filtros */
.filters {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
    gap: 20px;
    flex-wrap: wrap;
}

.filter-group {
    display: flex;
    flex-direction: column;
}

label {
    font-weight: 600;
    margin-bottom: 5px;
    color: #555;
}

select {
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 5px;
    font-size: 1rem;
    width: 100%;
    max-width: 300px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    transition: border-color 0.3s;
}

select:focus {
    border-color: #0056b3;
    outline: none;
}

/* Indicadores */
.indicadores {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    justify-content: center;
}

.card {
    background: #f9f9f9;
    border: 1px solid #e1e1e1;
    border-radius: 10px;
    padding: 20px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    width: 250px;
    transition: transform 0.3s, box-shadow 0.3s;
}

.card:hover {
    transform: translateY(-5px);
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

h3 {
    font-size: 1.2rem;
    color: #0056b3;
    margin-bottom: 15px;
}

.indicator {
    display: flex;
    justify-content: space-between;
    margin: 5px 0;
}

.indicator-label {
    font-weight: 500;
    color: #666;
}

.indicator-value {
    font-weight: 600;
    color: #333;
}

/* Gráfico */
.chart-container {
    margin-top: 30px;
    padding: 20px;
    background: #fff;
    border: 1px solid #e1e1e1;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.chart-container h2 {
    text-align: center;
    font-size: 1.5rem;
    color: #333;
    margin-bottom: 20px;
}

/* Responsividade */
@media (max-width: 768px) {
    .filters {
        flex-direction: column;
        gap: 15px;
    }

    .indicadores {
        justify-content: center;
    }

    .card {
        width: 100%;
        max-width: 400px;
    }
}
</style>