<template>
  <div style="position: absolute width: 100%">
    <highchart
      :options="chartOptions"
      :update="watchers"
    />
  </div>
</template>

<script>
export default {
  props: {
    series: {
      type: Array,
      required: true
    },
    xlabel: {
      type: String,
      required: true
    },
    ylabel: {
      type: String,
      required: true
    }
  },

  data () {
    return {
      watchers: ['options.series', 'options.yAxis', 'options.xAxis']
    }
  },

  computed: {
    chartOptions () {
      return {
        chart: {
          type: 'line',
          height: (3 / 4 * 100) + '%',
          redraw: true,
          animation: true,
          zoomType: 'xy',
          panning: true,
          panKey: 'shift'
        },
        boost: { enabled: true },
        tooltip: {
        },
        series: this.series,
        yAxis: {
          title: {
            text: this.ylabel
          }
        },
        xAxis: {
          title: {
            text: this.xlabel
          }
        },
        title: {
          text: ''
        },
        credits: {
          enabled: false
        }
      }
    }
  }
}
</script>
