<template>
  <div>
    <a-table
      v-if="!error"
      :loading="mounts.length === 0"
      :columns="columns"
      :data-source="mounts"
      defaultExpandAllRows
    >
      <template #memory="record">
        <span>
          <vuci-progress-bar
            :value="formatPercentage(record)"
            :show-info="false"
            :title="`Used ${record.used}/${record.available}`"
          />
        </span>
      </template>
      <template #actions="record">
        <span>
          <a-button
            key="file-explorer"
            style="padding: 0rem 0.2rem"
            @click="handleModal(record.mountpoint)"
          >
            <img src="/icons/folder-icon.png" />
          </a-button>
        </span>
      </template>
    </a-table>
    <custom-modal
      v-if="!error"
      @closeModal="handleModal"
      :toggleModal="toggleModal"
      :path="mountpoint"
      :fileCounter="fileCounter"
      @addNew="handleAddition"
      @deleteItem="handleDelete"
      @uploadedFile="fileCounter++"
    ></custom-modal>
    <div v-else><h1>No mounts detected</h1></div>
  </div>
</template>

<script>
import VuciProgressBar from '../components/VuciProgressBar.vue'
import CustomModal from './components/CustomModal.vue'
export default {
  components: {
    CustomModal,
    VuciProgressBar
  },
  // timers: {
  //   update: { time: 1000, autostart: true, immediate: true, repeat: true }
  // },
  data () {
    return {
      mounts: [],
      columns: [
        { dataIndex: 'mountpoint', key: 'mountpoint', title: 'Mountpoint' },
        { dataIndex: 'dev', key: 'dev', title: 'Device' },
        { dataIndex: 'fs', key: 'fs', title: 'Filesystem' },
        {
          dataIndex: 'used_percentage',
          key: 'used_percentage',
          title: 'Used memory percentage',
          scopedSlots: { customRender: 'memory' }
        },
        { key: 'actions', title: '', scopedSlots: { customRender: 'actions' } }
      ],
      toggleModal: false,
      mountpoint: '',
      fileCounter: 0,
      memoryUsed: 0,
      memoryAvail: 0,
      error: false
    }
  },
  methods: {
    // additional formatting for progress bar
    formatPercentage (data) {
      const index = data.used_percentage.search('%')
      const percentage = Number(data.used_percentage.slice(0, index))
      return percentage
    },
    async getMountsData () {
      // command to extract mounts primary data from router
      const data = await this.$rpc.ubus('file', 'exec', {
        command: '/bin/fmt-usb-msd.sh',
        params: ['unmountable']
      })
      const keys = Object.keys(JSON.parse(data.stdout))
      if (keys.length === 0) {
        this.error = true
        return
      }
      this.error = false
      // this approach only if one usb port present
      // code should be a bit changed if more than one usb port present
      const convertedData = JSON.parse(data.stdout)[keys[0]]
      return [convertedData]
    },
    async setMountsData () {
      const data = await this.getMountsData()
      if (!data) {
        this.error = true
        return
      }
      this.error = false
      const convertedData = data.map((el, i) => {
        return {
          key: `mount-${i}`,
          mountpoint: el.mountpoint,
          dev: el.dev,
          used_percentage: {
            used: el.used,
            available: el.available,
            used_percentage: el.used_percentage
          },
          fs: el.fs
        }
      })
      this.mounts = convertedData
    },
    async handleModal (mountpoint) {
      mountpoint ? (this.mountpoint = mountpoint) : (this.mountpoint = '')
      this.toggleModal = !this.toggleModal
    },
    // files handling (addition/ deletion) from child emit event. Child passes data (ref below to respective methods for more detail)
    async handleFiles (command, data) {
      let file
      let params
      if (data.name.includes('.')) {
        file = '.' + data.path + '/' + data.name
        params = [file]
      } else {
        file = data.path + '/' + data.name
        if (command === 'mkdir') {
          params = ['-p', file]
        } else if (command === 'rm') {
          params = ['-rf', file]
        }
      }
      await this.$rpc.ubus('file', 'exec', {
        command: command,
        params: params
      })
      await this.setMountsData()
      // counter to rerender files in explorer
      this.fileCounter++
    },
    async handleAddition (data) {
      // for handleFiles function should be passed 2 params. 1 - string command, 2 - data object (file name + path to file(incl file))
      !data.name.includes('.')
        ? await this.handleFiles('mkdir', data)
        : await this.handleFiles('touch', data)
    },
    async handleDelete (data) {
      !data.name.includes('.')
        ? await this.handleFiles('rm', data)
        : await this.handleFiles('rm', data)
    }
    // async update () {
    //   const data = await this.getMountsData()
    //   console.log(data)
    //   if (!data) {
    //     this.error = true
    //     return
    //   } return
    // }
  },
  async created () {
    await this.setMountsData()
  }
}
</script>

<style scoped>
img {
  width: 30px;
  height: auto;
}

.container {
  text-align: center;
  border-radius: 4px;
  margin-bottom: 20px;
  padding: 30px 50px;
  margin: 20px 0;
}
</style>
