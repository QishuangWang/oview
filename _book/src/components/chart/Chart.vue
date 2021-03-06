<template>
  <div
    :style="{
      backgroundColor: backgroundColor,
      width: width + 'px',
      height: height + 'px'
    }"
  >
    <canvas ref="chart" height="300" class="noselect"></canvas>
    <slot></slot>
  </div>
</template>

<script>
import F2 from "@antv/f2/lib/index-all";
import { getX, isEmpty } from "./utils/string";
import {
  renderPie,
  renderHistogram,
  renderLine,
  renderRadar,
  renderPoint
} from "./utils/render.js";
import { CHART_TYPE } from "./utils/constants.js";
const Util = F2.Util;
export default {
  props: {
    width: Number,
    height: {
      type: Number,
      default: 300
    },
    chartConfig: {
      type: Object
    },
    backgroundColor: {
      type: String,
      default: "#fff"
    },
    //度量 Scale，是数据空间到图形空间的转换桥梁，负责原始数据到 [0, 1] 区间数值的相互转换工作。针对不同的数据类型对应不同类型的度量。
    scale:{
      type:Object
    },

    customRender: {
      //是否自定义render，类型有:prevent，完全自定义,extra：在已有渲染的基础上自定义
      type: String,
      default: null,
      valiator: function(val) {
        return [null, "prevent", "extra"].indexOf(val) != -1;
      }
    },

    legend: {
      //是否显示图例
      type: Object,
      default: function() {
        return {
          disable: true
        };
      }
    },

    tooltip: {
      //是否显示提示信息
      type: Object,
      default: function() {
        return {
          disable: true //默认不显示提示信息
        };
      }
    },
    data: {
      //数据源
      type: Array,
      required: true
    },
          //提供的数据源，每一个字段的配置

    colDefs: {
      type: Object,
      default: function() {
        return {};
      }
    }
  },

  data() {
    return {
      chartType: "", //图表的类型
      datas: this.data,
      position: null
    };
  },
  watch: {
    data: {
      immediate: true,
      handler(val) {
        this.datas = val;
        if (this.chart != null) {
          this.changeData();
        }
      }
    }
  },
  computed: {
    chartData() {
      //图表的数据,如果Wie饼状图需要加上一个常量
      if (this.chartType == CHART_TYPE.pie) {
        return this.datas.slice().map(item => {
          item.data = "1";
          return item;
        });
      }
      return this.data;
    }
  },
  methods: {
    onTouchstart(e) {
      this.preventDefault && e.preventDefault();
    },
    changeData() {
      //修改数据
      this.chart.changeData(this.chartData);
    },
    repaint() {
      //重新绘制
      this.chart.repaint();
    },
    setChart(type, options) {
      let chart = this.chart;
      this.position = options.position;
      if (this.customRender === "prevent") {
        this.$emit("on-render", { chart });
        return;
      }
      //根据不同类型，绘制不同的图形
      this.chartType = type;

      let params = {
        chart: chart,
        colors: options.colors,
        position: options.position,
        colorFieldName: this.colorFieldName(options.colorField),
        options: options
      };

      //设置图形
      switch (type) {
        case CHART_TYPE.pie:
          renderPie(params);
          break;
        case CHART_TYPE.histogram:
          renderHistogram(params);
          break;

        case CHART_TYPE.line:
          renderLine(params);
          break;
        case CHART_TYPE.radar:
          renderRadar(params);
          break;
        case CHART_TYPE.point:
          renderPoint(params);
          break;
      }

      if (type != CHART_TYPE.pie) {
        if (options.axis.length > 0) {
          options.axis.forEach(element => {
            if (!isEmpty(element.fieldName)) {
              chart.axis(element.fieldName, element);
            }
          });
        }
      }
      this.setLegend();
      this.setTooltip();

      chart.source(this.chartData, this.colDefs);
      chart.scale(this.scale);
      if (this.customRender === "extra") {
        this.$emit("on-render", { chart });
      }
      chart.render();
    },
    setLegend() {
      //设置图例
      if (this.legend) {
        if (!isEmpty(this.legend.disable) && this.legend.disable) {
          this.chart.legend(false);
        } else {
          if (Util.isObject(this.legend)) {
            //legend默认使用X轴
            if (isEmpty(this.legend.fieldName)) {
              this.chart.legend(getX(this.position), this.legend);
            } else {
              this.chart.legend(this.legend.fieldName, this.legend);
            }
          }
        }
      }
    },

    setTooltip() {
      //设置图例
      if (this.tooltip) {
        if (!isEmpty(this.tooltip.disable) && this.tooltip.disable) {
          this.chart.tooltip(false);
        } else {
          if (Util.isObject(this.tooltip)) {
            this.chart.tooltip(this.tooltip);
          }
        }
      }
    },
    //渲染
    render() {
      const windowWidth = window.innerWidth;
      const windowHeight = window.innerHeight;

      let chart = new F2.Chart({
        el: this.$refs.chart,
        width: this.width || windowWidth,
        height: this.height ? this.height : 300,
        pixelRatio: this.$devicePixelRatio || window.devicePixelRatio,
        ...this.chartConfig
      });

      this.chart = chart;
    },
    reRender() {
      if (this.chart != null) {
        window.location.reload();
      }
    },
    colorFieldName(colorField) {
      //得到给哪个度量值设置颜色，默认为X轴
      if (isEmpty(colorField)) {
        return getX(this.position);
      }
      return colorField;
    }
  },
  mounted() {
    this.render();
    window.addEventListener("resize", this.reRender);
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.reRender);
  }
};
</script>

<style scoped>
.noselect {
  -webkit-touch-callout: none; /* iOS Safari */
  -webkit-user-select: none; /* Safari */
  -khtml-user-select: none; /* Konqueror HTML */
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* Internet Explorer/Edge */
  user-select: none; /* Non-prefixed version, currently
                                  supported by Chrome and Opera */
}
</style>
