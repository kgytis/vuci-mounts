<template>
  <div class="cardContainer" @click="handleCardClick(reworkedName)" @dblclick="handleDblClick">
    <div :class="{ card: true }">
      <a-popconfirm v-if="type !== 'addFile'" :title="`Are you sure you want to delete ${name}?`" ok-text="Yes" cancel-text="No" @confirm="confirmDelete" placement="rightBottom">
        <img v-if="type !== 'addFile'" class="icon delIcon" src="/icons/delete.png" @click="handleDelIcon" />
      </a-popconfirm>
      <img v-if="type !== 'addFile' && type !== 'directory'" class="icon downIcon" src="/icons/download.png" @click="download"/>
      <img v-if="type === 'directory'" src="/icons/folder-br.png" />
      <img v-else-if="type === 'file'" src="/icons/document.png" />
      <a-popover v-else title="Add new file" trigger="click" v-model="addVisible" placement="bottom">
        <template #content>
          <a-form :model="formState" @submit.prevent>
            <a-form-item style="margin: 0.2rem">
              <a-input @blur="formState.name.isValid = true" v-model="formState.name.value" placeholder="File name..."/>
            </a-form-item>
            <p v-if="!formState.name.isValid" class="error">
              Input cannot be empty
            </p>
            <p v-if="fileExists" class="error">
              File with this name already exists
            </p>
            <a-form-item style="margin: 0">
              <a-button type="primary" @click="onSubmit">Create</a-button>
              <a-button @click="onCancel" :style="{ margin: '0 0.3rem' }">Cancel</a-button>
            </a-form-item>
          </a-form>
        </template>
        <img src="/icons/add-file.png" :style="{ cursor: 'pointer' }" />
      </a-popover>
      <p :id="reworkedName" class="name">{{ name }}</p>
    </div>
    <form v-if="currPath && type !== 'directory'" ref="form" method="POST" action="/download">
      <input v-show="false" type="text" :value="downloadPath" name="path" />
      <input v-show="false" type="text" :value="name" name="filename" />
    </form>
  </div>
</template>

<script>
export default {
  props: {
    name: {
      type: String,
      required: true
    },
    type: {
      type: String,
      required: true
    },
    actCounter: {
      type: Number
    },
    currPath: {
      type: Object
    },
    files: {
      type: Array
    }
  },
  data () {
    return {
      activeCard: false,
      formState: {
        name: {
          value: '',
          isValid: true
        }
      },
      addVisible: false,
      fileExists: false
    }
  },
  computed: {
    // sets path from where file should be downloaded
    downloadPath () {
      if (!this.currPath.updPath) {
        return `${this.currPath.path}/${this.name}`
      } else return `${this.currPath.updPath}/${this.name}`
    },
    // rework of file name to assign as ID (for style purposes / one click)
    reworkedName () {
      return this.name.replace('.', '_')
    }
  },
  methods: {
    handleCardClick (newName) {
      if (this.type !== 'addFile') {
        this.clearActivity()
        document.querySelector(`#${newName}`).classList.add('active')
      }
    },
    clearActivity () {
      const cards = document.querySelectorAll('.name')
      cards.forEach((card) => {
        card.classList.remove('active')
      })
    },
    handleDblClick () {
      if (!this.name.includes('.')) {
        this.clearActivity()
        this.$emit('doubleClick')
      }
    },
    confirmDelete () {
      this.$emit('deleteItem', this.name)
    },
    download () {
      this.clearActivity()
      this.$refs.form.submit()
    },
    handleDelIcon () {
      this.clearActivity()
    },
    validate () {
      if (this.formState.name.value.length === 0) {
        this.formState.name.isValid = false
        this.fileExists = false
        return
      } this.formState.name.isValid = true
    },
    // method to add new file/folder. With minimal validation. Two same files cannot be added and input field cannot be empty
    onSubmit () {
      this.validate()
      const name = this.formState.name.value
      if (name) {
        const foundVal = this.files.findIndex((file) => file.name === name)
        if (foundVal === -1) {
          this.$emit('addNew', name)
          this.addVisible = !this.addVisible
          this.formState.name.value = ''
          this.fileExists = false
        } else {
          this.fileExists = true
        }
      }
    },
    onCancel () {
      this.addVisible = !this.addVisible
      this.formState.name.value = ''
      this.fileExists = false
      this.formState.name.isValid = true
    }
  },
  watch: {
    // watcher to remove active class
    actCounter () {
      const cards = document.querySelectorAll('.name')
      cards.forEach((card) => {
        card.classList.remove('active')
      })
    }
  }
}
</script>

<style scoped>
img {
  width: 50px;
}

.name {
  padding: 0;
  margin: 0;
  width: 100%;
  text-align: center;
  border-radius: 10%;
}

.card {
  width: 70px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0;
  margin: 0 0.4rem;
  position: relative;
}

.cardContainer {
  display: inline-block;
}

.active {
  background-color: #1890ff;
  color: white;
}
.icon {
  width: 15px;
  position: absolute;
  right: 0.2rem;
  cursor: pointer;
}
.delIcon {
  right: 0.2rem;
}

.downIcon {
  left: 0.2rem;
}

.error {
  color: red;
  margin: 0;
  padding: 0;
}
</style>
