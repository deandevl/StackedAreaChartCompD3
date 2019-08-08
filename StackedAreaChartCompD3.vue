<template>
  <div>
    <div class="stackedAreaCompD3">
      <section :class="[scaling_is_active ? 'stackedAreaCompD3_scaling' : 'stackedAreaCompD3_not_scaling']">
        <input-comp
            header_position="above"
            placeholder="Enter min"
            input_size="7"
            :heading="head_min"
            :input_value="min"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.min = value}">
        </input-comp>
        <input-comp
            header_position="above"
            placeholder="Enter max"
            input_size="7"
            :heading="head_max"
            :input_value="max"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.max = value}">
        </input-comp>
        <input-comp
            header_position="above"
            placeholder="Enter step"
            input_size="7"
            :heading="head_step"
            :input_value="step"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.step = value}">
        </input-comp>
        <select-comp
            v-show="doing_time"
            heading="step time unit"
            placeholder="Select step time"
            :select_value="select_step_time"
            :items="time_intervals"
            :single_border="single_border"
            :blur_panel="blur_panel"
            :css_variables="select_css_variables"
            v-on:select_comp_value_changed="value => {this.select_step_time = value}">
        </select-comp>
        <input-comp
            header_position="above"
            placeholder="Enter tick format"
            input_size="14"
            :heading="head_tick_format"
            :input_value="tick_format"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.tick_format = value}">
        </input-comp>
        <button-comp
            :css_variables="button_css_variables"
            v-on:button_comp_clicked="update_scaling_axis">Update
        </button-comp>
        <button-comp
            :css_variables="button_css_variables"
            v-on:button_comp_clicked="reset_scaling">Reset
        </button-comp>
      </section>
      <section class="stackedAreaCompD3_svg__sec">
        <slot></slot>
      </section>
    </div>
  </div>
</template>

