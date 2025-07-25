<template>
  <div id="visual-keymap" :style="styles">
    <template v-for="meta in currentLayer">
      <transition :key="meta.id" name="fade" appear>
        <component :is="getComponent(meta)" v-bind="meta" :key="meta.id" />
      </transition>
    </template>
  </div>
</template>
<script>
import isUndefined from 'lodash/isUndefined';
import { mapState, mapGetters, mapMutations } from 'vuex';
import * as pinia from 'pinia';
import BaseKeymap from '@/components/BaseKeymap.vue';
import BaseKey from '@/components/BaseKey.vue';
import AnyKey from '@/components/AnyKey.vue';
import LayerKey from '@/components/LayerKey.vue';
import ContainerKey from '@/components/ContainerKey.vue';
import LayerContainerKey from '@/components/LayerContainerKey.vue';

import { useStatusStore } from '@/store/status';

export default {
  name: 'VisualKeymap',
  components: { BaseKey, AnyKey, LayerKey, ContainerKey },
  extends: BaseKeymap,
  props: {
    profile: Boolean,
    debug: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      width: 0,
      height: 0
    };
  },
  computed: {
    styles() {
      let keySize = this.fontSize;
      let smolKeySize = this.fontSize * 0.7;
      if (this.config.SCALE < 1) {
        keySize *= (1 + this.config.SCALE) / 2;
        smolKeySize *= (1 + this.config.SCALE) / 2;
      }
      return {
        ...{
          '--unit-width': '1',
          '--unit-height': '1',
          '--default-smaller-key-font-size': `${smolKeySize}mm`,
          '--default-key-font-size': `${keySize}mm`,
          '--default-key-height': `${this.config.KEY_HEIGHT}px`,
          '--default-key-width': `${this.config.KEY_WIDTH}px`,
          '--default-key-x-spacing': `${this.config.KEY_X_SPACING}px`,
          '--default-key-y-spacing': `${this.config.KEY_Y_SPACING}px`,
          width: `${this.width}px`,
          height: `${this.height}px`
        },
        'font-family': `${this.font}`
      };
    },
    ...mapState('keymap', ['config', 'layer']),
    ...mapGetters('keymap', [
      'getLayer',
      'loadingKeymapPromise',
      'colorway',
      'defaults',
      'font',
      'fontSize'
    ]),
    ...mapState('app', ['layout', 'layouts', 'legends', 'previewRequested']),
    currentLayer() {
      const layout = this.layouts[this.layout];
      const keymap = this.getLayer(this.layer);
      if (isUndefined(layout) || isUndefined(keymap)) {
        return [];
      }
      // Calculate Max with given layout
      this.profile && console.time('currentLayer');
      const colorway = this.colorway;
      let curLayer = layout.map((pos, index) => {
        let _pos = Object.assign({ w: 1, h: 1 }, pos);
        const coor = this.calcKeyKeymapPos(_pos.x, _pos.y);
        const dims = this.calcKeyKeymapDims(_pos.w, _pos.h);
        const matrix = pos.matrix;
        return Object.assign(
          {
            id: index,
            layer: this.layer,
            meta: keymap[index],
            colorway: colorway,
            legends: this.legends,
            matrix
          },
          coor,
          dims
        );
      });
      this.profile && console.timeEnd('currentLayer');
      if (this.loadingKeymapPromise) {
        // signal to observers of keymap loading that UI component has finished calculations
        // this is a tricky way to introduce async signalling into a sync computed property
        const _promise = this.loadingKeymapPromise;
        this.setLoadingKeymapPromise(undefined);
        _promise();
      }
      return curLayer;
    }
  },
  watch: {
    layout(newLayout, oldLayout) {
      this.profile && console.time('layout');
      if (
        !isUndefined(newLayout) &&
        !this.isLayoutUIUpdate(newLayout, oldLayout) &&
        newLayout !== oldLayout
      ) {
        this.recalcEverything(newLayout);
      } else if (
        isUndefined(newLayout) &&
        !isUndefined(oldLayout) &&
        oldLayout !== ''
      ) {
        this.recalcEverything(oldLayout);
      }
      this.profile && console.timeEnd('layout');
    }
  },
  methods: {
    ...mapMutations('keymap', [
      'changeLayer',
      'clear',
      'initKeymap',
      'resetConfig',
      'resizeConfig',
      'setLoadingKeymapPromise'
    ]),
    ...pinia.mapActions(useStatusStore, ['append', 'scrollToEnd']),
    /**
     * Due to a quirk in how reactivity works we have to clear the layout
     * name to reset the UI to it's old value.
     * We should ignore these events to avoid updating the visual keymap.
     * If either the existing or the new layout is empty string return true.
     * We use a change in layout to decide whether to reset the keymap.
     */
    isLayoutUIUpdate(newLayout, oldLayout) {
      return newLayout === '' || oldLayout === '';
    },
    getComponent(key) {
      const { meta } = key;
      if (meta === undefined) {
        this.debug && console.log(`key ${key.id} has undefined metadata`);
        return BaseKey;
      }
      switch (meta.type) {
        case 'container':
          return ContainerKey;
        case 'layer':
          return LayerKey;
        case 'layer-container':
          return LayerContainerKey;
        case 'text':
          return AnyKey;
        default:
          return BaseKey;
      }
    },
    setSize(max) {
      this.width = max.x;
      this.height = max.y;
    },
    recalcEverything(newLayout) {
      this.profile && console.time('layout::reset');
      this.resetConfig();
      this.changeLayer(0);
      this.profile && console.time('layout::initkeymap');
      if (this.getLayer(0).length === 0) {
        this.initKeymap({
          layer: 0,
          layout: this.layouts[newLayout]
        });
      }
      this.profile && console.timeEnd('layout::initkeymap');
      this.profile && console.timeEnd('layout::reset');

      this.profile && console.time('layout::scale');
      const layout = this.layouts[newLayout];
      if (isUndefined(layout)) {
        const msg = `\n\nWARNING: layout ${newLayout} does not exist on this keyboard\n\n`;
        console.log(msg);
        this.append(msg);
        this.scrollToEnd();
        return;
      }
      const max = layout.reduce(
        (acc, pos) => {
          let _pos = Object.assign({ w: 1, h: 1 }, pos);
          const coor = this.calcKeyKeymapPos(_pos.x, _pos.y);
          const dims = this.calcKeyKeymapDims(_pos.w, _pos.h);
          acc.x = Math.max(acc.x, coor.x + dims.w);
          acc.y = Math.max(acc.y, coor.y + dims.h);
          return acc;
        },
        {
          x: 0,
          y: 0
        }
      );
      if (max.x > this.defaults.MAX_X) {
        this.resizeConfig(max);
        max.x *= this.config.SCALE;
        max.y *= this.config.SCALE;
      }
      this.setSize(max);
      this.profile && console.timeEnd('layout::scale');
    }
  }
};
</script>
<style>
.fade-enter-active {
  transition: all 0.5s ease;
}
.fade-leave-active {
  transition: all 0.8s cubic-bezier(1, 0.5, 0.8, 1);
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>
