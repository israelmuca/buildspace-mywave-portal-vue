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

     <!-- Wave Button -->
     <div class="columns">
       <div class="column is-12">
         <b-button type="is-primary" expanded :loading="isMining" @click="wave">
           {{ isMining ? "Mining..." : "Wave here!" }}
         </b-button>
       </div>
     </div>
    </div>
    </section>
  </div>
</template>

<script>
import { ethers } from "ethers";
import {abi} from "./utils/WavePortal.json"

const {ethereum} = window;
const contractAddress = "0x79FF8435C41aca64B4b71AF3D5b1Af9c5deaebb5";


export default {
  data(){
    return {
      contractABI: abi,
      contractAddress: contractAddress,

      ethereum: ethereum,


      account: null,

      isMining: false,
    }
  },

  computed: {
    isConnected(){
      return Boolean(this.account);
    },

    accountMinimized(){
      const firstLetters = this.account.slice(0, 4);
      const lastLetters = this.account.slice(-4);

      return `${firstLetters}...${lastLetters}`;
    }
  },

  mounted() {
    if(this.ethereum) {
      this.getWallet();
    }
  },

  methods: {
    getWallet(){
      ethereum.request({method: "eth_accounts"})
      .then((accounts) => {
        if(accounts.length !== 0) {
          // this.account = accounts[0]
          console.log(accounts[0]);
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

    async wave(){
      const provider = new ethers.providers.Web3Provider(this.ethereum);
      console.log(provider);
      const signer = provider.getSigner();
      console.log(signer);
      const waveportalContract = new ethers.Contract(this.contractAddress, this.contractABI, signer);
      console.log(waveportalContract);

      let count = await waveportalContract.getTotalWaves();
      console.log("Total wave count: ", count.toNumber());

      const waveTxn = await waveportalContract.wave();
      console.log("Mining... ", waveTxn.hash);
      this.isMining = true;

      await waveTxn.wait();
      console.log("Mined! ", waveTxn.hash);
      this.isMining = false;

      count = await waveportalContract.getTotalWaves();
      console.log("New total wave count: ", count.toNumber());
    }
  }
}
</script>
