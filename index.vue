<!--
  `<file-upload v-model="files" />` upload files to firebase
-->

<template>
  <input v-if="!uploading" type="file" id="files" :accept="accept" name="files[]" multiple @change="handleFileSelect" />
  <span v-else>
    ... {{uploading}}
  </span>
</template>
<script>
  import {auth, fs, storage} from "@/db"
  import SparkMD5 from "spark-md5"

  export default {
    props:{
      input:{
        type: Array,
        default: function() {
          return []
        },
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
    },
    data: () => ({
      uploading:0,
      timeout: null,
      files:[]
    }),
    watch:{
      files(val) {
        this.$emit("input",val)
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
          var fileReader = new FileReader()
          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
          var chunkSize = 2097152
          var chunks = Math.ceil(fileN.size / chunkSize)
          var currentChunk = 0
          var spark = new SparkMD5.ArrayBuffer()
          console.log("debugger")
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
        let that = this
        let uid = that.uid || auth().currentUser.uid || "anyone"
        let folder = this.folder || this.base64ToHex(hash) 
        let path = folder + "/" + uid + "/" + fileN.name
        let toUploadto = storage().ref().child(path)
        let meta = Object.assign({}, that.meta)
        meta[uid] = "UserID"


        toUploadto.getDownloadURL().then( function (url) {
          firebase.storage().ref().getMetadata(path).then(function(metadata) {
            console.log(metadata)
            var add  = {
              name: fileN.name,
              url,
              type: metadata.contentType,
              ref: metadata.ref,
              hash,
            }
            if (metadata.contentType.indexOf("image") !== -1) {
              add.image = url
            }
            that.files.push(that.addMeta(add))
          }).catch(e => {
            console.error(e)
          })
        }).catch(function () {
          that.uploading += 1
          toUploadto.put(fileN, {customMetadata:meta}).then(function(snapshot) { //upload
            var add  = {
              name: fileN.name,
              url: snapshot.downloadURL,
              type: snapshot.metadata.contentType,
              ref: snapshot.ref,
              hash,
              metadata: snapshot.metadata
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
