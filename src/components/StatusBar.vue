<template>
  <div class="backend-status">
    <div class="qmk-branding">
      <div class="qmk-logo"></div>
      <div class="qmk-app-name">
        QMK Configurator
      </div>
      <div class="bes-version">
        {{ $t('message.apiVersionPrefix') }}
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
          {{ $t('message.serverIs') }}
        </div>
        <div class="bes-status-right">{{ jobs }}</div>
      </div>
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
      {{ $t('message.settings') }}
    </div>
    <a v-if="hasError" target="_blank" :href="discordLink">QMK Discord</a>
  </div>
</template>
<script>
import axios from 'axios';
import escape from 'lodash/escape';
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
      return this.status === 'UP' ? 'bes-status-up' : 'bes-status-down';
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
      return 'https://discord.gg/Uq7gcHh';
    }
  },
  methods: {
    ...mapMutations('app', ['setSettingsPanel']),
    getPollInterval() {
      return 25000 + 5000 * Math.random();
    },
    fetchData() {
      axios
        .get(backend_status_url)
        .then(({ data }) => {
          this.version = data.version;
          this.jobCount = parseInt(data.queue_length, 10);
          this.jobs = `${this.$t('message.currentQueue')}: ${this.jobCount}`;
          if (this.jobCount === 0) {
            this.jobs = this.$t('message.ready');
          } else {
            this.jobs = `${this.jobCount} ${this.$t('message.jobsAhead')}`;
          }
          if (this.jobCount < highWaterMark) {
            var stat = data.status;
            stat = stat === 'running' ? 'UP' : stat;
            this.status = escape(`${stat}`);
            this.hasError = false;
          } else {
            this.status = 'Redis is probably down. Please contact devs on ';
            this.hasError = true;
          }
        })
        .catch(json => {
          this.status = 'DOWN';
          this.hasError = true;
          console.error('API status error', json);
        });
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
