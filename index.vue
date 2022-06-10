<template>
  <div v-if="uploading"> {{text}}... {{uploading}} </div>
  <form v-else class="file">
    <label class="file-select">
      <slot>
        <div>{{label}}</div>
      </slot>
      <input ref="file" style="display: none" type="file" :accept="accept" name="files[]" class="custom-file-input" multiple @change="handleFileSelect" />
    </label>
  </form>
</template>
<script lang="ts">
import { fs, fbApp } from "@/db" // @/*": ["src/*"]
import { getStorage, ref, uploadBytes, getDownloadURL } from "firebase/storage"
import { doc, setDoc } from "firebase/firestore"
import { createMD5 } from "hash-wasm"

export default {
    emits: ["newFile", "input"],
    props: {
        value: {
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
        label: {
            default: "upload",
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
        selectFile(){
            let fileInputElement = this.$refs.file
            fileInputElement.click()
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
                const blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
                const chunkSize = 64 * 1024 * 1024
                const fileReader = new FileReader();
                let hasher = null

                function hashChunk(chunk) {
                    return new Promise((resolve, reject) => {
                        fileReader.onload = async(e) => {
                            const view = new Uint8Array(e.target.result)
                            hasher.update(view)
                            resolve(null)
                        }

                        fileReader.readAsArrayBuffer(chunk)
                    })
                }
                const readFile = async(file) => {
                    if (hasher) {
                        hasher.init()
                    } else {
                        hasher = await createMD5()
                    }

                    const chunkNumber = Math.floor(file.size / chunkSize)

                    for (let i = 0; i <= chunkNumber; i++) {
                        const chunk = file.slice(
                            chunkSize * i,
                            Math.min(chunkSize * (i + 1), file.size)
                        )
                        await hashChunk(chunk)
                    }

                    const hash = hasher.digest()
                    return Promise.resolve(hash)
                }
                fileReader.onload = function (e) {
                    readFile(fileN).then(hash => {
                        that.upload(hash, fileN)
                        const reader = new FileReader()
                        reader.addEventListener("load", function () {
                            // convert image file to base64 string
                            if (Array.isArray(that.value)) {
                                that.$emit("input", [...that.value, {preview: true, hash, url:reader.result, name: fileN.name, type: fileN.type}])
                            } else {
                                that.$emit("input", [{preview: true, hash, url:reader.result, name: fileN.name, type: fileN.type}])
                            }
                        }, false)
                        reader.readAsDataURL(fileN)
                    })
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
            let storage = getStorage(fbApp)
            const toUploadto = ref(storage, path)
            const uid = this.uid || "anyone"
            meta[uid] = "UserID"
            getDownloadURL(toUploadto).then(url => { // see if it up there
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
                uploadBytes(toUploadto, fileN, {customMetadata:meta}).then(async snapshot => { //upload
                    var add = {
                        hash,
                        metadata: snapshot.metadata,
                        name: fileN.name,
                        type: snapshot.metadata.contentType,
                        url: await getDownloadURL(snapshot.ref),
                    }
                    that.addMeta(add)
                    that.$emit("newFile", add)
                    if (Array.isArray(that.value)) {
                        that.$emit("input", [...that.value, add])
                    } else {
                        that.$emit("input", [add])
                    }
                    that.uploading -= 1
                }).catch(function(e) {
                    console.error(e)
                    that.uploading -= 1
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
                setDoc(doc(fs, "files",this.base64ToHex(file.hash)), returnFile, {merge: true})
            }
            return returnFile
        },
    },
}
</script>
