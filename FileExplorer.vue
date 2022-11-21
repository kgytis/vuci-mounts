<template>
  <div class="cardContainer" @click="handleCardClick(reworkedName)" @dblclick="handleDblClick">
    <div class="card">
      <div class="folders">
        <img v-if="type === 'back'" src="/icons/folder-br.png" class="back" @click="back"/>
        <img v-else-if="type === 'directory'" src="/icons/folder-br.png" />
        <img v-else-if="type === 'file'" src="/icons/document.png" />
        <p :id="reworkedName" class="name">{{ name }}</p>
      </div>
      <div class="actions" v-if="type !== 'back'">
        <img v-if="type !== 'directory'" class="icon downIcon" src="/icons/download.png" @click="download"/>
        <a-popconfirm :title="`Are you sure you want to delete ${name}?`" ok-text="Yes" cancel-text="No" @confirm="confirmDelete" placement="rightBottom">
          <img class="icon delIcon" src="/icons/delete.png" @click="handleDelIcon" />
        </a-popconfirm>
      </div>
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
  computed: {
    // sets path from where file should be downloaded
    downloadPath () {
      if (!this.currPath.updPath) {
        return `${this.currPath.path}/${this.name}`
      } else return `${this.currPath.updPath}/${this.name}`
    },
    // rework of file name to assign as ID (for style purposes / one click)
    reworkedName () {
      return this.name.replace('.', '_').replace(/\s/g, '_')
    }
  },
  methods: {
    handleCardClick (newName) {
      if (this.type !== 'back') {
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
    back () {
      this.$emit('back')
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
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 0.3rem 0;
  border: 1px solid rgba(0, 0, 0, 0.138);
  box-shadow: rgba(99, 99, 99, 0.2) 0px 2px 8px 0px;
}

.cardContainer {
  width: 100%;
}

.active {
  background-color: #1890ff;
  color: white;
}
.icon {
  cursor: pointer;
  width: 1.5rem;
  margin: 0 0.6rem;
}

.folders {
  width: 80px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  margin-left: 0.3rem;
  padding: 0.3rem 0;
}

.back{
  cursor: pointer
}

</style>
