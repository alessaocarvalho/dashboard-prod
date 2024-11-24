<template>
    <div>
        <canvas ref="treemapCanvas"></canvas>
    </div>
</template>

<script>
import { Chart, registerables } from "chart.js";
import { TreemapController, TreemapElement } from "chartjs-chart-treemap";

Chart.register(...registerables, TreemapController, TreemapElement);

export default {
    props: {
        data: {
            type: Array,
            required: true,
        },
    },
    watch: {
        data: "renderChart",  // Quando os dados mudarem, renderiza o gráfico novamente
    },
    methods: {
        renderChart() {
            // Destruir o gráfico existente para evitar múltiplas instâncias
            if (this.chart) this.chart.destroy();

            // Criando o gráfico do tipo "treemap"
            this.chart = new Chart(this.$refs.treemapCanvas, {
                type: "treemap",  // Tipo de gráfico
                data: {
                    datasets: [
                        {
                            tree: this.data,  // Dados a serem passados para o gráfico
                            key: "value",      // Chave que contém os valores (área dos retângulos)
                            groups: ["name"],  // Chave que será usada como grupo (nome das ordens de produção)
                            backgroundColor: "rgba(75, 192, 192, 0.6)",
                            borderColor: "rgba(75, 192, 192, 1)",
                            borderWidth: 1,
                        },
                    ],
                },
            });
        },
    },
    mounted() {
        this.renderChart();  // Renderiza o gráfico ao montar o componente
    },
};
</script>