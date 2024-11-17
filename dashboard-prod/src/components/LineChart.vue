<template>
    <div>
        <h3>{{ title }}</h3>
        <canvas ref="canvas"></canvas>
    </div>
</template>

<script>
import { Chart, registerables } from "chart.js";
Chart.register(...registerables);

export default {
    props: {
        data: {
            type: Object,
            required: true,
        },
        title: {
            type: String,
            default: "Gráfico",
        },
    },
    watch: {
        data: "renderChart",
    },
    mounted() {
        this.renderChart();
    },
    methods: {
        renderChart() {
            if (this.chart) this.chart.destroy();
            this.chart = new Chart(this.$refs.canvas, {
                type: "line",
                data: this.data,
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                },
            });
        },
    },
    beforeUnmount() {
        if (this.chart) this.chart.destroy();
    },
};
</script>

<style scoped>
canvas {
    width: 100% !important;
    height: 400px !important;
}
</style>