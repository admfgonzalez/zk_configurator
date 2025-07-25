<template>
  <div>
    <div ref="console">
      <controllerTop v-if="appInitialized" />
      <statusPanel />
      <controllerBottom />
    </div>
    <div class="hint hint-right">
      <a
        href="https://github.com/qmk/qmk_toolbox/releases"
        v-tooltip="$t('downloadToolbox.label')"
        target="_blank"
        rel="noopener"
        >{{ $t('downloadToolbox.label') }}</a
      >
    </div>
    <div class="split-content">
      <div class="left-side">
        <layerControl />
      </div>
      <div class="right-side">
        <div class="keymap--area">
          <label class="keymap--label" v-tooltip="$t('ColorwayTip.title')">
            {{ $t('keymap.label') }}:
            <font-awesome-icon
              v-if="continuousInput"
              icon="keyboard"
              fixed-width
            />
          </label>
          &nbsp;
          <!-- maintain spacing for paragraph -->
          <select
            class="keymap--keyset"
            id="colorway-select"
            v-model="curIndex"
            @focus="focus"
            @blur="blur"
          >
            <option
              class="option"
              v-for="(name, index) in displayColorways"
              :key="index"
              :value="index"
            >
              {{ name }}
            </option>
          </select>
          <a
            id="favorite-colorway"
            v-tooltip="$t('favoriteColor')"
            @click="favColor"
            :class="{
              active: isFavoriteColor
            }"
          >
            <font-awesome-icon icon="star" size="lg" fixed-width />
          </a>
          &nbsp;
          <select
            class="keymap--keyset"
            id="font-select"
            v-model="curFont"
            @focus="focus"
            @blur="blur"
          >
            <option
              class="option"
              v-for="(name, index) in fonts"
              :key="index"
              :value="name"
            >
              {{ name }}
            </option>
          </select>
          &nbsp;
          <select
            class="keymap--keyset"
            id="font-size-select"
            v-model="curFontSize"
            @focus="focus"
            @blur="blur"
          >
            <option
              class="option"
              v-for="(size, index) in fontSizes"
              :key="index"
              :value="size"
            >
              {{ size }}mm
            </option>
          </select>
          &nbsp;
          <select
            class="keymap--keyset"
            id="os-keyboard-layout-select"
            v-model="osKeyboardLayout"
            @focus="focus"
            @blur="blur"
          >
            <option
              v-for="osLayout in sortedOSKeyboardLayouts"
              :key="osLayout"
              :value="osLayout"
            >
              {{ $t(`settingsPanel.osKeyboardLayout.label.${osLayout}`) }}
            </option>
          </select>
        </div>

        <visualKeymap :profile="false" />
        <span class="keymap--count"
          ><span class="keymap--counter">{{ keyCount }}</span
          >Keys</span>
      </div>
    </div>
  </div>
</template>

<script>
import capitalize from 'lodash/capitalize';
import { mapMutations, mapState, mapGetters, mapActions } from 'vuex';
import ControllerTop from '@/components/ControllerTop.vue';
import StatusPanel from '@/components/StatusPanel.vue';
import ControllerBottom from '@/components/ControllerBottom.vue';
import VisualKeymap from '@/components/VisualKeymap.vue';
import LayerControl from '@/components/LayerControl.vue';

