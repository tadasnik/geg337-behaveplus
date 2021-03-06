<template>
  <div>
    <section class="section">
      <div class="tile is-ancestor">
        <div class="tile is-parent">
          <div class="tile is-child box">
            <h4 class="title is-4">
              Fuel models
            </h4>

            <div class="columns is-mobile is-multiline is-centered">
              <fuel-tile
                v-for="(fuel, index) in selectedFuels"
                :key="fuel + index"
                :fuel-model-code="fuel"
                :node-props="nodeProps.fuelNodeProps"
                @change="fuelModelRun(fuel)"
              />
            </div>
          </div>
        </div>
      </div>
      <div class="tile is-ancestor">
        <div class="tile is-5 is-parent">
          <div class="tile is-child box">
            <model-inputs
              key="uidinputs"
              :selected-inputs="selectedInputs"
              :node-props="nodeProps.inputNodes"
              @change="runModels()"
            />
          </div>
        </div>
        <div class="tile is-parent">
          <div class="tile is-child box">
            <sensitivity
              @change="runOutputs()"
            />
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { Sim, StorageNodeMap } from '@cbevins/fire-behavior-simulator'
import { min, max, median, quantile } from 'd3-array'
import { mapGetters } from 'vuex'
import { defaultConfig } from '@/assets/defaultConfig.js'
import { nodeProps } from '@/assets/nodeProps.js'
import { exFuels } from '@/assets/fuels.js'
import FuelTile from '~/components/FuelTile'
import ModelInputs from '~/components/ModelInputs'

