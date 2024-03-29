<template>
  <div class="row">
    <div class="m-auto">
    <form>
      <loading :active.sync="isLoading" 
             :is-full-page="fullPage"></loading>
      <div class="card mb-3">
        <div class="card-header gf-header">
          <img src="/assets/image/LOGO_GFEX.png" style="max-width:150px;max-height:180px;margin-left:-7px;float:left">
          FastPlan* Gas & Gas Condensate<br>
          <p style="font-size:3rem !important">Conventional and Shale Reservoirs</p>
        </div>
        <div class="row g-0" style="background-color:var(--background-color);">
          <div class="col-md-10 offset-md-1">
            <div class="card-body">
              <h3 class="card-title gf-title"><{{projectName}}> Field Project
              </h3>
              <p class="card-text" style="font-size: 2.4rem !important;text-align: center !important;">SYSTEM MONITORING / WORK-OVER / DRILLING RESULTS</p>

              <div style="display:flex;margin-bottom:6px;text-align:left" class="row">
                <div class="col-3">
                  <p class="gf-item">Output Files</p>
                  <multiselect v-model="outputFile" @input="onChange" :options="outputFileOptions" track-by="name" label="name" placeholder="Select OUTPUT files"></multiselect>
                </div>
                <div class="col-4" style="display:flex">
                  <label class="btn btn-primary gf-button" style="margin:auto auto 0 auto" v-on:click="onPlot">{{plotLabel}}</label>
                  <label class="btn btn-primary gf-button" style="margin:auto auto 0 auto" v-on:click="onPlot2">WELL GRAPH</label>
                </div>
              </div>

              <hr class="gf-line">

              <div style="display:flex;margin-bottom:6px;text-align:left" class="row" v-show="bShowPlot == true">
                <graph v-bind:options="options"
                       v-bind:type="graphType"
                      v-bind:data="graphData"></graph>
              </div>

              <div style="display:flex;margin-bottom:6px;text-align:left" class="row" v-show="bShowPlot == false">
                <textarea v-model.lazy="dataContent" style="margin-left:10px;font-weight: 400;"></textarea>
              </div>


              <div class="d-flex justify-content-between">
                <label class="btn btn-primary gf-button" v-on:click="onPrevPage">Previous</label>
                <div>
                  <label class="btn btn-primary gf-button " v-on:click="onExitPage">Exit</label>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </form>

    </div>

    <div id="exitModal" class="gf-modal">
      <div class="gf-modal-content">
        <div class="gf-modal-header">
          <span class="gf-comment" style="margin-left:30px">FastPlan* Gas & Gas Condensate</span>
          <span class="gf-close" id="gf-exit-cancel">&times;</span>
        </div>
        <p class="gf-comment" style="margin-top:6px !important; margin-bottom:6px !important;"><{{projectName}}> Field Project</p>
        <span style="font-size: 1.25rem" v-show="isSaveAs==false">Do you want to save this project?</span>
        <div v-show="isSaveAs==true">
          <span style="font-size: 1.25rem">Project Name: </span>
          <input style="font-size: 1.25rem" maxlength="20" v-model="newProjectName" placeholder="Please input new project name">
        </div>
        <div style="margin-bottom:16px;margin-top:16px">
          <label class="btn btn-primary gf-button" v-on:click="onSaveAs" v-show="hideSaveAsButton==false">Save As</label>
          <label class="btn btn-primary gf-button" v-on:click="onYes">Yes</label>
          <label class="btn btn-primary gf-button" v-on:click="onNo">No</label>
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
import 'vue-loading-overlay/dist/vue-loading.css';
import html2canvas from 'html2canvas';
import Graph from '~/components/Graph.vue';


