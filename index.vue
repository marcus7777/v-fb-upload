<!--
  `<file-upload v-model="files" />` upload files to firebase
-->

<template>
  <div v-if="uploading"> {{text}}... {{uploading}} </div>
  <form v-else class="file" :style="'overflow: hidden; width: '+width">
    <input type="file" :accept="accept" name="files[]" :data-text="label" class="custom-file-input" multiple @change="handleFileSelect" />
  </form>
</template>
<style>
  .custom-file-input::-webkit-file-upload-button {
    visibility: hidden;
  }
  .custom-file-input::before {
    content: attr(data-text);
    display: inline-block;
    white-space: nowrap;
    cursor: pointer;
  }
</style>
<script>
  import {auth, fs, storage} from "@/db"
  import SparkMD5 from "spark-md5"

  export default {
    props:{
      value:{
        type: Array,
      },
      meta: {
        default: function() {
          return {}
        },
        type: Object,
      },
      uid: String,
      accept: {
        default: "*",
        type: String,
      },
      bucket: String,
      folder: String,
      text: {
        default: "uploading",
        type: String,
      },
      label:{
        default: "upload",
        type: String,
      },
      width:{
        default: "50px",
        type: String,
      },
    },
    data: () => ({
      uploading: 0,
      timeout: null,
    }),
    watch:{
      uploading(val) {
        this.$emit("uploading", val, this.meta)
      },
    },
    methods:{
      base64ToHex(str) {
        const raw = atob(str)
        let result = ''
        for (let i = 0; i < raw.length; i++) {
          const hex = raw.charCodeAt(i).toString(16)
          result += (hex.length === 2 ? hex : '0' + hex)
        }
        return result.toUpperCase()
      },
      handleFileSelect(evt) {
        let that = this
        Array.prototype.forEach.call(evt.target.files, function(fileN) {
          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
          var chunkSize = 2097152
          var chunks = Math.ceil(fileN.size / chunkSize)
          var currentChunk = 0
          var fileReader = new FileReader()
          var spark = new SparkMD5.ArrayBuffer()
          fileReader.onload = function (e) {
            spark.append(e.target.result)
            currentChunk += 1
            if (currentChunk < chunks) {
              let start = currentChunk * chunkSize
              let end = start + chunkSize >= fileN.size ? fileN.size : start + chunkSize
              fileReader.readAsArrayBuffer(blobSlice.call(fileN, start, end))
            } else {
              const hash = btoa(spark.end(true)) 
              that.upload(hash, fileN)
              const reader = new FileReader()
              reader.addEventListener("load", function () {
                // convert image file to base64 string
                that.$emit("input", [...that.value, {preview: true, hash, url:reader.result, name: fileN.name, type: fileN.type}])
              }, false)
              reader.readAsDataURL(fileN)
            }
          }
          {
            let end = chunkSize >= fileN.size ? fileN.size : chunkSize
            fileReader.readAsArrayBuffer(blobSlice.call(fileN, 0, end))
          }
        })
        evt.target.removeAttribute('value')
        evt.target.value = ""
      },
      upload(hash, fileN) {
        const folder = this.folder || this.base64ToHex(hash)
        const path = folder + "/" + fileN.name
        let meta = { ...this.meta}
        const toUploadto = storage().ref().child(path)
        const uid = this.uid || auth().currentUser.uid || "anyone"
        meta[uid] = "UserID"
        toUploadto.getDownloadURL().then(url => { // see if it up there
          var add = {
            hash, url,
            name: fileN.name,
            type: fileN.type,
          }
          this.$emit("newFile", add)
          this.addMeta(add)
          this.$emit("input", [...this.value, add])
        }).catch(e => {
          console.info(e)
          this.uploading += 1
          let that = this
          toUploadto.put(fileN, {customMetadata:meta}).then(async snapshot => { //upload
            var add = {
              hash,
              metadata: snapshot.metadata,
              name: fileN.name,
              type: snapshot.metadata.contentType,
              url: await snapshot.ref.getDownloadURL(),
            }
            this.addMeta(add)
            this.$emit("newFile", add)
            this.$emit("input", [...that.value, add])
            this.uploading -= 1
          }).catch(function(e) {
            console.error(e)
            this.uploading -= 1
          })
        })
      },
      addMeta(file) {
        let returnFile = Object.keys(file).reduce((a, prop) => {
          if (typeof file[prop] === "string" || typeof file[prop] === "boolean") {
            a[prop] = file[prop]
          }
          return a
        }, {})
        if (file.hash && fs) {
          fs.collection("files").doc(this.base64ToHex(file.hash)).set(returnFile, {merge: true})
        }
        return returnFile
      },
    },
  }
</script>
