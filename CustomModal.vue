<template>
  <a-modal :visible="toggleModal" footer @cancel="closeModal" @ok="closeModal">
    <div class="header">
      <p v-if="!updPath">{{ path }}</p>
      <p v-else>{{ updPath }}</p>
      <a-button @click="handleBack" :disabled="path === updPath || !updPath"
        >Back</a-button
      >
      <a-upload
        action="/upload"
        :data="{ path: uploadPath }"
        @change="handleUpload"
        :beforeUpload="beforeUpload"
        :showUploadList="false"
      >
        <a-button :style="{ margin: '1rem 0.3rem' }"
          ><a-icon type="upload" />Upload</a-button
        >
      </a-upload>
    </div>
    <div @click="handleBodyClick" id="modalBody">
      <file-explorer
        v-for="(item, i) in files"
        :key="i"
        :name="item.name"
        :type="item.type"
        :currPath="{ updPath, path }"
        :actCounter="actCounter"
        @doubleClick="handleDblClick({ name: item.name, type: item.type })"
        @deleteItem="deleteItem"
      />
      <!-- Additional icon that add new file would be displayed -->
      <file-explorer
        name="Add new"
        type="addFile"
        :files="files"
        :actCounter="actCounter"
        @addNew="add"
      />
      <a-modal :visible="uplVis" @ok="overrideFile(true)" @cancel="overrideFile(false)">
        <h3>
          File with the same name exists. Would you like to overwrite
          <u>{{ fileName }}</u
          >?
        </h3>
      </a-modal>
    </div>
  </a-modal>
</template>

<script>
import FileExplorer from './FileExplorer.vue'
export default {
  components: {
    FileExplorer
  },
  props: {
    toggleModal: {
      type: Boolean,
      required: true
    },
    path: {
      type: String,
      required: true
    },
    fileCounter: {
      type: Number,
      required: true
    }
  },
  data () {
    return {
      files: [],
      actCounter: 0,
      updPath: '',
      size: 10000,
      fileName: '',
      uplVis: false,
      resolveGlobal: undefined,
      rejectGlobal: undefined
    }
  },
  computed: {
    // path where file should be uploaded
    uploadPath () {
      if (!this.updPath) {
        return `${this.path}/${this.fileName}`
      } else return `${this.updPath}/${this.fileName}`
    }
  },
  methods: {
    closeModal () {
      this.$emit('closeModal')
      this.updPath = ''
    },
    // promise resolve/reject whether duplicate file should be overriden
    overrideFile (value) {
      if (value) {
        this.resolveGlobal(true)
      } else {
        this.rejectGlobal(undefined)
      }
      this.uplVis = false
    },
    handleBodyClick (e) {
      // to turn off active class from element
      if (e.path[0].id === 'modalBody') {
        this.actCounter++
      }
    },
    handleDblClick (props) {
      // updates path when double clicked on folder
      const { name } = props
      if (this.updPath === '') {
        this.updPath = this.path + '/' + name
      } else {
        this.updPath = this.updPath + '/' + name
      }
    },
    handleBack () {
      // back is allowed if only paths are not equal
      if (this.updPath !== this.path) {
        const index = this.updPath.lastIndexOf('/')
        this.updPath = this.updPath.slice(0, index)
      }
    },
    async beforeUpload (file) {
      this.fileName = file.name
      const fileSearch = this.files.find((file) => {
        return file.name === this.fileName
      })
      // creates a promise which will be resolved when buttons of modal is clicked, this prmose created only of duplicate file is found
      if (fileSearch) {
        this.uplVis = true
        const upl = await new Promise((resolve, reject) => {
          this.resolveGlobal = resolve
          this.rejectGlobal = reject
        })
        if (upl) return upl
      }
    },
    handleUpload (info) {
      const file = info.file
      const status = file.status
      if (file.size > this.size) {
        this.$message.error(`File size exceeds ${this.size}B limit.`)
        return
      }
      if (status === 'uploading' || status === 'removed') return
      if (status !== 'done') {
        this.$message.error(`upload '${this.fileName}' failed.`)
        return
      }
      if (status === 'done') {
        this.$message.success(`File '${this.fileName}' uploaded.`)
        this.$emit('uploadedFile')
      }
    },
    // method reponsible to get all information of files in designated and assign it to global variable
    async getFiles (path) {
      if (this.path === '') return
      // file reset
      this.files = []
      const data = await this.$rpc.ubus('file', 'list', { path: path })
      return data.entries
    },
    // sets files which were retrieved from router / usb
    async setFiles (data) {
      this.files = await data
    },
    getPath () {
      let path
      this.updPath ? (path = this.updPath) : (path = this.path)
      return path
    },
    add (name) {
      const path = this.getPath()
      this.$emit('addNew', { name: name, path: path })
    },
    deleteItem (name) {
      const path = this.getPath()
      this.$emit('deleteItem', { name: name, path: path })
    }
  },
  watch: {
    path () {
      this.setFiles(this.getFiles(this.path))
    },
    updPath () {
      this.setFiles(this.getFiles(this.updPath))
    },
    fileCounter () {
      this.updPath ? this.setFiles(this.getFiles(this.updPath)) : this.setFiles(this.getFiles(this.path))
    }
  }
}
</script>

<style scoped>
p {
  margin: 0;
}
</style>
