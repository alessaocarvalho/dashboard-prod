<template>
    <div>
        <canvas ref="lineChartCanvas"></canvas>
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
    },
    watch: {
        data: "renderChart",
    },
    methods: {
        renderChart() {
            if (this.chart) this.chart.destroy();

            this.chart = new Chart(this.$refs.lineChartCanvas, {
                type: "line",
                data: this.data,
            });
        },
    },
    mounted() {
        this.renderChart();
    },
};
</script>