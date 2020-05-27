<!--
  `<file-upload v-model="files" />` upload files to firebase
-->

<template>
  <input type="file" id="files" :accept="accept" name="files[]" multiple @change="handleFileSelect" />
  <span v-if="{{uploading}}">
    ... {{uploading}}
  </span>
</template>
<script>
  import {auth, fs, storage} from "@/db"
  import SparkMD5 from "spark-md5"

  export default {
    prop:{
      input:{
        type: Array,
        value: [],
      },
      log: {
        value: true,
      },
      meta: {
        value: {},
        type: Object,
      },
      uploading: {
        value: 0
      },
      accept: {
        value: "*"
      },
      bucket: String,
      folder: String,
    },
    computed: {
      _addMeta() {
        return this.addMeta(this.input)
      }
    },
    methods:{
      addMeta(files) {
        var that = this
        files.forEach(function(file,index){
          if (file.ref && !file.name) {
            file.name = file.ref.split("/").slice(-1)[0]
            storage().ref().child(file.ref).getDownloadURL().then( function (url) {
              file.url = url
              storage().ref().child(file.ref).getMetadata().then(function (metadata) {
                if (metadata.contentType.indexOf("image") !== -1) {
                  file.image = url
                }
                that.input[index] = file
                clearTimeout(that.timeout)
                that.timeout = setTimeout(function(){
                  var dedup = {}
                  that.files.forEach(function (file) {
                    if (!dedup[file.name]) {
                      dedup[file.name] = {}
                    }
                    Object.assign(dedup[file.name], file)
                  })
                  that.set("files",Object.keys(dedup).map(function (name) {
                    return dedup[name]
                  }))
                  that.set("files",that.files.map(function (file) {
                    return file
                  }))
                }, 0)
              })
            })
          }
        })
      },
    },
    mounted(){
      var that = this
      function handleFileSelect(evt) {
        var uid = that.uid || auth().currentUser.uid || "anyone"
        var db = fs
        Array.prototype.forEach.call(evt.target.files, function(fileN) {
          var fileReader = new FileReader()
          var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
          var chunkSize = 2097152
          var chunks = Math.ceil(fileN.size / chunkSize)
          var currentChunk = 0
          var spark = new SparkMD5.ArrayBuffer()

          function loadNext() {
            var start = currentChunk * chunkSize
            var end = start + chunkSize >= fileN.size ? fileN.size : start + chunkSize
            fileReader.readAsArrayBuffer(blobSlice.call(fileN, start, end))
          }

          function upload(hash) {
            var toUploadto = storage().ref().child(uid + "/" + hash + "/" + fileN.name)

            toUploadto.getDownloadURL().then( function (url) {
              firebase.storage().ref().getMetadata(uid + "/" + hash + "/" + fileN.name).then(function(metadata) {


            //  var add  = {name: fileN.name, url: url.downloadURL, type: url.metadata.contentType}
            //  if (url.metadata.contentType.indexOf("image") !== -1) {
            //    add.image = url.downloadURL
            //  }

                var add  = {name: fileN.name, url: url, type: metadata.contentType}
                if (metadata.contentType.indexOf("image") !== -1) {
                  add.image = url
                }
                var dedup = {}
                that.files.concat([add]).forEach(function (file) {
                  dedup[file.name] = file
                })
                that.set("files",Object.keys(dedup).map(function (name) {
                  return dedup[name]
                }))
                var meta = Object.assign({}, that.meta)
                meta[uid] = "userId"
                if (db && that.logUpload && hash) {
                  meta.path = uid + "/" + hash + "/" + fileN.name
                  meta.type = metadata.contentType
                  db.collection("files").doc(hash).set(meta, {merge: true})
                }
              })
            }).catch(function () {
              that.set("uploading", that.uploading + 1)
              var meta = Object.assign({}, that.meta)
              meta[uid] = "userId"
              toUploadto.put(fileN, {customMetadata:meta}).then(function(snapshot) { //upload
                var add  = {name: fileN.name, url: snapshot.downloadURL, type: snapshot.metadata.contentType}
                if (snapshot.metadata.contentType.indexOf("image") !== -1) {
                  add.image = snapshot.downloadURL
                }
                var dedup = {}
                that.files.concat([add]).forEach(function (file) {
                  dedup[file.name] = file
                })
                that.set("files",Object.keys(dedup).map(function (name) {
                  return dedup[name]
                }))

                toUploadto.getMetadata().then( function(x){console.log(x)} ).catch( function(e){console.log(e)} )
                if (db && that.logUpload && hash) {
                  meta.path = uid + "/" + hash + "/" + fileN.name
                  meta.type = snapshot.metadata.contentType
                  db.collection("files").doc(hash).set(meta, {merge: true})
                }
                that.uploading = that.uploading - 1
             }).catch(function(e) {
                console.log(e)
                that.uploading = that.uploading - 1)
              })
            })
          }

          fileReader.onload = function (e) {
            spark.append(e.target.result)
            currentChunk += 1
            if (currentChunk < chunks) {
              loadNext()
            } else {
              upload(spark.end())
            }
          }

          if (that.noMD5) {
            upload("")
          } else {
            loadNext()
          }
        })
        evt.target.removeAttribute('value')
        evt.target.value = ""
      }
    },
  })
</script>