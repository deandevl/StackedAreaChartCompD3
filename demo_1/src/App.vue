<template>
  <div class='demo_container'>
    <button v-on:click="read_data">Plot Federal Spending</button>
    <stacked-area-chart-comp-d3
        title_1="Federal Expenditures by Function"
        title_2="US Office of Management and Budget Historical Tables"
        :axis="axis"
        :chart_data="chart_data"
        :css_variables="css_variables">
      <svg class="svg_chart_1" width="1700" height="700"></svg>
    </stacked-area-chart-comp-d3>
  </div>
</template>

<script>
  import StackedAreaChartCompD3 from 'stackedareachartcompd3';
  import {read_csv} from 'd3fetchmodule';

  export default {
    name: 'App',
    data: function() {
      return {
        chart_data: null,
        axis: {
          x: {
            title: 'Year',
            key: 'year',
            data_type: 'time',
            tic_format: '%Y',
            tic_rotation: 0,
          },
          y: {
            title: 'Dollars',
            tic_format: '.3s',
            groups: [
              {key: 'National_Defense', format: '.1f', color: '#ff8c00'},
              {key: 'International_Affairs', format: '.1f', color: '#d0743c'},
              {key: 'Environment', format: '.1f', color: '#a05d56'},
              {key: 'Science_Space_Tech', format: '.1f', color: '#6b486b'},
              {key: 'Agriculture', format: '.1f', color: '#7b6888'}
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
      read_data: function(){
        const convert = [
          {field: 'year', type: 'time', time_format: '%Y'},
          {field: 'National_Defense', type: 'linear'},
          {field: 'International_Affairs', type: 'linear'},
          {field: 'Environment', type: 'linear'},
          {field: 'Science_Space_Tech', type: 'linear'},
          {field: 'Agriculture', type: 'linear'}
        ];
        const get_data = async () => {
          try{
            this.chart_data = await read_csv('data/federal_spend.csv', convert);
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

<style scoped>

</style>