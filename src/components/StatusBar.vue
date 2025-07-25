<template>
  <div class="backend-status">
    <div class="qmk-branding">
      <a target="_blank" rel="noopener" href="https://zonekeyboards.cl"
        ><div title="Visita nuestra página web" class="zk-logo"></div
      ></a>
      <div class="qmk-app-name">ZK Configurator</div>
      <div class="bes-version">
        {{ $t('apiVersionPrefix') }}
        <span class="version-num">v{{ version }}</span>
      </div>
    </div>
    <div class="bes-title">
      <div class="bes-status">
        <div class="bes-status-left" :class="currentStatusClass">
          <ul>
            <li></li>
          </ul>
        </div>
        <div class="bes-status-center">
          {{ $t('serverIs') }}
        </div>
        <div class="bes-status-right">{{ jobs }}</div>
      </div>
    </div>
    <div class="bes-discord">
      <a v-if="hasError" target="_blank" rel="noopener" :href="discordLink">
        Error? Let us know on QMK Discord.
      </a>
    </div>
    <div class="bes-controls" @click="clickSettings">
      <font-awesome-icon
        v-if="!settingsPanelVisible"
        icon="chevron-left"
        fixed-width
      />
      <font-awesome-icon icon="cog" size="lg" />
      <font-awesome-icon
        v-if="settingsPanelVisible"
        icon="chevron-right"
        fixed-width
      />
      {{ $t('settings') }}
    </div>
  </div>
</template>
<script>
import { backend_status_url } from '@/store/modules/constants';
import { mapState, mapMutations } from 'vuex';
/**
 * checkStatus - check status component to poll API for errors and status
 * @return {object} Vue app component
 */

const highWaterMark = 20;
const warningWaterMark = 10;
export default {
  name: 'status-bar',
  computed: {
    ...mapState('app', ['settingsPanelVisible']),
    currentStatusClass() {
      switch (this.status) {
        case 'running':
          return 'bes-status-running';
        case 'degraded':
          return 'bes-status-degraded';
        default:
          return 'bes-status-down';
      }
    },
    jobCountClass() {
      if (this.jobCount < warningWaterMark) {
        return '';
      }
      if (this.jobCount < highWaterMark) {
        return 'bes-odd-job-count';
      }
      return 'bes-high-job-count';
    },
    discordLink() {
      return 'https://discord.gg/qmk';
    }
  },
  methods: {
    ...mapMutations('app', ['setSettingsPanel']),
    getPollInterval() {
      return 25000 + 5000 * Math.random();
    },
    async fetchData() {
      try {
        const resp = await fetch(backend_status_url);
        if (resp.ok) {
          const data = await resp.json();
          this.version = data.version;
          this.jobCount = parseInt(data.queue_length, 10);
          if (this.jobCount === 0) {
            this.jobs = this.$t('ready');
          } else {
            this.jobs = `${this.jobCount} ${this.$t('jobsAhead')}`;
          }
          if (this.jobCount < highWaterMark) {
            this.status = data.status;
            this.hasError = false;
          } else {
            this.status = 'Redis is probably down. Please contact devs on ';
            this.hasError = true;
          }
        }
      } catch (error) {
        this.status = 'DOWN';
        this.hasError = true;
        console.error('API status error', error);
      }
      setTimeout(this.fetchData, this.getPollInterval());
    },
    clickSettings() {
      this.setSettingsPanel(!this.settingsPanelVisible);
    }
  },
  data: () => {
    return {
      status: '',
      version: '0.1',
      jobCount: 0,
      hasError: false,
      jobs: 'Initializing'
    };
  },
  mounted() {
    setTimeout(this.fetchData, 1000);
  }
};
</script>
<style>
.bes-high-job-count {
  color: red;
}
.bes-odd-job-count {
  color: yellow;
}
</style>