<script>

  import ButtonComp from 'buttoncomp';
  import InputComp from 'inputcomp';
  import SelectComp from 'selectcomp';
  import {define_scale,minmax,calculate_scale,color_scale,time_interval} from 'd3ScaleModule';
  import {define_axis,update_axis} from 'd3axismodule';
  import {make_draggable,init_chart} from 'd3utilmodule';
  import {timeParse,timeFormat} from 'd3-time-format';
  import {stack,area} from 'd3-shape';
  import {max,range} from 'd3-array';
  import {select} from 'd3-selection';

  export default {
    name: 'StackedAreaChartCompD3',
    data(){
      return {
        stack: null,
        data_stack: null,
        group_keys: null,
        min: null,
        max: null,
        step: null,
        tick_format: null,
        current_min: {x: null, y: null},
        current_max: {x: null, y: null},
        current_step: {x: null, y: null},
        current_tick_format: {x: null, y: null},
        initials: {x: {min: null, max: null, tick_values: null, tick_format: null},
          y: {min: null, max: null, tick_values: null, tick_format: null}},
        current_axis: null,
        head_min: null,
        head_max: null,
        head_step: null,
        head_tick_format: null,
        chart_axis: {x: null,y: null},
        scale: {x: null,y: null, color: null},
        pixels: {x: null,y: null},
        chart_g: null,
        doing_time: false,
        time_intervals: ['auto','year','month','week','day','hour','minute','second','millisecond'],

        button_css_variables: {
          button_comp_font_size: '.6rem'
        },

        scaling_is_active: false,

        //input-comp stuff
        single_border: true,

        //select=comp stuff
        select_step_time: 'auto',
        blur_panel: false,
        select_css_variables: {
          select_comp_items_panel_position: 'absolute',
          select_comp_items_panel_z_index: '100'
        }
      }
    },
    props: {
      chart_data: {
        type: Array,
        default: null
      },
      axis: {
        type: Object,
        default: null
      },
      title_1: {
        type: String,
        default: null
      },
      title_2: {
        type: String,
        default: null
      },
      margin_left: {
        type: Number,
        default: 80
      },
      margin_bottom: {
        type: Number,
        default: 50
      },
      css_variables: {
        type: Object,
        default: null
      }
    },
    watch: {
      chart_data: function(new_data){
        if(new_data !== null){
          //define stack
          this.group_keys = this.axis.y.groups.map(d => {return d.key;});
          this.stack = stack().keys(this.group_keys);

          //remove all current elements
          this.remove_elements();

          //define x scaling and axis
          this.define_x_scale_axis();

          //define y scaling and axis
          this.define_y_scale_axis();

          //draw the areas
          this.draw_areas();

          //draw axis titles
          this.draw_axis_titles();

          //draw main titles
          this.draw_main_titles();

          //draw legend
          this.draw_legend();
        }
      }
    },
    computed: {},
    methods: {
      define_x_scale_axis: function(){
        let x_nice_scale = null;
        let x_nice_min_max = null;
        let tick_values = null;

        switch(this.axis.x.data_type){
          case 'linear':{
            //compute overall raw x min/max of numeric data
            const x_min_max = minmax(this.chart_data, [this.axis.x.key]);
            x_nice_scale = calculate_scale(x_min_max[0],x_min_max[1]);
            x_nice_min_max = [x_nice_scale.min,x_nice_scale.max];
            //update currents
            this.set_currents('x',x_nice_scale.min,x_nice_scale.max,x_nice_scale.step,this.axis.x.tic_format);
            break;
          }
          case 'band':
            x_nice_min_max = this.axis.x.bands;
            break;
          case 'time':{
            x_nice_min_max = minmax(this.chart_data, [this.axis.x.key]);
            //update currents
            this.set_currents('x',x_nice_min_max[0],x_nice_min_max[1],'auto',this.axis.x.tic_format);
            break;
          }
        }

        //define x scaling
        this.scale.x = define_scale(
          this.axis.x.data_type,
          'x',
          x_nice_min_max,
          this.pixels.x
        );

        //define tick_values
        if(this.axis.x.data_type === 'time'){
          tick_values = this.scale.x.ticks();
        }else{
          tick_values = range(this.scale.x.domain()[0], this.scale.x.domain()[1], x_nice_scale.step);
        }

        //define initials
        this.set_initials('x',x_nice_min_max[0],x_nice_min_max[1],tick_values,this.axis.x.tic_format);

        //define x axis
        this.chart_axis.x = define_axis(
          'bottom',
          this.axis.x.data_type,
          'stackedAreaCompD3_axis stackedAreaCompD3_axis_x',
          this.scale.x,
          this.chart_g,
          this.pixels.y,
          this.axis.x.tic_format,
          tick_values
        );

        //add click event to x axis
        const axis_group = this.chart_g.select('.stackedAreaCompD3_axis_x');
        axis_group
          .on('click',() => {
            this.current_axis = 'x';
            this.head_min = 'x minimum';
            this.head_max = 'x maximum';
            this.head_step = 'x step';
            this.head_tick_format = 'x tick format';
            this.min = this.current_min.x;
            this.max = this.current_max.x;
            this.step = this.current_step.x;
            this.tick_format = this.current_tick_format.x;
            if(this.axis.x.data_type === 'time'){
              this.doing_time = true;
              this.min = null;
              this.max = null;
            }
            this.scaling_is_active = !this.scaling_is_active;
          });

        //address tic label rotation on x axis
        let x_tic_rotation = 0.0;
        let x_tic_dx = 0.0;
        let x_tic_dy = 0.6;
        let x_tic_anchor = 'middle';
        if(this.axis.x.tic_rotation !== undefined && this.axis.x.tic_rotation !== 0){
          x_tic_rotation = this.axis.x.tic_rotation;
          x_tic_dx = -0.8;
          x_tic_dy = 0.15;
          x_tic_anchor = 'end';
        }
        const x_axis_group = this.chart_g.select('.stackedAreaCompD3_axis_x');
        x_axis_group
          .selectAll('text')
          .attr('text-anchor', x_tic_anchor)
          .attr('dx', `${x_tic_dx}em`)
          .attr('dy', `${x_tic_dy}em`)
          .attr('transform', `rotate(${x_tic_rotation})`);
      },
      define_y_scale_axis: function(){
        // Note: y scaling is 'linear'
        //compute overall min/max of data
        this.data_stack = this.stack(this.chart_data);
        const y_min_max = minmax(this.chart_data, this.group_keys);
        const y_values = [];
        this.data_stack[this.group_keys.length-1].map(d => {
          y_values.push(d[1]);
        });
        y_min_max[1] = max(y_values);
        const y_nice_scale = calculate_scale(0,y_min_max[1]);
        const tick_values = range(y_nice_scale.min,y_nice_scale.max,y_nice_scale.step);

        //define y scaling
        this.scale.y = define_scale(
          'linear',
          'y',
          [y_nice_scale.min,y_nice_scale.max],
          this.pixels.y);

        //define y axis
        this.chart_axis.y = define_axis(
          'left',
          'linear',
          'stackedAreaCompD3_axis stackedAreaCompD3_axis_y',
          this.scale.y,
          this.chart_g,
          this.pixels['y'],
          this.axis.y.tic_format,
          tick_values
        );

        //update currents
        this.set_currents('y',y_nice_scale.min,y_nice_scale.max,y_nice_scale.step,this.axis.y.tic_format);
        //define initials
        this.set_initials('y',y_nice_scale.min,y_nice_scale.max,tick_values,this.axis.y.tic_format);

        //add click event to y axis
        const axis_group = this.chart_g.select('.stackedAreaCompD3_axis_y');
        axis_group
          .on('click',() => {
            this.current_axis = 'y';
            this.head_min = 'y minimum';
            this.head_max = 'y maximum';
            this.head_step = 'y step';
            this.head_tick_format = 'y tick format';
            this.min = this.current_min.y;
            this.max = this.current_max.y;
            this.step = this.current_step.y;
            this.tick_format = this.current_tick_format.y;
            this.doing_time = false;
            this.scaling_is_active = !this.scaling_is_active;
          });


        //define group color scaling
        const colors = this.axis.y.groups.map(d => {return d.color;});
        this.scale.color = color_scale(colors);
      },
      draw_areas: function(){
        this.chart_g.append('g')
          .attr('class','stackedAreaCompD3_areas_g')
          .selectAll('path').data(this.data_stack).enter()
          .append('path')
          .style('fill',(d,i) => {
            return this.scale.color(i);
          })
          .attr('d',area()
            .x((d,i) => {
              return this.scale.x(this.chart_data[i][this.axis.x.key]);
            })
            .y0(d => {
              return this.scale.y(d[0]);
            })
            .y1(d => {
              return this.scale.y(d[1]);
            })
          );
      },
      draw_axis_titles: function(){
        //draw x axis title
        const axis_title_x_g = this.chart_g.append('g')
          .attr('class','stackedAreaCompD3_axis_title stackedAreaCompD3_axis_title_x');
        axis_title_x_g.append('text')
          .attr('x',this.pixels.x / 3)
          .attr('y',this.pixels.y+this.margin_bottom-10)
          .text(this.axis.x.title);
        make_draggable(axis_title_x_g);

        //draw y axis title
        const axis_title_y_g = this.chart_g.append('g')
          .attr('class','stackedAreaCompD3_axis_title stackedAreaCompD3_axis_title_y');
        axis_title_y_g.append('text')
          .attr('transform','rotate(-90)')
          .attr('x',-0.67 * this.pixels.y)
          .attr('y',-this.margin_left+30)
          .text(this.axis.y.title);
        make_draggable(axis_title_y_g);
      },
      draw_main_titles: function(){
        const title_1_g = this.chart_g.append('g')
          .attr('class','stackedAreaCompD3_title_g stackedAreaCompD3_title_g_1');
        title_1_g.append('text')
          .attr('transform',`translate(${this.margin_left+this.pixels.x/2},30)`)
          .attr('text-anchor','middle')
          .style('font-size','1.3rem')
          .text(this.title_1);
        make_draggable(title_1_g);

        const title_2_g = this.chart_g.append('g')
          .attr('class','stackedAreaCompD3_title_g stackedAreaCompD3_title_g_2');
        title_2_g.append('text')
          .attr('transform',`translate(${this.margin_left+this.pixels.x/2},50)`)
          .attr('text-anchor','middle')
          .style('font-size','.8rem')
          .text(this.title_2);
        make_draggable(title_2_g);
      },
      draw_legend: function(){
        const legends_g = this.chart_g.append('g')
          .attr('class','stackedAreaCompD3_legends_g');
        const legend = legends_g.selectAll('g').data(this.group_keys).enter()
          .append('g')
          .attr('transform', (d, i) => {
            return `translate(0,${i * 20})`;
          });
        legend.append('rect')
          .attr('class', 'stackedAreaCompD3_rect_legend')
          .attr('x', this.pixels.x - 19)
          .attr('width', 19)
          .attr('height', 19)
          .attr('fill', (d, i) => {
            return this.scale.color(i);
          });
        legend.append('text')
          .attr('x', this.pixels.x - 24)
          .attr('y', 9.5)
          .attr('dy', '0.32em')
          .attr('font-family', 'Verdana')
          .attr('font-size', 10)
          .attr('text-anchor', 'end')
          .text(d => {
            return d;
          });
        //make legend draggable
        make_draggable(legends_g);
      },
      update_scaling_axis: function(){
        //redefine domain scaling for current axis
        let tick_values = null;
        let nice_minmax = [];
        let data_type = this.axis[this.current_axis].data_type;
        if(this.current_axis === 'y'){
          data_type = 'linear';
        }
        const axis_group = this.chart_g.select('.stackedAreaCompD3_axis_' + this.current_axis);

        switch(data_type){
          case 'linear':{
            const min = parseFloat(this.min);
            const max = parseFloat(this.max);
            tick_values = range(min,max,parseFloat(this.step));
            nice_minmax.push(min);
            nice_minmax.push(max);

            //update scaling
            this.scale[this.current_axis].domain(nice_minmax);

            //update axis
            update_axis(
              this.chart_axis[this.current_axis],
              axis_group,
              this.scale[this.current_axis],
              'linear',
              this.tick_format,
              tick_values
            );

            //update currents
            this.set_currents(this.current_axis,nice_minmax[0],nice_minmax[1],parseFloat(this.step),this.tick_format);
            break;
          }
          case 'time': {
            const parseTime = timeParse(this.tick_format);
            nice_minmax = [parseTime(this.min),parseTime(this.max)];

            //update scaling
            this.scale[this.current_axis].domain(nice_minmax);

            //update axis
            if(this.select_step_time !== 'auto'){
              //update currents
              this.set_currents(this.current_axis,nice_minmax[0],nice_minmax[1],parseFloat(this.step),this.tick_format);
              const tick_generator = time_interval[this.select_step_time].every(this.current_step[this.current_axis]);
              tick_values = tick_generator.range(
                this.scale[this.current_axis].domain()[0],
                this.scale[this.current_axis].domain()[1]);
            }else {
              //update currents
              this.set_currents(this.current_axis,nice_minmax[0],nice_minmax[1],'auto',this.tick_format);
              tick_values = this.scale[this.current_axis].ticks();
            }
            update_axis(
              this.chart_axis[this.current_axis],
              axis_group,
              this.scale[this.current_axis],
              'time',
              this.tick_format,
              tick_values
            );
            break;
          }
          case 'band': {
            nice_minmax.push(this.min);
            nice_minmax.push(this.max);
            //update scaling
            //TODO this needs updating:
            this.scale[this.current_axis].domain(this.axis[this.current_axis].bands);
            //update axis
            update_axis(
              this.chart_axis[this.current_axis],
              axis_group,
              this.scale[this.current_axis],
              null,
              null,
              this.axis[this.current_axis].bands
            );
          }
        }

        //redraw the areas with updated scaling
        this.redraw_areas();
      },
      reset_scaling: function(){
        //reset x scale
        this.scale.x.domain([this.initials.x.min,this.initials.x.max]);
        //update axis
        const x_axis_group = this.chart_g.select('.stackedAreaCompD3_axis_x');

        update_axis(
          this.chart_axis.x,
          x_axis_group,
          this.scale.x,
          this.axis.x.data_type,
          this.initials.x.tick_format,
          this.initials.x.tick_values
        );

        //update x currents
        this.set_currents('x',this.initials.x.min,this.initials.x.max,'auto',this.initials.x.tick_format);
        if(this.axis.x.data_type !== undefined && this.axis.x.data_type === 'time'){
          this.select_step_time = 'auto';
        }
        //reset y scale
        this.scale.y.domain([this.initials.y.min,this.initials.y.max]);
        //update axis
        const y_axis_group = this.chart_g.select('.stackedAreaCompD3_axis_y');
        update_axis(
          this.chart_axis.y,
          y_axis_group,
          this.scale.y,
          'linear',
          this.initials.y.tick_format,
          this.initials.y.tick_values
        );

        //update y currents
        this.set_currents('y',this.initials.y.min,this.initials.y.max,'auto',this.initials.y.tick_format);

        //redraw areas
        this.redraw_areas();
      },
      redraw_areas: function(){
        const data = [];
        const x_values = [];
        for(let i=0; i < this.chart_data.length; i++){
          if(this.chart_data[i][this.axis.x.key] >= this.scale.x.domain()[0]
            && this.chart_data[i][this.axis.x.key] <= this.scale.x.domain()[1])
          {
            data.push(this.chart_data[i]);
            x_values.push(this.chart_data[i][this.axis.x.key]);
          }
        }
        const data_stack = this.stack(data);
        //redefine bottom with possible minimum change
        data_stack[0].forEach(d => {
          d[0] = this.scale.y.domain()[0];
        });

        select('.stackedAreaCompD3_areas_g')
          .selectAll('path').data(data_stack)
          .attr('d',area()
            .x((d,i) => {
              return this.scale.x(x_values[i]);
            })
            .y0((d,i) => {
              return this.scale.y(d[0]);
            })
            .y1(d => {
              return this.scale.y(d[1]);
            }));
      },
      set_currents(xy,min,max,step,tick_format){
        if(this.axis[xy].data_type !== undefined && this.axis[xy].data_type === 'time'){
          this.current_min[xy] = timeFormat(tick_format)(min);
          this.current_max[xy] = timeFormat(tick_format)(max);
          this.current_step[xy] = step;
          this.current_tick_format[xy] = tick_format;
        }else {
          this.current_min[xy] = min;
          this.current_max[xy] = max;
          this.current_step[xy] = step;
          this.current_tick_format[xy] = tick_format;
        }

        //update gui
        if(this.current_axis === xy){
          this.min = this.current_min[xy];
          this.max = this.current_max[xy];
          this.step = this.current_step[xy];
          this.tick_format = this.current_tick_format[xy];
        }
      },
      set_initials(xy,min,max,tick_values,tick_format){
        this.initials[xy].min = min;
        this.initials[xy].max = max;
        this.initials[xy].tick_values = tick_values;
        this.initials[xy].tick_format = tick_format;
      },
      remove_elements: function(){
        //remove x axis
        this.chart_g.select('.stackedAreaCompD3_axis_x').remove();
        //remove y axis
        this.chart_g.select('.stackedAreaCompD3_axis_y').remove();
        //remove areas
        this.chart_g.select('.stackedAreaCompD3_areas_g').remove();
        //remove axis titles
        this.chart_g.select('.stackedAreaCompD3_axis_title_x').remove();
        this.chart_g.select('.stackedAreaCompD3_axis_title_y').remove();
        //remove main titles
        this.chart_g.select('.stackedAreaCompD3_title_g_1').remove();
        this.chart_g.select('.stackedAreaCompD3_title_g_2').remove();
        //remove legend
        this.chart_g.select('.stackedAreaCompD3_legends_g').remove();
      }
    },
    components: {
      ButtonComp,
      InputComp,
      SelectComp
    },
    mounted(){
      //locate svg
      const svg_class = this.$slots.default[0].elm.classList[0];
      const svg = select(`.${svg_class}`);

      const chart = init_chart(svg,this.margin_left,undefined,undefined,this.margin_bottom);
      this.pixels.x = chart.chart_width;
      this.pixels.y = chart.chart_height;
      this.chart_g = chart.chart_g;

      if(this.css_variables !== null){
        for(let key of Object.keys(this.css_variables)){
          this.$el.style.setProperty(`-- ${key}`, this.css_variables[key]);
        }
      }
    }
  }
