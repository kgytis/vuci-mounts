<template>
  <div>
    <a-button type="primary" @click="openModal">Create folder</a-button>
    <a-popover title="Add new file" trigger="click" v-model="addVisible" placement="bottom">
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
    </a-popover>
  </div>
</template>

<script>
export default {
  props: {
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
  methods: {
    openModal () {
      this.addVisible = !this.addVisible
    },
    validate () {
      if (this.formState.name.value.length === 0) {
        this.formState.name.isValid = false
        this.fileExists = false
        return
      }
      this.formState.name.isValid = true
    },
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
  }
}
</script>

<style scoped>
.error {
  color: red;
  margin: 0;
  padding: 0;
}
</style>
