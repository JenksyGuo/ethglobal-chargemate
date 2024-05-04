<template>
  <div id="FileUploader">
    <!-- <b-form-file
      v-model="files"
      v-if="address!==null"
      :state="Boolean(files)"
      placeholder="Choose a file or drop it here..."
      :multiple="true"
    ></b-form-file> -->

    <p v-if="address===null">Connect Your Wallet </p>
    
    <br />
    <br />

    <br /><br />
      <p v-if="address!==null && sending===false">
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-wallet" viewBox="0 0 16 16">
          <path d="M0 3a2 2 0 0 1 2-2h13.5a.5.5 0 0 1 0 1H15v2a1 1 0 0 1 1 1v8.5a1.5 1.5 0 0 1-1.5 1.5h-12A2.5 2.5 0 0 1 0 12.5zm1 1.732V12.5A1.5 1.5 0 0 0 2.5 14h12a.5.5 0 0 0 .5-.5V5H2a2 2 0 0 1-1-.268M1 3a1 1 0 0 0 1 1h12V2H2a1 1 0 0 0-1 1"/>
        </svg>
        {{ address[0].slice(0,3) }}...{{ address[0].slice(-4) }}</p>
      <br /><br />

    <div v-if="sending===false">
      <b-button
        v-if="address===null"
        variant="secondary"
        @click="requestAccount()"
      > 
        Connect My Wallet
      </b-button>
      
      <b-button
        v-if="address!==null"
        variant="outline-dark"
        @click="uploadFile(files)"
      > 
        Connect MANTLE
      </b-button>
      <br />
      <br />
      <b-button
        v-if="address!==null"
        variant="outline-dark"
        @click="switchtoBase()"
      > 
      Connect &nbsp;&nbsp;BASE&nbsp;&nbsp;&nbsp;&nbsp;
      </b-button>
      
    </div>


    <div v-if="sending===true">
      <b-spinner label="Spinning"></b-spinner>
      <b-card-text>Sending to IPFS</b-card-text>
    </div>


  </div>
</template>

<script>
  import { Web3Storage } from 'web3.storage';
  import Dwetransfer from '../../artifacts/contracts/Dwetransfer.sol/Dwetransfer.json'
  import { BigNumber, ethers } from "ethers";

  export default {
    name: "FileUploader",
    data() {
      return {
        files: null,
        sending: false,
        address: null,
        fileId: null,
      }
    },
    methods: {
      async switchtoBase() {
        window.ethereum.request({
          method: 'wallet_switchEthereumChain',
          params: [{ chainId: '0x14a34' }]
        }).then(response => console.log(response))
      },

      async requestAccount() {
        var currentAccount = null;

        if(window.ethereum) {
          console.log("Metamask detected");
        } else {
          console.log("Metamask not detected...");
        }

        currentAccount = await window.ethereum.request({ method: 'eth_requestAccounts' });

        this.address = currentAccount;
        
        console.log("Current Account :" + currentAccount);
      },

      async uploadFile(files) {
        console.log(files);

        const client = new Web3Storage({ token: process.env.VUE_APP_WEB3STORAGE_API_TOKEN });

        const contractAddress = "0x5FbDB2315678afecb367f032d93F642f64180aa3";
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner()

        try {
          const contract = new ethers.Contract(contractAddress, Dwetransfer.abi, signer)
          const options = {value: ethers.utils.parseEther("1")};
          let rootCid = "";

          
          try {
              this.sending = true;
              rootCid = await client.put(files);
              console.log("Successfully sent to IPFS");
              console.log("rootCid :" + [rootCid]);
          } catch {
            this.sending = false;
            this.$emit('update-error', true);
            console.log("Failed to send file... ");
          }

          let tx = await contract.uploadFile(rootCid, options);
          let rc = await tx.wait();
          let BigNumberFileIdHex = rc.events[0].args["fileId"]._hex;
          this.fileId = BigNumber.from(BigNumberFileIdHex).toNumber();
          console.log(tx);
          console.log("rc:");
          console.log(rc);
          console.log(rc.events[0].args["fileId"]._hex);
          console.log("fileId");
          console.log(this.fileId);

          this.sending = false;
          this.$emit('update-fileid', this.fileId);
          this.$emit('update-cids', [rootCid]);
          console.log("contract executed successfully!")
          console.log("https://localhost:8080/download/" + this.fileId);
          
            
        } catch {
            this.$emit('update-error', true);
            console.log("Failed to execute contract!");
        }
      } 
    }
  }
</script>