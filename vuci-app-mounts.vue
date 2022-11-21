<template>
  <div>
    <a-table :loading="mounts.length === 0" :columns="columns" :data-source="mounts" defaultExpandAllRows>
      <template #memory="record">
        <span>
          <vuci-progress-bar :value="formatPercentage(record)"/>
        </span>
      </template>
      <template #actions="record">
        <span>
          <a-button key="file-explorer" style="padding: 0rem 0.2rem" @click="handleModal(record.mountpoint)">
            <img src="/icons/folder-icon.png" />
          </a-button>
        </span>
      </template>
    </a-table>
    <custom-modal @closeModal="handleModal" :toggleModal="toggleModal" :path="mountpoint" :fileCounter="fileCounter" @addNew="handleAddition" @deleteItem="handleDelete" @uploadedFile="fileCounter++"></custom-modal>
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
  data () {
    return {
      mounts: [],
      columns: [
        { dataIndex: 'mountpoint', key: 'mountpoint', title: 'Mountpoint' },
        { dataIndex: 'dev', key: 'dev', title: 'Device' },
        { dataIndex: 'fs', key: 'fs', title: 'Filesystem' },
        { dataIndex: 'used_percentage', key: 'used_percentage', title: 'Used memory percentage', scopedSlots: { customRender: 'memory' } },
        { key: 'actions', title: '', scopedSlots: { customRender: 'actions' } }
      ],
      toggleModal: false,
      mountpoint: '',
      fileCounter: 0
    }
  },
  methods: {
    // additional formatting for progress bar
    formatPercentage (data) {
      const index = data.search('%')
      const percentage = Number(data.slice(0, index))
      return percentage
    },
    async getMountsData () {
      // command to extract mounts primary data from router
      const data = await this.$rpc.ubus('file', 'exec', {
        command: '/bin/fmt-usb-msd.sh',
        params: ['unmountable']
      })
      // hard coded default value of usb port
      const convertedData = JSON.parse(data.stdout)['/dev/sda2']
      return [convertedData]
    },
    async setMountsData () {
      const data = await this.getMountsData()
      const convertedData = data.map((el, i) => {
        return {
          key: `mount-${i}`,
          ...el
        }
      })
      this.mounts = convertedData
    },
    handleModal (mountpoint) {
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
        command === 'mkdir' ? (params = ['-p', file]) : (params = [file])
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
      !data.name.includes('.') ? await this.handleFiles('mkdir', data) : await this.handleFiles('touch', data)
    },
    async handleDelete (data) {
      !data.name.includes('.') ? await this.handleFiles('rmdir', data) : await this.handleFiles('rm', data)
    }
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
