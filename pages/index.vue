<template>
  <div class="main flex">
    <div
      class="mint flex flex-column"
      v-if="
        web3Data.accounts &&
        web3Data.minters.some(
          (minter) => minter.toLowerCase() === web3Data.accounts[0]
        )
      "
    >

      <input @change="imageInput" type="file" id="file" placeholder="0" />

      <div class="flex nft">
        <img class="image" :src="toMint.image" alt="" />
        <div class="flex-column">
          <div class="name">Name: {{toMint.name}}</div>
          <div class="description">Description: {{toMint.description}}</div>
          <div class="link">Link: {{toMint.url}}</div>
        </div>
        <div class="form-attributes">
        Attributes
        <div
          class="form-attribute flex"
          v-for="(attribute, index) in toMint.attributes"
        >
          <div class="display">
            {{ attribute.display_type ? attribute.display_type : "Normal" }}
          </div>
          <div class="type">{{ attribute.trait_type }}</div>
          <div class="value">{{ attribute.value }}</div>
          <button class="primary-button" v-on:click="removeAttribute(index)">
            -
          </button>
          </div>
        </div>

      </div>

      <input type="text" v-model="toMint.name" placeholder="Name" />
      <input type="text" v-model="toMint.url" placeholder="NFT Link" />
      <input
        type="text"
        v-model="toMint.description"
        placeholder="Description"
      />


      <select name="types" id="types" v-model="toMint.selectedType">
        <option v-for="type in toMint.types" :value="type">
          {{ type.display }}
        </option>
      </select>
      <input type="text" placeholder="Trait Name" v-model="toMint.traitType" />

      <input
        v-if="
          toMint.selectedType && toMint.selectedType.display === 'Normal Text'
        "
        type="text"
        placeholder="Text"
        v-model="toMint.traitValue"
      />

      <input
        v-if="
          toMint.selectedType &&
          toMint.selectedType.display === 'Boost Percentage'
        "
        type="range"
        min="0"
        max="100"
        v-model="toMint.traitValue"
      />

      <input
        v-if="
          toMint.selectedType && toMint.selectedType.display === 'Boost Number'
        "
        type="number"
        placeholder="Number"
        v-model="toMint.traitValue"
      />
      <div
        v-if="
          toMint.selectedType &&
          (toMint.selectedType.display === 'Boost Number' ||
            toMint.selectedType.display === 'Boost Percentage')
        "
      >
        {{ toMint.traitValue }}
      </div>
      <input
        v-if="toMint.selectedType && toMint.selectedType.display === 'Number'"
        type="number"
        placeholder="Number"
        v-model="toMint.traitValue"
      />

      <button v-on:click="pushAttribute" class="primary-button">Add</button>
      <!-- <button class="mint primary-button" :disabled="!toMint.file" v-on:click="uploadFile">Upload</button> -->
      <button
        class="primary-button"
        :disabled="!toMint.file"
        v-on:click="mint()"
      >
        Mint
      </button>
      <button class="primary-button" v-on:click="grantMintersRole" >Grant role</button>
    </div>
  </div>
</template>

