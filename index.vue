<!--
  `<file-upload v-model="files" />` upload files to firebase
-->

<template>
  <input v-if="!uploading" type="file" :accept="accept" name="files[]" multiple @change="handleFileSelect" />
  <span v-else> {{text}}... {{uploading}} </span>
</template>
<script>
  import {auth, fs, storage} from "@/db"
  import SparkMD5 from "spark-md5"

  export default {
    props:{
      input:{
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
        default: "*"
      },
      bucket: String,
      folder: String,
      text: {
        default: "uploading"
      },
    },
    data: () => ({
      uploading:0,
      timeout: null,
      files:[]
    }),
    watch:{
      files(val) {
        this.$emit("input", val)
      },
      uploading(val) {
        this.$emit("uploading", val, this.meta)
      },
    },
    methods:{
      base64ToHex(str) {
        const raw = atob(str);
        let result = '';
        for (let i = 0; i < raw.length; i++) {
          const hex = raw.charCodeAt(i).toString(16);
          result += (hex.length === 2 ? hex : '0' + hex);
        }
        return result.toUpperCase();
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
        let meta = Object.assign({}, that.meta)
        let path = folder + "/" + fileN.name
        let that = this
        let toUploadto = storage().ref().child(path)
        let uid = that.uid || auth().currentUser.uid || "anyone"
        meta[uid] = "UserID"
        toUploadto.getDownloadURL().then( function (url) {
          toUploadto.getMetadata().then(function(metadata) {
            var add  = {
              hash, url,
              name: fileN.name,
              ref: metadata.ref,
              type: metadata.contentType,
            }
            if (metadata.contentType.indexOf("image") !== -1) {
              add.image = url // add for other types
            }
            that.files.push(that.addMeta(add))
          }).catch(e => {
            console.error(e)
          })
        }).catch(function (e) {
          console.log(e)
          that.uploading += 1
          toUploadto.put(fileN, {customMetadata:meta}).then(function(snapshot) { //upload
            var add  = {
              hash,
              metadata: snapshot.metadata,
              name: fileN.name,
              ref: snapshot.ref,
              type: snapshot.metadata.contentType,
              url: snapshot.downloadURL,
            }
            if (snapshot.metadata.contentType.indexOf("image") !== -1) {
              add.image = snapshot.downloadURL
            }
            that.files.push(that.addMeta(add))
            that.uploading = that.uploading - 1
          }).catch(function(e) {
            console.error(e)
            that.uploading = that.uploading - 1
          })
        })
      },
      addMeta(file) {
        let that = this
        let returnFile = Object.keys(file).reduce((a, prop) => {
          if (typeof file[prop] === "string") {
            a[prop] = file[prop]
          }
          return a
        },{})

        if (file.ref) {
          file.ref.getDownloadURL().then(function (url) {
            returnFile.url = url
            file.ref.getMetadata().then(function (metadata) {
              if (metadata.md5Hash !== file.hash) {
                throw("Service's hash does not match local hash")
              }
              if (metadata.contentType.indexOf("image") !== -1) {
                returnFile.image = url
              }
              let vueable = Object.keys(metadata).reduce((a, prop) => {
                if (typeof metadata[prop] === "string" || prop === "customMetadata") {
                  a[prop] = metadata[prop]
                }
                return a
              },{})
              let filePlus = Object.assign({}, returnFile, vueable)
              fs.collection("files").doc(that.base64ToHex(file.hash)).set(filePlus, {merge: true})
              that.files = that.files.map(f => {
                if (f.hash === filePlus.md5Hash) {
                  return filePlus
                }
                return f
              })
            }).catch(e => {
              console.error(e)
            })
          }).catch(e => {
            console.error(e)
          })
        }
        return returnFile
      },
    },
  }
</script>