export default {
  name: 'MainComponent',
  data() {
    return {
      fonts: [
        'Roboto',
        'Roboto Mono',
        'Fira Code',
        'JetBrains Mono',
        'Consolas',
        'Source Code Pro',
        'serif',
        'Arial',
        'Verdana',
        'Courier New',
        'Times New Roman'
      ],
      fontSizes: [1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 5, 5.5, 6, 6.5, 7, 7.5, 8, 8.5, 9]
    };
  },
  components: {
    ControllerTop,
    StatusPanel,
    ControllerBottom,
    VisualKeymap,
    LayerControl
  },
  computed: {
    ...mapState('app', ['appInitialized', 'configuratorSettings', 'osKeyboardLayouts']),
    ...mapGetters('app', ['keyCount']),
    ...mapState('keymap', ['continuousInput']),
    ...mapGetters('keymap', ['colorwayIndex', 'colorways', 'size', 'font', 'fontSize']),
    osKeyboardLayout: {
      get() {
        return this.configuratorSettings.osKeyboardLayout;
      },
      async set(value) {
        try {
          await this.changeOSKeyboardLayout(value);
        } catch (error) {
          console.error(
            'Setting a new value for the OS keyboard layout failed!',
            error
          );
        }
      }
    },
    sortedOSKeyboardLayouts: function () {
      // Locale-aware sort of the OS keyboard layouts by their labels.
      const translatedLayoutName = (osLayout) =>
        this.$t(`settingsPanel.osKeyboardLayout.label.${osLayout}`);
      return [...this.osKeyboardLayouts].sort((a, b) =>
        translatedLayoutName(a).localeCompare(translatedLayoutName(b))
      );
    },
    curFont: {
      get() {
        return this.font;
      },
      set(value) {
        this.setFont(value);
      }
    },
    curFontSize: {
      get() {
        return this.fontSize;
      },
      set(value) {
        this.setFontSize(value);
      }
    },
    curIndex: {
      get() {
        return this.colorwayIndex;
      },
      set(value) {
        this.nextColorway(value);
      }
    },
    displayColorways() {
      return this.colorways.map((keyset) => {
        return keyset
          .replace(/-/g, ' ')
          .split(' ')
          .map((word) => capitalize(word))
          .join(' ')
          .replace(/Gmk/, 'GMK')
          .replace(/^Sa/, 'SA')
          .replace(/^Dsa/, 'DSA')
          .replace(/^Jtk/, 'JTK')
          .replace(/Kat/, 'KAT')
          .replace(/Wob/, 'WOB')
          .replace(/Ta/, 'TA')
          .replace(/Mt3/, 'MT3')
          .replace(/Dcs/, 'DCS')
          .replace(/Dev Tty/, '/dev/tty')
          .replace(/ ?Plus/g, '+')
          .replace(/ ?Dot ?/g, '.')
          .replace(/Ascii/, 'ASCII');
      });
    },
    redditPost() {
      return 'https://www.reddit.com/r/MechanicalKeyboards/comments/aio97b/qmk_configurator_updates_beta_need_your_input/';
    },
    isFavoriteColor() {
      return (
        this.configuratorSettings.favoriteColor &&
        this.displayColorways[this.curIndex].toLowerCase() ===
          this.configuratorSettings.favoriteColor.toLowerCase()
      );
    }
  },
  methods: {
    ...mapActions('app', ['setFavoriteColor', 'initKeypressListener', 'changeOSKeyboardLayout']),
    ...mapMutations('keymap', ['nextColorway', 'setFont', 'setFontSize']),
    ...mapMutations('app', [
      'resetListener',
      'stopListening',
      'startListening'
    ]),
    favColor() {
      if (this.isFavoriteColor) {
        this.setFavoriteColor('');
      } else {
        this.setFavoriteColor(this.displayColorways[this.curIndex]);
      }
    },
    focus() {
      this.stopListening();
    },
    blur() {
      this.startListening();
    }
  },
  async mounted() {
    await this.initKeypressListener();
    // Loading favorite color
    if (this.configuratorSettings.favoriteColor) {
      const favoriteColor =
        this.configuratorSettings.favoriteColor.toLowerCase();
      this.curIndex = this.displayColorways.findIndex(
        (color) => color.toLowerCase() === favoriteColor
      );
    }
  },
  beforeDestroy() {
    this.resetListener();
  }
};
</script>
<style lang="scss">
.hint-right {
  display: grid;
  justify-content: end;
}
#colorway-select {
  font-family: 'Roboto', 'Helvetica Neue', Helvetica, Arial, sans-serif;
}
.beta-feedback {
  position: fixed;
  right: 10px;
  top: 30px;
}
.beta-button {
  height: 30px;
  font-size: 15px;
  border-radius: 9px;
  cursor: pointer;
}
.keymap--label {
  float: left;
}
.keymap--counter {
  display: inline-block;
  padding: 0 5px;
  margin-top: 2px;
}
.keymap--count {
  float: right;
  color: #999;
}
.keymap--keyset {
  float: right;
  border-radius: 4px;
  border: 1px solid;
  &:focus {
    outline: 2px solid black;
  }
}
.keymap--area {
  margin-top: 1em;
  margin-bottom: 2em; /* Increased margin */
  overflow: hidden; /* Clearfix */
}
</style>
