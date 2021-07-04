# v-fb-upload
## vue firebase upload
Upload files to firebase storage
built-in vue
## use

in template
```
  <fb-upload ref="upload" v-model="files" :meta="meta" :uid="user.uid" @uploading="updateUploading">
    <v-btn @click="$refs.upload.selectFile()" >Upload helpful pictures or videos ...</v-btn> 
  </fb-upload>
```

in script
```
  import fbUpload from "v-fb-upload"
  
  export default {
    components: {
      fbUpload,
    },
  }

```

## Features

* local integrity check of uploaded files
* does not include in upload if already there
* saves custom metadata about the uploaded file (can be used for access permissions (I do))
* if the folder is not set uploads to a unique folder for each hash of a file
* async upload all files
* exposes how many files are still uploading