<script>
import Web3 from "web3";
import contract from "../assets/abi/contract";
import { address, owners, minters } from "../contractConfig";
export default {
  name: "IndexPage",

  asyncData({ $config: { jwt, coinMarketCap } }) {
    return { jwt, coinMarketCap };
  },
  data() {
    return {
      myNFTs: [],
      allNFTs: [],
      toMint: {
        file: null,
        image: null,
        name: "",
        description: "",
        attributes: [],
        traitType: null,
        traitValue: null,
        selectedType: {
          display: "Normal Text",
          value: "",
        },
        url: "",
        types: [
          {
            display: "Boost Number",
            value: "boost_number",
          },
          {
            display: "Boost Percentage",
            value: "boost_percentage",
          },
          {
            display: "Number",
            value: "number",
          },
          {
            display: "Normal Text",
            value: "",
          },
        ],
      },
      web3Data: {
        provider: null,
        web3: null,
        pending: false,
        contract: null,
        minters: minters,
        owners: owners,
        inUSD: 0,
        accounts: null,
        metamask: false,
        connected: false,
        networksDetail: {
          mumbai: {
            chainName: "Mumbai Testnet",
            chainId: Web3.utils.toHex(80001),
            nativeCurrency: { name: "MATIC", decimals: 18, symbol: "MATIC" },
            rpcUrls: ["https://rpc-mumbai.maticvigil.com"],
            blockExplorerUrls: ["https://polygonscan.com/"],
          },
          polygon: {
            chainName: "Polygon",
            chainId: Web3.utils.toHex(137),
            nativeCurrency: { name: "MATIC", decimals: 18, symbol: "MATIC" },
            rpcUrls: ["https://polygon-rpc.com"],
            blockExplorerUrls: ["https://mumbai.polygonscan.com/"],
          },
        },
        chainId: null,
      },
    };
  },
  mounted() {
    this.chromeConnect();
    console.log(this.jwt)
    console.log(this.$axios)
  },
  methods: {
    removeAttribute(index) {
      this.toMint.attributes.splice(index, index + 1);
    },
    pushAttribute() {
      let obj = {
        trait_type: this.toMint.traitType,
        value: this.toMint.traitValue,
      };

      if (this.toMint.selectedType.display === "Normal Text") {
        // delete obj["display_type"]
      } else {
        obj["display_type"] = this.toMint.selectedType.value;
      }
      this.toMint.attributes.push(obj);
      console.log(this.toMint.attributes);
    },
    imageInput(e) {
      let files = e.target.files || e.dataTransfer.files;
      if (!files.length) return;

      this.toMint.file = files[0];
      this.createImage(files[0]);
    },
    createImage(file) {
      let image = new Image();
      let reader = new FileReader();
      reader.onload = (e) => {
        this.toMint.image = e.target.result;
      };
      reader.readAsDataURL(file);
    },
    async uploadFile() {
      let file = this.toMint.file;
      let data = new FormData();
      data.append("file", file, file.name);
      data.append("pinataOptions", '{"cidVersion": 1}');
      data.append("pinataMetadata", '{"name": "Test"}');

      console.log(data);
      this.$axios.setToken(this.jwt, "Bearer");
      this.$axios.setHeader("Content-Type", `multipart/form-data`);
      let uri;
      await this.$axios
        .$post("https://api.pinata.cloud/pinning/pinFileToIPFS", data)
        .then((response) => {
          console.log(response);
          uri = `https://frodo.mypinata.cloud/ipfs/${response.IpfsHash}`;
          console.log(uri);
        });
      return uri;
    },
    async uploadJson(nft) {
      console.log(nft);
      console.log(this.jwt);
      this.$axios.setToken(this.jwt, "Bearer");
      this.$axios.setHeader("Content-Type", "application/json");
      let pin = {
        pinataOptions: {
          cidVersion: 1,
          wrapWithDirectory: true,
        },
        pinataMetadata: {
          name: nft.name + ".json",
        },
        pinataContent: nft,
      };
      let uri;
      await this.$axios
        .post("https://api.pinata.cloud/pinning/pinJSONToIPFS", pin)
        .then((response) => {
          console.log(response);
          uri = `https://frodo.mypinata.cloud/ipfs/${response.data.IpfsHash}`;
          console.log(uri);
        });
      return uri;
    },
    downloadJson() {
      console.log(this.nfts);
      this.nfts.forEach((nft, index) => {
        setTimeout(() => {
          this.download(nft, index + 1);
        }, index * 300);
      });
    },
    async bulkMint() {
      // let filtered = this.nfts.filter(nft => (this.getValue(nft.attributes, "Name") === this.minting.name));
      // console.log(filtered);
      let jsonList = [];
      let toMint = this.nfts.filter(
        (nft) => nft.id > this.minting.from && nft.id <= this.minting.to
      );
      console.log(toMint);
      let jsonCID = "QmW9YmK4wAGUEA5L2Vw91sisEzDsJBnFdQXeix2vq2rZ92";
      toMint.forEach((nft) => {
        let uri = `https://frodo.mypinata.cloud/ipfs/${jsonCID}/${nft.id}.json`;
        // let uri = `https://frodo.mypinata.cloud/ipfs/QmaHWFiazSVJnMxnFcpRxW6F1ALYTYQngDd78quVv8WqyD/2.json`
        jsonList.push(uri);
      });
      console.log(jsonList);
      console.log(toMint.length);
      let transaction = this.web3Data.contract.methods
        .bulkMint(toMint.length, jsonList)
        .send({
          from: this.web3Data.accounts[0],
          // gasPrice: this.web3Data.web3.utils.toHex(20e9),
          gas: Math.floor(15000000),
          gasLimit: Math.floor(15000000),
          value: 1e17,
        })
        .then((response) => {
          console.log(response);
        })
        // .on('confirmation', async (confirmationNumber, receipt) => {
        //     this.$router.go(0);
        // })
        .catch((error) => {
          console.log(error);
        });
    },
    async grantMintersRole() {
      this.web3Data.minters.forEach((minter) => {
        console.log("Granting minter role for > " + minter);
        this.web3Data.contract.methods
          .grantMintingRole(minter)
          .send({
            from: this.web3Data.accounts[0],
            gas: Math.floor(2000000),
            gasLimit: Math.floor(2000000),
          })
          .then((response) => {
            console.log(response);
          });
      });
    },
    getValue(list, trait) {
      return list.find((obj) => obj.trait_type === trait).value;
    },
    async mint() {
      let fileURI = await this.uploadFile();
      this.toMint.url = `https://google.com/search?q=${this.toMint.name}`
      let obj = {
        name: this.toMint.name,
        image: fileURI,
        external_url: this.toMint.url,
        attributes: [],
        description:  `${this.toMint.description} [Link](${this.toMint.url})`,
      };
      this.toMint.attributes.forEach((attribute) => {
        obj.attributes.push(attribute);
      });
      let uri = await this.uploadJson(obj);
      console.log(uri);
      let transaction = this.web3Data.contract.methods
        .mintHero(uri)
        .send({ from: this.web3Data.accounts[0] })
        .then((response) => {
          console.log(response);
        })
        // .on('confirmation', async (confirmationNumber, receipt) => {
        //     this.$router.go(0);
        // })
        .catch((error) => {
          console.log(error);
        });
    },
    async chromeConnect() {
      this.web3Data.web3 = new Web3(window.ethereum);
      await window.ethereum
        .send("eth_requestAccounts")
        .then((accounts) => {
          this.web3Data.connected = true;
          this.web3Data.accounts = accounts.result;
          window.ethereum.on("accountsChanged", (accounts) => {
            this.web3Data.accounts = accounts;
            this.$nextTick(() => {
              this.loadMyNFTs();
            });
          });
          window.ethereum.on("chainChanged", (chainId) => {
            this.web3Data.chainId = this.web3Data.web3.utils.toHex(chainId);
            console.log(this.web3Data.chainId);
            if (
              this.web3Data.chainId ===
              this.web3Data.networksDetail.mumbai.chainId
            ) {
              // this.loadAllNFTs();
            }
          });
          window.ethereum.on("disconnect", (code, reason) => {
            console.log(code, reason);
          });
          this.web3Data.contract = new this.web3Data.web3.eth.Contract(
            contract.abi,
            address
          );
          this.loadMyNFTs();
          console.log(this.web3Data.contract);

          this.web3Data.contract.methods
            .owner()
            .call()
            .then((response) => {
              this.web3Data.owner = response;
              console.log("Owner : " + response);
            });

          this.web3Data.web3.eth.getChainId().then((res) => {
            this.web3Data.chainId = res;
            this.web3Data.chainId = this.web3Data.web3.utils.toHex(
              this.web3Data.chainId
            );
            console.log("Chain Id: " + this.web3Data.chainId);
            if (
              this.web3Data.chainId ===
              this.web3Data.networksDetail.mumbai.chainId
            ) {
              // this.loadAllNFTs();
              // this.loadMyNFTs();
            }
          });
        })
        .catch((error) => {
          console.warn(error);
          this.$store.commit("alertMessage", error.message);
        });
    },
    download(object, index) {
      const a = document.createElement("a");
      a.href = URL.createObjectURL(
        new Blob([JSON.stringify(object, null, 2)], {
          type: "text/plain",
        })
      );
      a.setAttribute("download", `${index}.json`);
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
    },
    async loadMyNFTs() {
      this.myNFTs = [];
      let data = await this.web3Data.contract.methods
        .fetchMyNFTs()
        .call({ from: this.web3Data.accounts[0] })
        .catch((error) => {
          console.log(error);
        });
      const items = await Promise.all(
        data.map(async (i) => {
          let item = {
            tokenId: parseInt(i.tokenId),
            tokenURI: i.uri,
            owner: i.owner,
          };
          // const meta = await axios.get(item.tokenURI);
          // item['image'] = meta.data.image;
          // item['description'] = meta.data.description;
          this.myNFTs.push(item);
          return item;
        })
      );
      console.log(this.myNFTs);
    },
    async loadPartially(count, interval) {
      let total = 20;
      let from = 1;
      let myInterval = setInterval(() => {
        this.loadNFTs(from, from + count);
        from = from + count;
        if (from >= total) {
          clearInterval(myInterval);
        }
      }, interval);
    },
    async loadMyNFTs() {
                this.myNFTs = [];
                let data = await this.web3Data.contract.methods.fetchMyNFTs().call({from: this.web3Data.accounts[0]})
                    .catch(error => {
                        console.log(error)
                    });
                const items = await Promise.all(data.map(async i => {
                    let item = {
                        tokenId: parseInt(i.tokenId),
                        tokenURI: i.uri,
                        owner: i.owner,
                    };
                    // const meta = await axios.get(item.tokenURI);
                    // item['image'] = meta.data.image;
                    // item['description'] = meta.data.description;
                    this.myNFTs.push(item)
                    return item
                }));
                console.log(this.myNFTs)
            },
    async loadNFTs(from, to) {
      let data = await this.web3Data.contract.methods
        .fetchNFTs(from, to)
        .call({ from: this.web3Data.accounts[0] })
        .catch((error) => {
          console.log(error);
        });
      const items = await Promise.all(
        data.map(async (i) => {
          let item = {
            tokenId: parseInt(i.tokenId),
            tokenURI: i.uri,
            owner: i.owner,
          };
          // const meta = await axios.get(item.tokenURI);
          // item['image'] = meta.data.image;
          // item['description'] = meta.data.description;
          // !this.allNFTs.find(nft => nft.id === tokenId) && this.allNFTs.push(item)
          this.allNFTs.push(item);
          return item;
        })
      );
      console.log(this.allNFTs);
    },

    async loadAllNFTs() {
      this.allNFTs = [];
      this.myNFTs = [];
      let data = await this.web3Data.contract.methods
        .fetchAllNFTs()
        .call()
        .catch((error) => {
          console.log(error);
        });
      const items = await Promise.all(
        data.map(async (i) => {
          let item = {
            tokenId: parseInt(i.tokenId),
            tokenURI: i.uri,
            owner: i.owner,
          };
          // const meta = await axios.get(item.tokenURI);
          // item['image'] = meta.data.image;
          // item['category'] = meta.data.attributes[0].value;
          this.allNFTs.push(item);
          return item;
        })
      );
      console.log(this.allNFTs);
    },
  },
};
</script>

<style lang="scss" scoped>
.main {
  width: 90vw;
  justify-content: center;
  align-items: center;
  flex-direction: column  !important;
  .mint {
    margin: auto;
    flex-direction: column;
    width: 100%;
    height: 100vh;
    align-self: center;
    justify-content: center;
    align-items: center;
    .nft {
      width: 100%;
      justify-content: space-between;
      .image {
        @include equal(300px);
        border-radius: 10px;
        margin: 20px 0;
      }
      .flex-column {
        width: 30%;
      }
      .form-attributes {
        width: 30%;
        .form-attribute {
          justify-content: space-between;
          align-items: center;
          width: 100%;
        }
        .primary-button {
          width: 40px;
        }
      }
    }
    .primary-button {
      @include colorful;
      width: 300px;
      height: 40px;
      padding: 8px 0;
      /*font-size: 0.8rem;*/
      margin-top: 10px;
    }
    textarea {
      width: 300px;
    }
    input {
      margin: 5px 0;
      width: 300px!important;
    }
    select {
      margin: 5px 0;
      width: 300px;
    }
  }
}
</style>