// import axios from 'axios'
export default {
  name: 'FastplanResult',
  middleware: ['auth', 'theme'],

  metaInfo () {
    return { title: this.$t('Fastplan Result') }
  },

  components: {
    Multiselect,
    Loading,
    Graph
  },

  data() {
    return {
      isLoading: false,
      fullPage: true,
      isSaveAs : false,
      hideSaveAsButton: false,
      newProjectName: "",
      plotLabel: "Data",
      bShowPlot : true,
      outputFile: null,
      outputFileOptions: [
        // { name: 'WELL'},
      ],
      dataContent: '',
      options: [],
      graphData: [],
      previousPage: '',
      graphType: null,
      resMonitoring: null
    }
  },

  computed: {
    ...mapState({
      projectName : state => state.project.projectName,
      projectId: state => state.project.projectId,
      isFDP: state => state.project.isFDP,
      isCondensate: state => state.project.isCondensate,
      isEconomics: state => state.project.isEconomics,
      isSeparatorOptimizer: state => state.project.isSeparatorOptimizer,
      sep : state => state.project.sep,
      drygas : state => state.project.drygas,
      surface: state => state.project.surface,
      reservoir: state => state.project.reservoir,
      wellhistory: state => state.project.wellhistory,
      economics: state => state.project.economics,
      operations: state => state.project.operations,
      relPerm: state => state.project.relPerm,
      gascondensate : state => state.project.gascondensate,
    }),
  },

  methods: {
    onChange: function(event) {
      if (this.outputFile != null) {
        if (this.outputFile.name == 'PRESSURE_MATCHING') {
          this.options = [
            { name: "Qg (mmscf/day)", index: 0}, 
            { name: "Pr (psia)", index: 1}, 
          ]
          this.graphType = 'bar'
        }
        else {
          this.options = [
            { name: "Time (Months)", index: 0}, 
            { name: "Pr (psia)", index: 1}, 
            { name: "PWF (psia)", index: 2}, 
            { name: "PWH (psia)", index: 3}, 
            { name: "PMAN (psia)", index: 4}, 
            { name: "P_in (psia)", index: 5}, 
            { name: "P_out (psia)", index: 6}, 
            { name: "P_SALES (psia)", index: 7}, 
            { name: "QG (mmscf/day)", index: 8}, 
            { name: "CUM.GAS (This Well) (BCF)", index: 9}, 
            { name: "QG (All Wells) (mmscf/day)", index: 10}, 
            { name: "TOTAL CUM.GAS (All Wells) (BCF)", index: 11}, 
            { name: "P/Z", index: 12}, 
            { name: "MATRIX-FRACTURE TRANSFER (MSCF/D)", index: 13}, 
            { name: "MATRIX CONTRIBUTION (BCF)", index: 14}, 
          ]
          this.graphType = null
        }
        this.dataContent = this.resMonitoring[this.outputFile.name]
        this.graphData = this.resMonitoring['RES_' + this.outputFile.name]
      }
      else {
        this.options = []
        this.graphData = []
        this.dataContent = ''
        this.graphType = null
      }
    },
    onPlot2: function(event) {
      debugger
      this.$router.replace({ name: 'monitoringresult2', params: { resMonitoring: this.resMonitoring } })
    },
    onPlot: function(event) {
      if(this.bShowPlot == false) {
        this.bShowPlot = true
        this.plotLabel = "Data"
      }
      else {
        this.bShowPlot = false
        this.plotLabel = "Plot"
      }
    },
    onSaveAs: function(event) {
      this.isSaveAs = true
      this.hideSaveAsButton = true
    },
    onYes: function(event) {
      // hide exit dialog
      var modal = document.getElementById("exitModal");
      modal.style.display = "none";

      this.isLoading = true
      this.onSaveProject()      
      this.isLoading = false

      // go to home vue
      this.$router.replace('home')
    },
    onNo: function(event) {
      // hide exit dialog
      var modal = document.getElementById("exitModal");
      modal.style.display = "none";

      // go to home vue
      this.$router.replace('home')
    },
    onExitPage: function (event) {
      this.isSaveAs = false
      this.hideSaveAsButton = false

      var modal = document.getElementById("exitModal");
      modal.style.display = "block";
    },
    onPrevPage: function(event) {
      this.$router.replace('drygas')
    },
  },
  mounted() {
    debugger
    this.resMonitoring = this.$route.params.resMonitoring
    this.outputFileOptions = []

    let wellCount = this.resMonitoring.NumberOfWells
    for (let index = 0; index < wellCount; index++) {
      this.outputFileOptions.push({name: 'WELL' + (index + 1)})
    }

    this.outputFileOptions.push({name: 'PRESSURE_MATCHING'})
    mountExitDialog();
  }

}

function mountExitDialog() {

  // Get the modal
  var modal = document.getElementById("exitModal");

  // Get the <span> element that closes the modal
  var span = document.getElementById("gf-exit-cancel");

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
.jexcel > thead > tr > td {
  font-size: 20px;
}
</style>

<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>
<style scoped>
.multiselect {
  max-width: 370px;
  /* margin: auto; */
}
textarea{
    width: 90%;
    height: 450px;
    white-space: nowrap;
    overflow-x: auto;
    resize: none;
    background-color: var(--secondary-color);
    color: var(--text-color);
    font-family: monospace !important;
    font-size: 1.25rem !important;
}
</style>
