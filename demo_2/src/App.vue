<template>
  <div class='demo_container'>
    <button v-on:click="plot_time_chart">Plot Time Chart</button>
    <stacked-area-chart-comp-d3
        title_1="Demo_2 for StackedAreaChartCompD3"
        title_2="(Year-Month-Day time format for one group)"
        :axis="axis"
        :chart_data="chart_data"
        :css_variables="css_variables">
      <svg class="svg_chart_1" width="1200" height="700"></svg>
    </stacked-area-chart-comp-d3>
  </div>
</template>

<script>
  import StackedAreaChartCompD3 from 'stackedareachartcompd3';
  import {read_json} from 'd3fetchmodule';

  export default {
    name: 'App',
    data: function() {
      return {
        chart_data: null,
        axis: {
          x: {
            title: 'Date',
            key: 'date',
            data_type: 'time',
            tic_format: '%Y-%m',
            tic_rotation: 0,
          },
          y: {
            title: 'Values',
            tic_format: '.3s',
            groups: [
              {key: 'value', format: '.0f', color: '#ff8c00'}
            ]
          },
        },
        css_variables: {}
      };
    },
    components: {
      StackedAreaChartCompD3
    },
    methods: {
      plot_time_chart: function(){
        const convert = [
          {field: 'date', type: 'time', time_format: '%Y-%m-%d'},
          {field: 'value', type: 'linear'}
        ];
        const get_data = async () => {
          try{
            this.chart_data = await read_json('data/data.json', convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      }
    }
  }
</script>

<style>
  .demo_container {
    background-color: black;
  }
</style>