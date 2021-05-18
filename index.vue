<!--
  `<file-upload v-model="files" />` upload files to firebase
-->

<template>
  <span v-if="uploading"> {{text}}... {{uploading}} </span>
  <form class="file" v-else :style="'overflow: hidden; width: '+width">
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
        default: function() {
          return []
        },
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
      files: [],
    }),
    watch:{
      value:{
        handler(val) {
          this.files = this.removeDups(val)
        },
	immediate: true,
      },
      uploading(val) {
        this.$emit("uploading", val, this.meta)
      },
    },
    methods:{
      removeDups(files) {
        let unique = {}
        files.forEach(function(file, i) {
          if(!unique[file.hash]) {
            unique[file.hash] = i
          }
        })
        return Object.keys(unique).reduce((a, h) => {
          a.push(files[unique[h]])
          return a
        },[])
      },
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
              that.upload(btoa(spark.end(true)), fileN)
            }
          }
          {
            let start = currentChunk * chunkSize
            let end = start + chunkSize >= fileN.size ? fileN.size : start + chunkSize
            fileReader.readAsArrayBuffer(blobSlice.call(fileN, start, end))
          }
        })
        evt.target.removeAttribute('value')
        evt.target.value = ""
      },
      upload(hash, fileN) {
        let folder = this.folder || this.base64ToHex(hash)
        let path = folder + "/" + fileN.name
        let that = this
        let meta = { ...that.meta}
        let toUploadto = storage().ref().child(path)
        let uid = that.uid || auth().currentUser.uid || "anyone"
        meta[uid] = "UserID"
        toUploadto.getDownloadURL().then(url => {
          toUploadto.getMetadata().then(metadata => {
            var add  = {
              hash, url,
              name: fileN.name,
              ref: metadata.ref,
              type: metadata.contentType,
            }
            if (metadata.contentType.indexOf("image") !== -1) {
              add.image = true
            }
            setTimeout(() => {
              that.$emit("newFile", that.addMeta(add))
              that.$emit("input", that.removeDups([...that.files, that.addMeta(add)]))
            }, 0)
          }).catch(e => {
            console.error(e)
          })
        }).catch(function (e) {
          console.info(e)
          that.uploading += 1
          toUploadto.put(fileN, {customMetadata:meta}).then(async function(snapshot) { //upload
            console.log({snapshot})
            var add = {
              hash,
              metadata: snapshot.metadata,
              name: fileN.name,
              ref: snapshot.ref,
              type: snapshot.metadata.contentType,
              url: await snapshot.ref.getDownloadURL(),
            }
            if (snapshot.metadata.contentType.indexOf("image") !== -1) {
              add.image = add.url
            }
            that.$emit("newFile", that.addMeta(add))
            that.$emit("input", that.removeDups([...files, that.addMeta(add)]))
            that.uploading -= 1
            
          }).catch(function(e) {
            console.error(e)
            that.uploading -= 1
          })
        })
      },
      addMeta(file) {
        let returnFile = Object.keys(file).reduce((a, prop) => {
          if (typeof file[prop] === "string") {
            a[prop] = file[prop]
          }
          return a
        },{})
        if (file.hash) {
          fs.collection("files").doc(this.base64ToHex(file.hash)).set(returnFile, {merge: true})
          this.files = this.files.map(f => {
            if (f.hash === filePlus.md5Hash) {
              return filePlus
            }
            return f
          })
        }
        return returnFile
      },
    },
  }
</script>
