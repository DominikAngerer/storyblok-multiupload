<template>
  <div class="uk-clearfix">
    <input class="uk-hidden" type="file" multiple="multiple" @change="changeFiles">
    
    <div class="uk-flex uk-margin-small-bottom" v-for="(key, item) in model.files">
      <img class="image" :src="item.url.replace('a.storyblok.com', 'img2.storyblok.com/160x90/filters:fill(auto,0)')">
      <input type="text" class="uk-form-small uk-flex-item-auto" v-model=item.url>
      <button @click="removeFile(key)" class="uk-button uk-button-small uk-margin-small-left"><i class="uk-icon-close"></i></button>
    </div>

    <div class="multiupload__droparea uk-placeholder" v-on:dragover="$event.preventDefault()" v-on:drop="changeFiles">
      <button @click="removeFileFromUpload(key)" v-for="(key, item) in model.to_upload" class="uk-button uk-button-small multiupload__droparea-item">{{item.name}} <i class="uk-icon uk-icon-close"></i></button>
      <span v-if="model.to_upload.length == 0">Drop Files Here</span>
    </div>
    <button class="uk-button uk-button-small uk-margin-small-top uk-float-right" v-bind:disabled="model.progress >= 0" @click="startUpload">Start Upload</button>
  </div>

</template>

<script>
export default {
  mixins: [window.Storyblok.plugin],
  methods: {
    initWith: function() {
      return {
        plugin: 'multiupload',
        files: [],
        progress: -1,
        to_upload: []
      }
    },
    startUpload: function() {
      this.model.progress = 0
      this.uploadNext()
    },
    uploadNext: function() {
      if (this.model.to_upload.length > this.model.progress) {
        this.uploadFile(this.model.to_upload[this.model.progress])
      } else {
        this.model.progress = -1
        this.model.to_upload = []
      }
    },
    uploadFile: function(file) {
      this.api(`spaces/${this.spaceId}/assets`)
        .save({ filename: file.name }).then(res => {
        let form_data = new FormData()
        let xhr = new XMLHttpRequest()

        Object.keys(res.data.fields).forEach(key => {
          form_data.set(key, res.data.fields[key])
        })

        form_data.set('file', file)
        xhr.onreadystatechange = () => {
          if (xhr.readyState === 4) {
            if (xhr.status == 204 || xhr.status == 200 || xhr.status == 201) {
              this.model.files.push({ url: res.data.pretty_url })
              this.model.progress++
              setTimeout(() => { this.uploadNext() }, 0)
            } else {
              console.log('Multiupload: Error during upload with status: ', xhr.status);
            }
          }
        }
        xhr.open('POST', res.data.post_url)
        xhr.send(form_data)
      })
    },
    changeFiles: function(event) {
      event.preventDefault()
      let files = event.target.files || event.dataTransfer.files
      this.model.to_upload = Object.keys(files).map(key => {
        return files[key]
      })
    },
    removeFileFromUpload: function(index) {
      this.model.to_upload.splice(index, 1)
    },
    removeFile: function(index) {
      this.model.files.splice(index, 1)
    }
  },
  watch: {
    'model': {
      handler: function (value) {
        this.$emit('changed-model', value)
      },
      deep: true
    }
  }
}
</script>

<style>
  .multiupload__droparea {
    padding: 20px;
    margin-bottom: 0px;
    margin-top: 0px;
  }
  .multiupload__droparea button {
    margin-right: 5px;
    margin-top: 5px;
  }
</style>
