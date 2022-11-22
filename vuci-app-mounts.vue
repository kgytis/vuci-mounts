<template>
  <div>
    <a-spin :spinning="unmounting" tip="Unmounting...">
      <a-table
        v-if="!isUnmounted"
        :loading="mounts.length === 0"
        :columns="columns"
        :data-source="mounts"
        defaultExpandAllRows
      >
        <template #memory="record">
          <span>
            <vuci-progress-bar
              :value="formatPercentage(record)"
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
            <a-button
              type="primary"
              @click="eject(record)"
              :style="{ padding: '0rem 0.2rem', margin: '0rem 1rem' }"
              ><img src="/icons/eject.png"
            /></a-button>
          </span>
        </template>
      </a-table>
      <custom-modal
        v-if="!isUnmounted"
        @closeModal="closeModal"
        :toggleModal="toggleModal"
        :path="mountpoint"
        :fileCounter="fileCounter"
        @addNew="handleAddition"
        @deleteItem="handleDelete"
        @uploadedFile="fileCounter++"
      ></custom-modal>
      <div v-else-if="isUnmounted" class="unmountedDiv">
        <div>
          <a-spin :spinning="isLoading" size="large">
            <div class="unmountedContent">
              <h1>No mounts detected</h1>
              <a-button @click="setMountsData">Refresh</a-button>
            </div>
          </a-spin>
        </div>
      </div>
    </a-spin>
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
  timers: {
    update: { time: 15000, autostart: true, immediate: true, repeat: true }
  },
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
      error: false,
      isLoading: false,
      isUnmounted: false,
      unmounting: false
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
        this.isUnmounted = true
        this.toggleModal = false
        return
      }
      this.isUnmounted = false
      // this approach only if one usb port present
      // code should be a bit changed if more than one usb port present
      const convertedData = JSON.parse(data.stdout)[keys[0]]
      return [convertedData]
    },
    async setMountsData () {
      // alert('data set')
      this.isLoading = true
      const data = await this.getMountsData()
      this.isLoading = false
      if (!data) {
        this.isUnmounted = true
        return
      }
      this.isUnmounted = false
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
      const data = await this.$rpc.ubus('file', 'list', { path: mountpoint })
      // this.toggleModal = !this.toggleModal
      if (data !== null) {
        this.toggleModal = !this.toggleModal
      } else {
        this.$message.error('No mounts detected. Check mounts in your device')
        this.isUnmounted = true
      }
    },
    async closeModal () {
      this.toggleModal = !this.toggleModal
      this.fileCounter++
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
      await this.handleFiles('rm', data)
    },
    async eject () {
      // hard coded device
      const data = await this.$rpc.ubus('file', 'exec', {
        command: 'umount',
        params: ['/dev/sda2']
      })
      function timeout (ms) {
        return new Promise((resolve) => setTimeout(resolve, ms))
      }
      if (data.code === 0) {
        this.unmounting = true
        await timeout(1500)
        this.unmounting = false
        this.isUnmounted = true
        this.$message.success('Mount succesfully unmounted.')
      } else {
        this.$message.error('Mount is still busy. Try again later.')
      }
    },
    async update () {
      await this.getMountsData()
    }
  },
  watch: {
    async isUnmounted (curr, prev) {
      if (!curr && prev) {
        await this.setMountsData()
        this.fileCounter++
      }
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

.unmountedDiv {
  height: 80vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.unmountedContent {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
</style>
