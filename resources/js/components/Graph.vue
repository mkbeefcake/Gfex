<template>
  <div>
    <loading :active.sync="isLoading" 
             :is-full-page="fullPage"></loading>
    <div style="display:flex;margin-bottom:6px;text-align:center;min-height:600px" class="row">
      <div class="col-3">
        <label class="typo__label gf-item">Axis X:</label>
        <multiselect v-model="axisX" :options="options" track-by="name" label="name" :taggable="true" placeholder="Select X axis."></multiselect>
        <label class="typo__label gf-item">Axes Y:</label>
        <multiselect v-model="axisY" :multiple="true" :options="options" :close-on-select="false" :maxHeight="250" track-by="name" label="name" :taggable="true" placeholder="Select Y axis."></multiselect>
        <label class="typo__label gf-item">Axis Y2:</label>
        <multiselect v-model="axisY2" :options="options" track-by="name" label="name" :taggable="true" placeholder="Select Y2 axis."></multiselect>
        <label class="typo__label gf-item">Step:</label>
        <multiselect v-model="selectedStep" :options="steps" track-by="name" label="name" :taggable="true" placeholder="Select step for graph."></multiselect>

        <div style="margin-top:32px;display:flex;text-align:left">
            <input type="color" style="min-width:50px;height:50px;margin-right:20px;" id="axisColor" name="axisColor" v-model="axisColor" @change="onApplyColor($event)">
            <label for="axisColor" class="typo__label gf-item">Axis Color</label>
        </div>

        <div style="margin-top:32px;max-height:200px;overflow-y:scroll;overflow-x:hidden">
            <div style="display:flex;text-align:left;margin-bottom:4px" v-for="entry in axisY" :key='entry.index'>
              <input type="color" style="min-width:50px;height:50px;margin-right:20px;" :id="'graphColor' + entry.index" 
                 :name="'graphColor' + entry.index" v-model="graphColor[entry.index]" @change="onApplyColor($event)">
              <label :for="'graphColor' + entry.index" class="typo__label gf-item">{{entry.name}}</label>
            </div>
        </div>

        <div style="margin-top:32px;display:flex;text-align:left" v-show="axisY2 != null && axisY2 != undefined">
            <input type="color" style="min-width:50px;height:50px;margin-right:20px;" id="graph1Color" name="graph1Color" v-model="graph1Color" @change="onApplyColor($event)">
            <label for="graph1Color" class="typo__label gf-item">{{axisY2 ? axisY2.name : 'Axis Y2 color'}}</label>
        </div>

      </div>
      <div class="col-2">
        <label class="btn btn-primary gf-button" style="margin-top:48px" v-on:click="onShow">Graph</label>
        <label class="btn btn-primary gf-button" style="margin-top:32px" v-on:click="onPrintGraph">Print Graph</label>
      </div>
      <div id="plot3" class="col-7" ref="plot3">
      </div>
    </div>

    <div id="plotModal3" class="gf-modal">
      <div class="gf-modal-content">
        <div class="gf-modal-header">
          <span class="gf-comment" style="margin-left:30px">FastPlan* Gas & Gas Condensate</span>
          <span class="gf-close" id="plot-gf-close3" v-on:click="onCancel" >&times;</span>
        </div>
        <p class="gf-comment" style="margin-top:6px !important; margin-bottom:6px !important;"><{{projectName}}> Field Project</p>
        <span style="font-size: 1.25rem">Please select all X, Y axes</span>
        <div style="margin-bottom:16px;margin-top:16px">
          <label class="btn btn-primary gf-button" v-on:click="onOK">OK</label>
        </div>
      </div>
    </div>

  </div>
</template>
<script>
import store from '~/store'
import { mapState } from 'vuex'
import Multiselect from 'vue-multiselect'
import Loading from 'vue-loading-overlay';
import html2canvas from 'html2canvas';
import 'vue-loading-overlay/dist/vue-loading.css';

