<template>
    <div>
        <Bar :data="data" :options="options" />
    </div>
</template>

<script>
import { Bar } from "vue-chartjs";
import { Chart as ChartJS, CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend } from "chart.js";

ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend);

export default {
    name: "BarChart",
    components: {
        Bar,
    },
    props: {
        data: {
            type: Object,
            required: true,
        },
    },
    data() {
        return {
            options: {
                responsive: true,
                plugins: {
                    legend: {
                        position: "top",
                    },
                    tooltip: {
                        mode: "index",
                        intersect: false,
                    },
                },
            },
        };
    },
};
</script>