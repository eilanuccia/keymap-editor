<script>
import filter from 'lodash/filter'
import get from 'lodash/get'
import isEmpty from 'lodash/isEmpty'
import keyBy from 'lodash/keyBy'
import times from 'lodash/times'

import { getKeyBoundingBox } from '../key-units'

import KeyboardLayout from './keyboard-layout.vue'
import LayerSelector from './layer-selector.vue'

export default {
  name: 'keymap',
  components: {
    'layer-selector': LayerSelector,
    'keyboard-layout': KeyboardLayout
  },
  props: ['layout', 'keymap'],
  emits: ['update'],
  inject: [
    'keycodes',
    'behaviours',
    'indexedKeycodes',
    'indexedBehaviours'
  ],
  provide() {
    return {
      getSearchTargets: this.getSearchTargets,
      getSources: () => this.sources
    }
  },
  data() {
    return {
      activeLayer: 0,
      editing: null
    }
  },
  computed: {
    availableLayers() {
      if (isEmpty(this.keymap)) {
        return []
      }

      return this.keymap.layers.map((_, i) => ({
        code: i,
        description: this.keymap.layer_names[i] || `Layer ${i}`
      }))
    },
    sources() {
      return {
        kc: this.indexedKeycodes,
        code: this.indexedKeycodes,
        mod: keyBy(filter(this.keycodes, 'isModifier'), 'code'),
        behaviours: this.indexedBehaviours,
        layer: keyBy(this.availableLayers, 'code')
      }
    }
  },
  methods: {
    isReady() {
      return (
        Object.keys(this.indexedKeycodes).length > 0 &&
        Object.keys(this.indexedBehaviours).length > 0 &&
        get(this.keymap, 'layers.length', 0) > 0
      )
    },
    getSearchTargets(param, behaviour) {
      // Special case for behaviour commands which can dynamically add another
      // parameter that isn't defined at the root level of the behaviour.
      // Currently this is just `&bt BT_SEL` and is only represented as an enum.
      if (param.enum) {
        return param.enum.map(v => ({ code: v }))
      }

      switch (param) {
        case 'behaviour':
          return this.behaviours
        case 'layer':
          return this.availableLayers
        case 'mod':
          return filter(this.keycodes, 'isModifier')
        case 'command':
          return get(this.sources, ['behaviours', behaviour, 'commands'], [])
        case 'kc':
        default:
          return this.keycodes
      }
    },
    boundingBox() {
      return this.layout.map(key => getKeyBoundingBox(
        { x: key.x, y: key.y },
        { u: key.u || key.w || 1, h: key.h || 1 },
        { x: key.rx, y: key.ry, a: key.r }
      )).reduce(({ x, y }, { max }) => ({
        x: Math.max(x, max.x),
        y: Math.max(y, max.y)
      }), { x: 0, y: 0 })
    },
    getWrapperStyle() {
      const bbox = this.boundingBox()
      return {
        width: `${bbox.x}px`,
        height: `${bbox.y}px`,
        margin: '0 auto',
        padding: '40px'
      }
    },
    handleCreateLayer() {
      const layer = this.keymap.layers.length
      const binding = '&trans'
      const makeKeycode = () => ({ value: binding, params: [] })

      const newLayer = times(this.layout.length, makeKeycode)
      const updatedLayerNames = [ ...this.keymap.layer_names, `Layer #${layer}` ]
      const layers = [ ...this.keymap.layers, newLayer ]

      this.$emit('update', { ...this.keymap, layer_names: updatedLayerNames, layers })
    },
    handleUpdateLayer(layerIndex, updatedLayer) {
      const original = this.keymap.layers
      const layers = [
        ...original.slice(0, layerIndex),
        updatedLayer,
        ...original.slice(layerIndex + 1)
      ]

      this.$emit('update', { ...this.keymap, layers })
    },
    handleDeleteLayer(layerIndex) {
      const layer_names = [...this.keymap.layer_names];
      layer_names.splice(layerIndex, 1);

      const layers = [...this.keymap.layers];
      layers.splice(layerIndex, 1);

      this.$emit("update", { ...this.keymap, layers, layer_names });
    },
  }
}
</script>

<template>
  <layer-selector
    v-if="isReady()"
    :layers="keymap.layer_names"
    :activeLayer="activeLayer"
    @select="activeLayer = $event"
    @new-layer="handleCreateLayer"
    @delete-layer="handleDeleteLayer($event)"
  />
  <div :style="getWrapperStyle()" v-bind="$attrs">
    <keyboard-layout
      v-if="isReady()"
      :data-layer="activeLayer"
      :layout="layout"
      :keys="keymap.layers[activeLayer]"
      @update="handleUpdateLayer(activeLayer, $event)"
      class="active"
    />
  </div>
</template>