</script>

<style lang="less">
  :root {
    --stacked_area_chart_compD3_font_family: Verdana,serif;
    --stacked_area_chart_compD3_color: black;
    --stacked_area_chart_compD3_background_color: white;

    --stacked_area_chart_compD3_axis_font_size: .8rem;
    --stacked_area_chart_compD3_axis_color: black;

    --stacked_area_chart_compD3_fill_opacity: 1.0;

    --stacked_area_chart_compD3_tooltip_fill: black;
  }

  .buttonComp, .selectComp{
    align-self: center;
  }

  .stackedAreaCompD3 {
    display: flex;
    flex-direction: column;
    font-family: var(--stacked_area_chart_compD3_font_family);
    color: var(--stacked_area_chart_compD3_color);
    background-color: var(--stacked_area_chart_compD3_background_color);

    &_title_g {
      font-weight: bold;
      fill: var(--stacked_area_chart_compD3_color);
      cursor: move;
    }

    &_axis_title{
      cursor: move;
      font-weight: bold;
      fill: var(--stacked_area_chart_compD3_axis_color);
    }

    &_axis {
      cursor: pointer;
    }

    &_axis path,
    &_axis line{
      shape-rendering: crispEdges;
      color: var(--stacked_area_chart_compD3_axis_color);
    }
    &_axis text {
      font-weight: bold;
      font-family: var(--stacked_area_chart_compD3_font_family);
      font-size: var(--stacked_area_chart_compD3_axis_font_size);
      color: var(--stacked_area_chart_compD3_axis_color);
    }

    &_scaling {
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      width: 780px;
      margin: 10px 0 10px 20px;
    }

    &_not_scaling {
      display: none;
    }

    &_tooltip {
      border: none;
      margin: 0;
      text-anchor: middle;
      font-weight: bold;
      font-size: 12px;
      font-family: var(--stacked_area_chart_compD3_font_family);
      fill: var(--stacked_area_chart_compD3_tooltip_fill)
    }

    &_tooltip__hidden {
      display: none;
    }

    &_legends_g {
      cursor: move;
    }
  }
</style>