export default {
  name: 'Graph',
  middleware: ['auth', 'theme'],

  props: ['options', 'data', 'labels', 'type'],
  
  components: {
    Multiselect,
    Loading
  },

  data() {
    return {
      axisX: null,
      axisY: null,
      axisY2: null,
      plot: null,
      axisColor: '#ffffff',
      graphColor: [],
      graph1Color: '#b9ff78',
      isLoading: false,
      fullPage: true,
      maxItems : 200,
      selectedStep: {name: 'Step: 1', value: 1},
      steps: [
        {name: 'Step: 1', value: 1}
      ]
    }
  },

  watch: {
    options: function(val, oldVal) {
      this.axisX = null
      this.axisY = null
      this.axisY2 = null
    },
    axisY: function (val, oldVal) {
      if ((val.length == 1 && oldVal == null) || (val.length > oldVal.length)) {
        let randomColor = Math.floor(Math.random()*16777215).toString(16);
        let last = val[val.length - 1]
        this.graphColor[last.index] = "#" + randomColor
      }
    },
    data: function(val, oldVal) {
      try {
        let maxRows = val.length    
        let maxStep = Math.floor(maxRows / 200)

        if (maxStep <= 0)
          maxStep = 1
        
        this.steps = []
        for (let i = 1; i <= maxStep; i ++) {
          this.steps.push({name: 'Step: ' + i, value: i})
        }
        this.selectedStep = this.steps[maxStep-1]
      }
      catch (e) {
        this.steps = []
        this.steps.push({name: 'Step: 1', value: 1})
        this.selectedStep = this.steps[0]
      }
    }
  },

  computed: {
    ...mapState({
      projectName : state => state.project.projectName,
    }),
  },

  methods: {
    onApplyColor: function(event) {
      // document.documentElement.style.setProperty('--axis-color', this.axisColor);
      // document.documentElement.style.setProperty('--graph-color', this.graphColor);
      // document.documentElement.style.setProperty('--graph1-color', this.graph1Color);
      this.onShow(null);
    },
    onShow: async function(event) {

      // ----------------------------------------------------------
      // Validation
      // ----------------------------------------------------------
      if (this.axisX == null || (this.axisY == null && this.axisY2 == null)) {
        var modal = document.getElementById("plotModal3");
        modal.style.display = "block";
        return;
      }

      document.documentElement.style.setProperty('--axis-color', this.axisColor);

      // ----------------------------------------------------------
      // Initialize variables
      // ----------------------------------------------------------
      // document.documentElement.style.setProperty('--axis-color', this.axisColor);
      // document.documentElement.style.setProperty('--secondary-color', this.graphColor);

      var axisX = this.axisX.name
      var columns = []
      var axes = {}
      var ylabel = null
      var ylabel2 = null
      var y2Max = null
      var y2Classes = {}

      // ----------------------------------------------------------
      // Start HERE
      // ----------------------------------------------------------
      let numRows = this.data.length;
      let steps = this.selectedStep.value

      // ----------------------------------------------------------------
      // add x data
      columns[0] = []
      columns[0][0] = this.axisX.name
      for (let index = 1; index <= Math.floor(numRows/steps); index++) {
        columns[0][index] = this.data[(index-1) * steps][this.axisX.index]
      }

      // ----------------------------------------------------------------
      // add y data : multiple axis Y
      // ----------------------------------------------------------------
      var maxY = 0
      for (let i = 0; i < this.axisY.length; i++) {

        columns[i+1] = []
        columns[i+1][0] = this.axisY[i].name
        for (let j = 1; j <= Math.floor(numRows/steps); j++) {
          columns[i+1][j] = this.data[(j-1) * steps][this.axisY[i].index]

          if (maxY < columns[i+1][j]) {
            ylabel = this.axisY[i].name
            maxY = columns[i+1][j]
          }
        }
      }

      // ----------------------------------------------------------------
      // add y2 data : axis Y2
      // ----------------------------------------------------------------
      if (this.axisY2 != null) {
        columns[this.axisY.length+1] = []
        columns[this.axisY.length+1][0] = this.axisY2.name

        y2Max = 0
        for (let j = 1; j <= Math.floor(numRows/steps); j++) {
          columns[this.axisY.length+1][j] = this.data[(j-1) * steps][this.axisY2.index]
          if (y2Max < parseFloat(this.data[(j-1) * steps][this.axisY2.index]))
            y2Max = parseFloat(this.data[(j-1) * steps][this.axisY2.index])
        }

        axes[this.axisY2.name] = 'y2'
        ylabel2 = this.axisY2.name
        y2Classes[this.axisY2.name] = 'dotted'

        y2Max = y2Max * 1.2
      }

      //
      // Removed only one Y, Y2 axis
      //

      // // ----------------------------------------------------------------
      // // add y data
      // columns[1] = []
      // columns[1][0] = this.axisX.name + " vs " + this.axisY.name
      // for (let index = 1; index <= numRows; index++) {
      //   columns[1][index] = this.data[index-1][this.axisY.index]
      // }
      // ylabel = this.axisY.name

      // // ----------------------------------------------------------------
      // // add y2 data : removed axis Y2
      // // ----------------------------------------------------------------
      // if (this.axisY2 != null) {
      //   columns[2] = []
      //   columns[2][0] = this.axisX.name + " vs " + this.axisY2.name
      //   for (let j = 1; j <= numRows; j++) {
      //     columns[2][j] = this.data[j-1][this.axisY2.index]
      //   }

      //   axes[this.axisX.name + " vs " + this.axisY.name] = 'y'
      //   axes[this.axisX.name + " vs " + this.axisY2.name] = 'y2'
      //   ylabel2 = this.axisY2.name
      // }

      let plotColor = []
      for (let k = 0; k < this.axisY.length; k++) {
        plotColor.push(this.graphColor[this.axisY[k].index])
      }
      plotColor.push(this.graph1Color)

      this.updatePlot(axisX, columns, axes, ylabel, ylabel2, plotColor, y2Max, y2Classes);
    },
    updatePlot: async function(_axisX, _columns, _axes, _ylabel, _ylabel2, plotColor, y2Max, y2Classes) {
      let plotOptions = {
          bindto: '#plot3',
          size: {
              height: 800,
          },
          data: {
            x: _axisX,
            columns: _columns,
            axes: _axes,
            type: this.type,
            classes: y2Classes
          },
          color: {
            pattern: plotColor
          },
          legend: {
            show: false,
            position: 'inset',
            inset: {
              anchor: 'top-left',
              x: 20,
              y: 40,
            },
          },
          axis: {
            x: {
              height: 55,
              label: {
                text: _axisX,
                position: 'outer-center'
              }
            },
            y: {
              label: {
                text: _ylabel,
                position: 'outer-middle'
              }
            },
            y2: {
              max: y2Max,
              show: _ylabel2 != null, // ADD
              label: {
                text: _ylabel2,
                position: 'outer-middle'
              }
            }
          },
          oninit: function () {
            this.isLoading = true
            console.log('chart is initialized')
          },
          onrendered: function() {
            this.isLoading = false
            console.log('chart is rendered')
          }
      }

      this.plot = c3.generate(plotOptions)
    },
    onPrintGraph: async function(event) {
      console.log("printing..");
      this.isLoading = true
      // svg.saveSvgAsPng(document.getElementById('plot').firstChild, 'diagram.png');

      const el = document.getElementById('plot3');
      const options = { type: "dataURL" };

      const printCanvas = await html2canvas(el, options);

      const link = document.createElement("a");
      link.setAttribute("download", "download.png");
      link.setAttribute(
        "href",
        printCanvas
          .toDataURL("image/png")
          .replace("image/png", "image/octet-stream")
      );
      link.click();

      this.isLoading = false
      console.log("done");
    },
    onOK: function(event) {
        var modal = document.getElementById("plotModal3");
        modal.style.display = "none";
    },
    onCancel: function(event) {
        var modal = document.getElementById("plotModal3");
        modal.style.display = "none";
    },
  },

  mounted() {
    mountPlotDialog();
  }

}

function mountPlotDialog() {

  // Get the modal
  var modal = document.getElementById("plotModal3");

  // Get the <span> element that closes the modal
  var span = document.getElementById("plot-gf-close3");

  // When the user clicks on <span> (x), close the modal
  span.onclick = function() {
    modal.style.display = "none";
  }

  // When the user clicks anywhere outside of the modal, close it
  window.onclick = function(event) {
    if (event.target == modal) {
      modal.style.display = "none";
    }
  }      
}

</script>

<style scoped>
#plot3 {
  background: var(--secondary-color);
}
</style>
<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>
