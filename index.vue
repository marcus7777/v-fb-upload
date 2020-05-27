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
        default: [],
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
      handleFileSelect(evt) {
        let that = this
        Array.prototype.forEach.call(evt.target.files, function(fileN) {
          var fileReader = new FileReader()
          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
          var chunkSize = 2097152
          var chunks = Math.ceil(fileN.size / chunkSize)
          var currentChunk = 0
          var spark = new SparkMD5.ArrayBuffer()

          fileReader.onload = function (e) {
            spark.append(e.target.result)
            currentChunk += 1
            if (currentChunk < chunks) {
              let start = currentChunk * chunkSize
              let end = start + chunkSize >= fileN.size ? fileN.size : start + chunkSize
              fileReader.readAsArrayBuffer(blobSlice.call(fileN, start, end))
            } else {
              that.upload(spark.end(),fileN,e.target.result)
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
      upload(hash, fileN, fileAsStr) {
        let that = this
        let uid = that.uid || auth().currentUser.uid || "anyone"
        var toUploadto = storage().ref().child(uid + "/" + hash + "/" + fileN.name)

        toUploadto.getDownloadURL().then( function (url) {
          firebase.storage().ref().getMetadata(uid + "/" + hash + "/" + fileN.name).then(function(metadata) {
            console.log(metadata)
            //  var add  = {name: fileN.name, url: url.downloadURL, type: url.metadata.contentType}
            //  if (url.metadata.contentType.indexOf("image") !== -1) {
            //    add.image = url.downloadURL
            //  }

            var add  = {
              name: fileN.name,
              url,
              type: metadata.contentType,
              fileAsStr,
              ref: metadata.ref
            }
            if (metadata.contentType.indexOf("image") !== -1) {
              add.image = url
            }
            that.files.push(that.addMeta(add))
            var meta = Object.assign({}, that.meta)
            meta[uid] = "UserID"
            if (fs && hash) {
              meta.path = uid + "/" + hash + "/" + fileN.name
              meta.type = metadata.contentType
              fs.collection("files").doc(hash).set(meta, {merge: true})
            }
          })
        }).catch(function (e) {
          console.log(e)
          that.uploading += 1
          var meta = Object.assign({}, that.meta)
          meta[uid] = "UserID"
          toUploadto.put(fileN, {customMetadata:meta}).then(function(snapshot) { //upload
            var add  = {
              name: fileN.name,
              url: snapshot.downloadURL,
              type: snapshot.metadata.contentType,
              ref: snapshot.ref,
              fileAsStr,
            }
            if (snapshot.metadata.contentType.indexOf("image") !== -1) {
              add.image = snapshot.downloadURL
            }

            that.files.push(that.addMeta(add))

            toUploadto.getMetadata().then( function(x){console.log(x)} ).catch( function(e){console.log(e)} )
            if (fs && hash) {
              meta.path = uid + "/" + hash + "/" + fileN.name
              meta.type = snapshot.metadata.contentType
              fs.collection("files").doc(hash).set(meta, {merge: true})
            }
            that.uploading = that.uploading - 1
          }).catch(function(e) {
            console.error(e)
            that.uploading = that.uploading - 1
          })
        })
      },
      addMeta(file) {
        if (file.ref) {
          if (!file.name) {
            file.name = file.ref.split("/").slice(-1)[0]
          }
          storage().ref().child(file.ref).getDownloadURL().then( function (url) {
            file.url = url
            storage().ref().child(file.ref).getMetadata().then(function (metadata) {
              if (metadata.contentType.indexOf("image") !== -1) {
                file.image = url
              }
            })
          })
        }
        return file
      },
    },
  }
</script>
