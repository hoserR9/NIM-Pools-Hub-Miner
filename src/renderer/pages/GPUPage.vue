<template>
  <div class="container">
    <div class="row py-2">
      <div class="col">
        <input
          placeholder="Nimiq Address"
          :value="address"
          @input="updateAddress"
          class="nq-input-s text-center"
          style="width: 100%; text-transform:uppercase;"
        />
      </div>
    </div>
    <div class="row space-between py-2">
      <BalanceCard />
      <HashrateCard />
    </div>
    <div class="row py-2">
      <line-chart :chart-data="datacollection" style="height: 120px; width:100%"></line-chart>
    </div>
    <div class="row">
      <div class="col space-between">
        <div class="row" style="width: 100%; align-items: center; margin-top:5px">
          <h1 class="nq-h1">Pool: {{ displayName }}</h1>
        </div>
        <button
          class="nq-button orange"
          @click="selectPool"
          :class="miningGPU || miningCPU ? 'disabled' : ''"
        >Change Pool</button>
      </div>
      <div class="col space-between">
        <h2 class="nq-h2" style="text-align: center; height: 50px">
          Open
          <a class="link" @click="openSettings">Settings</a>
          <br />for miner tunning
        </h2>
        <button v-if="!miningGPU" @click="startMining" class="nq-button light-blue">Start Mining</button>
        <button v-else @click="stopMining" class="nq-button red">Stop Mining</button>
      </div>
    </div>
    <Alert :message="alertMessage" v-if="showAlert" @hideAlert="hideAlertHandler" />
    <span class="app-version">v: {{ appVersion }}</span>
    <!-- Disabled until Temperatures are reliable, getting 16ºC on my PC, thanks to wmic reporting bad temperatures -->
    <!-- <span class="cpu-temp">CPU: {{ cpuTemp }} ºC</span> -->
  </div>
</template>

<script>
import { ipcRenderer } from "electron";
import HashrateCard from "@/components/HashrateCard.vue";
import BalanceCard from "@/components/BalanceCard.vue";
import LineChart from "@/components/LineChart.js";
import RangeSlider from "@/components/RangeSlider.vue";
import Alert from "@/components/Alert.vue";

import * as NimiqUtils from "@nimiq/utils";

import { mapActions, mapState } from "vuex";

export default {
  name: "cpu-page",
  components: {
    HashrateCard,
    BalanceCard,
    LineChart,
    RangeSlider,
    Alert,
  },
  data() {
    return {
      alertMessage: null,
      showAlert: false,
      cpuTemp: 0,
    };
  },
  computed: {
    ...mapState({
      appVersion: (state) => state.views.appVersion,
      address: (state) => state.settings.address,
      host: (state) => state.settings.host,
      port: (state) => state.settings.port,
      displayName: (state) => state.settings.displayName,
      miningCPU: (state) => state.hashrate.mining.cpu,
      miningGPU: (state) => state.hashrate.mining.gpu,
      gpuArray: (state) => state.hashrate.gpuArray,
      gpuTime: (state) => state.hashrate.gpuTime,
    }),
    datacollection() {
      return {
        labels: this.gpuTime,
        datasets: [
          {
            label: "Hashrate",
            borderColor: "#E9B213",
            pointBorderColor: "#E9B213",
            pointBackgroundColor: "#E9B213",
            pointBorderWidth: 5,
            pointRadius: 2,
            fill: false,
            borderWidth: 3,
            data: this.gpuArray,
          },
        ],
      };
    },
  },
  created() {
    // If updated finished downloading Alert the user the app is going to restart
    ipcRenderer.on("update-downloaded", () => {
      this.alert("New update downloaded, restarting in 5 seconds...");
    });

    // Disabled until Temperatures are reliable, getting 16ºC on my PC, thanks to wmic reporting bad temperatures
    // Check immediately what's the CPU Temp and add an interval
    /* ipcRenderer.send("cpu-temp");
    setInterval(() => ipcRenderer.send("cpu-temp"), 1 * 60 * 1e3); // Check CPU temperature every x minutes
    ipcRenderer.on("cpu-temp-reply", (event, message) => {
      this.cpuTemp =
        message > 0 ? message : "Run this app as Admin to check CPU ";
    }); */
  },
  destroyed() {
    ipcRenderer.removeAllListeners("update-downloaded");
  },
  methods: {
    ...mapActions(["setAddress", "setMiningGPU", "setcurrentPage"]),
    startMining() {
      ipcRenderer.send("stopMining", "gpu");

      const address = this.address;
      const host = this.host;
      const port = this.port;
      console.log({
        address,
        host,
        port,
      });
      if (address === "" || host === "" || port === "") {
        this.alert("Fill the Address input and choose a Pool");
        return;
      } else {
        try {
          NimiqUtils.ValidationUtils.isUserFriendlyAddress(address);
        } catch (e) {
          this.alert(e);
          return;
        }
        this.setMiningGPU(true);

        ipcRenderer.send("startMining", {
          address,
          host,
          port,
          gpu: true,
        });
      }
    },
    stopMining() {
      this.setMiningGPU(false);
      ipcRenderer.send("stopMining", "gpu");
    },
    alert(e) {
      this.alertMessage = e;
      this.showAlert = true;
    },
    hideAlertHandler() {
      this.showAlert = false;
      this.alertMessage = null;
    },
    selectPool() {
      this.$router.replace("pools");
    },
    updateAddress(e) {
      this.setAddress(e.target.value);
    },
    openSettings() {
      this.setcurrentPage("settings");
      this.$router.replace("/settings");
    },
  },
};
</script>

<style>
.container {
  width: calc(630px - 100px);
  height: calc(100% - 47px);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  margin-left: auto;
  margin-right: auto;
}

.row {
  display: flex;
  width: 80vw;
  justify-content: space-around;
}

.col {
  display: flex;
  width: 100%;
  flex-direction: column;
  justify-content: space-around;
  align-items: center;
}

.space-between {
  justify-content: space-between !important;
}

.py-2 {
  padding-top: 8px;
  padding-bottom: 8px;
}

.py-3 {
  padding-top: 14px;
  padding-bottom: 14px;
}

.text-center {
  text-align: center;
}

.app-version {
  position: absolute;
  bottom: 0;
  right: 5px;
  font-size: 12px;
}

.cpu-temp {
  position: absolute;
  bottom: 0;
  left: 5px;
  font-size: 12px;
}

.nq-button {
  margin: 1.5rem !important;
}

.nq-h1 {
  margin: 0 !important;
}

.nq-h2 {
  margin: 0 !important;
}

.disabled {
  pointer-events: none;
  opacity: 0.65;
}

.link {
  font-weight: 800;
  color: var(--nimiq-green);
  cursor: pointer;
}
</style>
