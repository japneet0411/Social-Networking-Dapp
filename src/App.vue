<template>
  <div id="app">
    <div style="display: flex; flex-direction: column;">
      <!-- Upload Interface -->
      <div id="upload">
        <div v-if="this.$root.$data.loading === false">
          <h1 >SHARE YOUR MEMORIES</h1>
          <h4>Add post!</h4>

          <!-- Form for file choose, caption text and submission -->
          <form class="margin-sm align" @submit.stop.prevent="handleSubmit">
            <div class="border-style">
              <b-form-file plain @change="captureFile" size="60"/>
            </div>
            <b-form-textarea
              v-model="caption"
              placeholder="Post description"
              :rows="3"
              :max-rows="6"
              class="margin-xs border-style"
            />
            <b-button class="btn margin-xs" variant="dark" @click="handleOk">
              Upload
            </b-button>
          </form>
        </div>
        <div
          v-if="this.$root.$data.loading === true"
          style="margin-top: 10%; margin-bottom: 5%"
        >
          <img
            class="upload-load"
            src="https://media.giphy.com/media/2A6xoqXc9qML9gzBUE/giphy.gif"
          />
        </div>
      </div>
      <h1 id="wall">..YOUR WALL..</h1>
      <!-- Posts Interface -->
      <ul class="home-list">
        <li
          v-for="item in this.$root.$data.currentPosts"
          :key="item.key"
          :item="item"
        >
          <!-- Card UI for post's image & caption text -->
          <b-card border-variant="secondary" :img-src="item.src">
            <p class="home-card-text">
              {{ item.caption }}
            </p>
          </b-card>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import ipfs from "./contracts/ipfs";

export default {
  name: "App",
  // data variables
  data() {
    return {
      buffer: "",
      caption: "",
    };
  },
  methods: {
    /* used to catch chosen image &
     * convert it to ArrayBuffer.
     */
    captureFile(file) {
      const reader = new FileReader();
      if (typeof file !== "undefined") {
        reader.readAsArrayBuffer(file.target.files[0]);
        reader.onloadend = async () => {
          this.buffer = await this.convertToBuffer(reader.result);
        };
      } else this.buffer = "";
    },

    async convertToBuffer(reader) {
      return Buffer.from(reader);
    },

    onSubmit() {
      alert("Uploading on IPFS...");
      this.$root.loading = true;
      let imgHash;
      ipfs
        .add(this.buffer)
        .then((hashedImg) => {
          imgHash = hashedImg[0].hash;
          console.log("imgHash: " + imgHash);
          return this.convertToBuffer(this.caption);
        })
        .then((bufferDesc) =>
          ipfs.add(bufferDesc).then((hashedText) => hashedText[0].hash)
        )
        .then((textHash) => {
          this.$root.contract.methods
            .sendHash(imgHash, textHash)
            .send(
              { from: this.$root.currentAccount },
              (error, transactionHash) => {
                if (typeof transactionHash !== "undefined") {
                  alert("Storing on Ethereum...");
                  this.$root.contract.once(
                    "NewPost",
                    { from: this.$root.currentAccount },
                    () => {
                      this.$root.getPosts();
                      alert("Operation Finished! Refetching...");
                    }
                  );
                } else this.$root.loading = false;
              }
            );
        });
    },
    /**
     * validates if image & captions
     * are filled before submission.
     */
    handleOk() {
      if (!this.buffer || !this.caption) {
        alert("Please fill in the information.");
      } else {
        this.onSubmit();
      }
    },
  },
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  justify-content: center;
  color:black;
  /* background-color: #f8ceec;
  background-image: linear-gradient(315deg, #f8ceec 0%, #a88beb 74%); */
  background-image: url('./images/social.jpg');
  background-size:cover;
  background-color: #f8eae4;
  background-blend-mode: luminosity;
  height:auto;
 
}
#wall{
 margin-top: 30px;
 font-family: 'Smooch', cursive;
 text-align: center;
 color: azure;
 background-color: black;
}
.home-load {
  width: 50px;
  height: 50px;
}
.card img {
  object-fit: cover;
  height: 500px;
  width: 500px;
}
.card {
  text-align:left;
  width: auto;
  margin-bottom: 20px;
  border: 5px solid black;
  
}
.home-list {
  padding: 0;
  list-style: none;
}
.align{
margin-left: 130px;
  
}
.home-card-text {
  text-align: justify;
  margin-top: 10px;
}
#upload {
  font-family: 'Smooch', cursive;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  margin-bottom: 35%;
  margin-top: 5%;
 
 
}
.upload-load {
  width: 20px;
  height: 20px;
}
.margin-xs {
  margin-top: 3%;
}
.margin-sm {
  margin-top: 7%;
}
.border-style {
  border: 1px solid white;
  border-radius: 10px;

  width: 250px;
}
.btn{
  margin-right: 150px;
}
</style>
