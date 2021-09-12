<template>
  <div id="app">
    <section class="section">
      <div class="container">
      <!-- Login button -->
      <div class="columns">
        <div class="column is-1 is-offset-10">
          <b-button type="is-light" @click="connectWallet" :disabled="isConnected">
            {{isConnected ? accountMinimized : "Login"}}
          </b-button>
        </div>
      </div>

      <!-- Title -->
     <div class="columns">
       <div class="column is-12">
         <h1 class="title has-text-centered">
            Wave App!
          </h1>
          <p class="subtitle has-text-centered">
            Israel's waving app, to encourage me to continue learning Web3!
          </p>
       </div>
     </div>

     <!-- Textbox & Wave Button -->
     <div class="columns" v-if="isConnected">
      <!-- Wave Textbox -->
      <div class="column is-8">
        <b-input v-model="message" :disabled="isMining"/>
      </div>

       <!-- Wave Button -->
       <div class="column is-4">
         <b-button type="is-primary" expanded :loading="isMining" @click="wave">
           {{ isMining ? "Mining..." : "Wave here!" }}
         </b-button>
       </div>
     </div>

     <div class="columns" v-else>
       <div class="column is-12">
         <p class="is-size-6 has-text-centered has-text-weight-light">
           Please connect metamask to interact with this Web3 app
         </p>
       </div>
     </div>

     <!-- Existing waves -->
     <section v-if="isConnected" class="section">
       <div class="container">
         <article class="media" v-for="(wave, i) in allWaves" :key="i">
          <div class="media-content">
            <div class="content">
              <p>
                <a :href="`https://rinkeby.etherscan.io/address/${wave.address}`" target="_blank">
                <strong>{{ minimizeAccount(wave.address) }}</strong></a> <small>said {{ formatDate(wave.timestamp) }} ago</small>
                <br>
                {{ wave.message }}
              </p>
            </div>
          </div>
        </article>
       </div>
     </section>

    </div>
    </section>
  </div>
</template>

<script>
import { ethers } from "ethers";
import { abi } from "./utils/WavePortal.json"

import { formatDistance } from 'date-fns'

const { ethereum } = window;
const contractAddress = "0x5ca4C72b671c14a4bFF7deCaEE0bbdd6dfff42C3";


export default {
  data(){
    return {
      contractABI: abi,
      contractAddress: contractAddress,

      ethereum: ethereum,


      message: "",

      account: null,

      isMining: false,

      allWaves: []
    }
  },

  computed: {
    isConnected(){
      return Boolean(this.account);
    },

    accountMinimized(){
     return this.minimizeAccount(this.account);
    }
  },

  mounted() {
    if(this.ethereum) {
      this.getWallet();
    }
  },

  methods: {
    minimizeAccount(account){
      const firstLetters = account.slice(0, 5);
      const lastLetters = account.slice(-5);

      return `${firstLetters}...${lastLetters}`;
    },

    getWallet(){
      ethereum.request({method: "eth_accounts"})
      .then((accounts) => {
        if(accounts.length !== 0) {
          // this.account = accounts[0]
          console.log(accounts[0]);

          // Get waves
          this.getAllWaves();
        }
      })
      .catch((err) => console.error(err));
    },

    connectWallet(){
      this.ethereum.request({method: "eth_requestAccounts"})
      .then(accounts=>{
        this.account=accounts[0];
      })
      .catch(err => console.error(err));
    },

    getContractData() {
      const provider = new ethers.providers.Web3Provider(this.ethereum);
      console.log(provider);
      const signer = provider.getSigner();
      console.log(signer);
      const waveportalContract = new ethers.Contract(this.contractAddress, this.contractABI, signer);
      console.log(waveportalContract);

      // Console log the events? not sure...
      waveportalContract.on("newWave", (author, oldValue, newValue, event) => {
        console.log({ author, oldValue, newValue, event });
      })

      return {
        provider,
        signer,
        waveportalContract
      }
    },

    async getAllWaves(){
      this.allWaves = [];

      const { waveportalContract } = this.getContractData();

      const waves = await waveportalContract.getAllWaves();

      const allWaves = waves.map(wave => {
        return {
          address: wave.waver,
          timestamp: new Date(wave.timestamp * 1000),
          message: wave.message
        }
      })

      this.allWaves = allWaves.reverse();
    },

    async wave(){
      const { waveportalContract } = this.getContractData();

      let count = await waveportalContract.getTotalWaves();
      console.log("Total wave count: ", count.toNumber());

      try {
        const waveTxn = await waveportalContract.wave(this.message);
        console.log("Mining... ", waveTxn.hash);
        this.isMining = true;

        await waveTxn.wait();
        console.log("Mined! ", waveTxn.hash);
        this.isMining = false;
        this.message = "";

        this.$buefy.toast.open({
            message: 'Your transaction has been mined!',
            type: 'is-success'
        })

        count = await waveportalContract.getTotalWaves();
        console.log("New total wave count: ", count.toNumber());
        
      } catch (error) {
        this.$buefy.toast.open({
            message: `${error.error.message}`,
            type: 'is-danger',
            duration: 5000
        })
      }

      this.getAllWaves();
    },

    formatDate(date){
      const today = new Date();

      return formatDistance(date, today);
    }
  }
}
</script>
