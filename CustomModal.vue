<template>
  <a-modal :visible="toggleModal" footer @cancel="closeModal">
    <a-spin v-if="uploading" tip="Uploading..."/>
    <div class="header" @click="removeActiveClass">
      <p v-if="!updPath" class="">Path: {{ path }}</p>
      <p v-else>Path: {{ updPath }}</p>
      <!-- <div>
      <a-button @click="handleBack" v-show="!(path === updPath || !updPath)" :disabled="path === updPath || !updPath">Back</a-button>
      </div> -->
    </div>
    <div @click="handleBodyClick" id="modalBody" :style="{'max-height': '500px', 'overflow': 'auto'}">
      <file-explorer name=".." type="back" @back="handleBack" v-if="updPath !== path && updPath !== ''"/>
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
    </div>
    <div class="actions" @click="removeActiveClass">
      <add-new @addNew="add" :files="files"></add-new>
      <a-upload action="/upload" :data="{ path: uploadPath }" @change="handleUpload" :beforeUpload="beforeUpload" :showUploadList="false">
        <a-button :style="{ margin: '1rem 0.3rem' }"><a-icon type="upload"/>Upload file</a-button>
      </a-upload>
      </div>
      <a-modal :visible="uplVis" @ok="overrideFile(true)" @cancel="overrideFile(false)">
        <h3>
          File with the same name exists. Would you like to overwrite
          <u>{{ fileName }}</u>?
        </h3>
      </a-modal>
  </a-modal>
</template>

<script>
import FileExplorer from './FileExplorer.vue'
import AddNew from './AddNew.vue'
export default {
  components: {
    FileExplorer,
    AddNew
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
      size: 100000000,
      fileName: '',
      uplVis: false,
      resolveGlobal: undefined,
      rejectGlobal: undefined,
      uploading: false
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
      this.$emit('closeModal', true)
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
      // if (status === 'uploading' || status === 'removed') return
      if (status === 'uploading') {
        this.uploading = true
        return
      }
      if (status !== 'done') {
        this.uploading = false
        this.$message.error(`upload '${this.fileName}' failed.`)
        return
      }
      if (status === 'done') {
        this.uploading = false
        this.$message.success(`File '${this.fileName}' uploaded.`)
        this.$emit('uploadedFile')
      }
    },
    // method reponsible to get all information of files in designated and assign it to global variable
    async getFiles (path) {
      if (this.path === '') return
      const data = await this.$rpc.ubus('file', 'list', { path: path })
      // file reset
      this.files = []
      // sets files which were retrieved from router / usb
      this.files = await data.entries
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
    },
    removeActiveClass () {
      const cards = document.querySelectorAll('.name')
      cards.forEach((card) => {
        card.classList.remove('active')
      })
    }
  },
  watch: {
    async path () {
      await this.getFiles(this.path)
    },
    async updPath (curr) {
      curr && await this.getFiles(this.updPath)
    },
    async fileCounter () {
      if (this.updPath) {
        await this.getFiles(this.updPath)
      } else {
        await this.getFiles(this.path)
      }
    }
  }
}
</script>

<style scoped>
p {
  margin: 0;
}

#modalBody {
  display: grid;
  gap: 0.2rem;
  border-top: 1px solid rgba(0, 0, 0, 0.138);
  padding-top: 0.7rem;
}

.actions {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  margin-top: 0.3rem;
}

.header {
  height: 3rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1rem;
}
</style>
