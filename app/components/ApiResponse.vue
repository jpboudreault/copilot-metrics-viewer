<template>
  <v-container>
    <div class="copy-container">
      <v-btn @click="checkMetricsDataQuality">Check Metric data quality</v-btn>
      <v-spacer/>
      <v-btn @click="copyToClipboard('metricsJsonText')">Copy Metrics to Clipboard</v-btn>
    </div>
    <transition name="fade">
      <div v-if="showQualityMessage || showCopyMessage || showSeatMessage" :class="{'copy-message': true, 'error': isError}">{{ message }}</div>
    </transition>
      <br><br>

      <!-- Displaying the JSON object -->
      <v-card max-height="575px" class="overflow-y-auto">
          <pre ref="metricsJsonText">{{ JSON.stringify(originalMetrics, null, 2) }}</pre>
      </v-card>
      <br>
      
      <div class="copy-container">
        <v-btn @click="showSeatCount">Show Assigned Seats count</v-btn>
        <transition name="fade">
          <div v-if="showSeatMessage" :class="{'copy-message': true, 'error': isError}">{{ message }}</div>
        </transition>
      </div>

      <v-card max-height="575px" class="overflow-y-auto">
          <pre ref="seatJsonText">{{ JSON.stringify(seats, null, 2) }}</pre>
      </v-card>
      <br>

      <div class="copy-container">
        <v-btn @click="fetchUserEmails">Fetch User Emails</v-btn>
        <transition name="fade">
          <div v-if="showEmailMessage" :class="{'copy-message': true, 'error': isError}">{{ emailMessage }}</div>
        </transition>
      </div>

      <v-card max-height="575px" class="overflow-y-auto">
          <pre ref="emailJsonText">{{ JSON.stringify(userEmails, null, 2) }}</pre>
      </v-card>
      <br>

      <v-card max-height="575px" class="overflow-y-auto">
          <pre ref="emailMapText">{{ JSON.stringify(userEmailsMap, null, 2) }}</pre>
      </v-card>
      <br>
  </v-container>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { MetricsValidator } from '@/model/MetricsValidator';
import type { CopilotMetrics } from '@/model/Copilot_Metrics';
import type { Metrics } from '@/model/Metrics';

export default defineComponent({
  name: 'ApiResponse',
  props: {
    originalMetrics: {
      type: Array as () => CopilotMetrics[],
      required: true
    },
    metrics: {
      type: Array as () => Metrics[],
      required: true
    },
    seats: {
      type: Array,
      required: true
    },
    userEmailsMap: {
      type: Object as () => Record<string, string>,
      required: true
    }
  },
  data() {
    return {
      showCopyMessage: false,
      showSeatMessage: false,
      showQualityMessage: false,
      showEmailMessage: false,
      isError: false,
      message: '',
      emailMessage: '',
      qualityMessage: ''
    };
  },
  methods: {
    copyToClipboard(refName: string) {
      const jsonText = this.$refs[refName] as HTMLElement;
      navigator.clipboard.writeText(jsonText.innerText)
        .then(() => {
          this.message = 'Copied to clipboard!';
          this.isError = false;
        })
        .catch(err => {
          this.message = 'Could not copy text!';
          this.isError = true;
          console.error('Could not copy text: ', err);
        });

      this.showCopyMessage = true;
      setTimeout(() => {
        this.showCopyMessage = false;
      }, 3000);
    },
    
    showSeatCount() {
      const seatCount = this.seats.length;
      //console.log('Seat count:', seatCount);
      this.message = `Seat count: ${seatCount}`;

      this.showSeatMessage = true;
      setTimeout(() => {
        this.showSeatMessage = false;
      }, 3000);
    },

    checkMetricsDataQuality() {
      const validator = new MetricsValidator(this.originalMetrics);
    
     // console.log(validator);
      // create a new MetricsValidator object
      // check all the metrics
      const results = validator.checkAllMetrics();
      //console.log(results);
      // check if all the metrics are valid
      const allValid = Object.values(results).every((result: any) => result.length === 0);

      if (allValid) {
        this.message = 'All metrics are valid!';
        this.isError = false;
      } else {
        this.message = 'Some metrics might be inconsistent, please double check the API response.\n';
        this.isError = true;
        let typeCounter = 1;
        for (const [key, value] of Object.entries(results)) {
          if (value.length > 0) {
            this.message += `\n${typeCounter}).  ${key}:\n${JSON.stringify(value, null, 2)}\n`;
            typeCounter++;
          }
        }
      }

      this.showQualityMessage = true;
      setTimeout(() => {
        this.showQualityMessage = false;
      }, 6000);
    },

    async fetchUserEmails() {
      try {
        const response = await fetch('/api/graphql', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.$config.githubToken}`
          },
          body: JSON.stringify({
            query: `
              query {
                organization(login: "${this.$config.githubOrg}") {
                  membersWithRole(first: 100) {
                    nodes {
                      login
                      email
                    }
                  }
                }
              }
            `
          })
        });

        const data = await response.json();
        this.userEmails = data.data.organization.membersWithRole.nodes;
        this.emailMessage = 'User emails fetched successfully!';
        this.isError = false;
      } catch (error) {
        this.emailMessage = 'Error fetching user emails!';
        this.isError = true;
        console.error('Error fetching user emails: ', error);
      }

      this.showEmailMessage = true;
      setTimeout(() => {
        this.showEmailMessage = false;
      }, 3000);
    }
  }
});
</script>

<style scoped>
.tiles-container {
  display: flex;
  justify-content: flex-start;
  flex-wrap: wrap;
}
.copy-container {
  display: flex;
  align-items: center;
}
.copy-message {
  margin-left: 10px;
  font-family: Roboto, sans-serif;
}
.copy-message.error {
  color: red;
}
.copy-message:not(.error) {
  color: darkgreen;
}
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to {
  opacity: 0;
}
</style>