export default {
  name: 'BehavePlus',

  components: {
    FuelTile,
    ModelInputs
  },

  data () {
    return {
      sim: {},
      dag: {},
      nodeProps: {},
      resultsObj: {},
      selectedInputs: [],
      selectedCanopyInputs: [],
      activeTab: 0,
      fuelPropNames: ['depth', 'dead1Load', 'dead10Load', 'dead100Load', 'totalHerbLoad', 'liveStemLoad']
    }
  },

  computed: {
    ...mapGetters({
      fuelModels: 'selector/fuelModels',
      siteInputs: 'selector/siteInputs',
      // canopyInputs: 'selector/canopyInputs',
      outputNodes: 'selector/outputNodes',
      defaultDagConfig: 'selector/defaultDagConfig',
      results: 'selector/results',
      rangeInput: 'selector/rangeInput',
      selectedOutputs: 'selector/selectedOutputs',
      selectedFuels: 'selector/selectedFuels'
    }),

    heatDataset () {
      const resArray = []
      this.selectedFuels.forEach((fuel, index) => {
        this.fuelPropNames.forEach((prop, pIndex) => {
          resArray.push([index, pIndex, this.fuelModels[fuel][prop]])
        })
      })
      return resArray
    },

    dataset () {
      const resArray = []
      this.selectedOutputs.forEach((node) => {
        const res = []
        this.selectedFuels.forEach((fuel) => {
          res.push({ x: fuel, y: this.getStats(this.results[node][fuel]) })
        })
        resArray.push({
          label: node,
          dataset: [{ data: res }]
        })
      })
      return resArray
    },

    fuelStrings () {
      const optionsText = []
      Object.keys(this.fuelModels).forEach((key) => {
        optionsText.push([key, this.fuelModels[key].code +
                         ': ' +
                         this.fuelModels[key].label])
      })
      return optionsText
    },

    availFuelModelStrings () {
      const modelStrings = []
      this.fuelModels.forEach((element) => {
        modelStrings.push(element.label)
      })
      return modelStrings
    },

    selectedNodes () {
      const outputNodeList = []
      this.selectedOutputs.forEach((item) => {
        outputNodeList.push(this.outputNodes[item].geneLabel)
      })
      return outputNodeList
    }
  },

  created () {
    this.nodeProps = nodeProps
    // this.fuelCatalog = FuelCatalog
    this.$store.dispatch('selector/initOutputNodes', this.nodeProps.outputNodes)
    this.$store.dispatch('selector/initSiteInputs', this.nodeProps.inputNodes)
    // this.$store.dispatch('selector/initCanopyInputs', this.nodeProps.canopyNodes)
    this.sim = new Sim()
    // this.crown = CrownFire
    this.dag = this.sim.createDag('basicUsage')
    this.dag.configure(this.defaultDagConfig)
    this.setNodeUnits(this.nodeProps)
    this.prepareFuels()
    // this.$store.dispatch('selector/initFuelModels', exFuels)
    this.$store.dispatch('selector/initConfig')
    this.resultsObj = new StorageNodeMap(this.dag)
    this.dag.setStorageClass(this.resultsObj)
    this.updateDagSelected()
    this.updateRequiredSiteInputs()
    // console.log('created', this.canopyInputs)
    // this.updateRequiredCanopyInputs()
  },

  mounted () {
    this.runModels()
  },

  beforeUpdate () {
    window.dispatchEvent(new Event('resize'))
  },

  methods: {
    getBoxplotLabel (node) {
      return this.outputNodes[node].text + ' (' +
        this.outputNodes[node].units + ')'
    },

    getStats (values) {
      if (values.length === 1) {
        return Array(5).fill(values[0])
      } else {
        const vMin = min(values)
        const vQ25 = quantile(values, 0.25)
        const vMax = max(values)
        const vMedian = median(values)
        const vQ75 = quantile(values, 0.75)
        return [vMin, vQ25, vMedian, vQ75, vMax]
      }
    },

    getSeries (results) {
      const res = []
      for (const [key, val] of Object.entries(results)) {
        if (val.length === 1) {
          res.push({ x: key, y: Array(5).fill(val[0]) })
        } else {
          const stats = this.getStats(val)
          res.push({ x: key, y: stats })
        }
      }
      return [{ data: res }]
    },

    runModel (fuel) {
      this.setFuelInputs(fuel)
      this.setInputs(this.siteInputs)
      // this.setInputs(this.canopyInputs)
      this.dag.run()
      this.selectedOutputs.forEach((item) => {
        const geneLabel = this.outputNodes[item].geneLabel
        const node = this.dag.get(geneLabel)
        const res = this.resultsObj.get(geneLabel)
        const dispRes = []
        res.forEach((item) => {
          dispRes.push(node._variant.displayValue(item))
        })
        this.$store.dispatch('selector/updateResults', { output: item, fuel, payload: dispRes.map(Number) })
      })
    },

    fuelModelRun (fuel) {
      this.runModel(fuel)
    },

    updateDagSelected () {
      this.dag.clearSelected()
      this.dag.select(this.selectedNodes)
    },

    updateRequiredCanopyInputs () {
      this.selectedCanopyInputs = []
      const reqNodes = []
      this.dag.requiredInputNodes().forEach((node) => {
        const splitKey = node.key().split('.')
        if (splitKey[0] === 'site' && splitKey[1] === 'canopy') {
          reqNodes.push(node.key())
        }
      })
      Object.keys(this.canopyInputs).forEach((key) => {
        if (reqNodes.includes(this.siteCanopyInputs[key].geneLabel)) {
          this.selectedCanopyInputs.push(key)
        }
      })
    },

    updateRequiredSiteInputs () {
      this.selectedInputs = []
      const reqNodes = []
      this.dag.requiredInputNodes().forEach((node) => {
        if (node.key().split('.')[0] === 'site') {
          reqNodes.push(node.key())
        }
      })
      Object.keys(this.siteInputs).forEach((key) => {
        if (reqNodes.includes(this.siteInputs[key].geneLabel)) {
          this.selectedInputs.push(key)
        }
      })
    },

    runModels () {
      this.selectedFuels.forEach((item) => {
        this.runModel(item)
      })
    },

    setNodeUnits (nodePropObject) {
      Object.values(nodePropObject).forEach((group) => {
        Object.values(group).forEach((item) => {
          const node = this.dag.get(item.geneLabel)
          node._variant.setDisplayUnits(item.units)
          node._variant.setDisplayDecimals(item.decimals)
        })
      })
    },

    prepareFuels () {
      // const Ms = FuelCatalog.models().filter(item => item.domain === 'behave')
      // const fuelMs = {}
      // exFuels.forEach((item) => {
      Object.keys(exFuels).forEach((key) => {
        Object.values(this.nodeProps.fuelNodeProps).forEach((fuelProp) => {
          const node = this.dag.get(fuelProp.geneLabel)
          const val = node._variant.displayValue(exFuels[key][fuelProp.catalogParam])
          exFuels[key][fuelProp.catalogParam] = parseFloat(val)
        })
      })
      const defFuels = defaultConfig.defaultFuels
      defFuels.forEach((item) => {
        exFuels[item].selected = true
      })
      this.$store.dispatch('selector/initFuelModels', exFuels)
    },

    arrayToNative (node, values) {
      const valuesNative = []
      values.forEach((element) => {
        valuesNative.push(node._variant.displayValueToNativeValue(element))
      })
      return valuesNative
    },

    setInputs (inputs) {
      Object.values(inputs).forEach((input) => {
        const node = this.dag.get(input.geneLabel)
        const val = input.value
        if (Array.isArray(val)) {
          const valuesNative = this.arrayToNative(node, val)
          this.dag.input([[node, valuesNative]])
        } else {
          this.dag.input([[node, node._variant.displayValueToNativeValue(val)]])
        }
      })
    },

    setFuelInputs (fuel) {
      this.dag.input([['surface.primary.fuel.model.catalogKey', fuel]])
      Object.values(this.nodeProps.fuelNodeProps).forEach((fuelProp) => {
        if (fuelProp.selected) {
          const node = this.dag.get(fuelProp.geneLabel)
          node._is._input = true
          const val = this.fuelModels[fuel][fuelProp.catalogParam]
          if (Array.isArray(val)) {
            const valuesNative = this.arrayToNative(node, val)
            this.dag.input([[node, valuesNative]])
          } else {
            this.dag.input([[node, node._variant.displayValueToNativeValue(val)]])
          }
        }
      })
    },

    modelSetup () {
      this.updateDagSelected()
      this.updateRequiredSiteInputs()
      // this.updateRequiredCanopyInputs()
    },

    runOutputs () {
      this.modelSetup()
      this.runModels()
    },

    updateSelectedFuel (label, payload) {
      if (payload !== true) {
        const index = this.selectedFuels.indexOf(label)
        if (index > -1) {
          this.selectedFuels.splice(index, 1)
        }
      } else {
        this.selectedFuels.push(label)
        this.runModel(label)
      }
    }
  }
}
</script>
