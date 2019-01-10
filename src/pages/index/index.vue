<template>
  <div class="container">

  <!-- <button type='primary' @click='bluetoothState'>
    蓝牙状态
  </button> -->
  <button type='primary' @click='search' :loading="loading">
    搜索设备
  </button>
  <button type='primary' @click='stopSearch'>
    停止搜索
  </button>
  <button type='primary' @click='reSearch'>
    重新搜索
  </button>
  <button type='primary' @click='allDevices'>
    所有设备
  </button>
  <div>已发现：{{deviceList.length}}个设备</div>

  <scroll-view
    class="scroll-view"
    scroll-y="true"
    enable-back-to-top="true">
    <div
      :data-device-id="item.deviceId"
      :data-name="item.name||item.localName"
      class="device-item"
      @tap="connectDevice"
      v-for="item in deviceList"
      :key="item.deviceId">
    {{item.name}} - {{item.end}}
  </div>
  </scroll-view>

  </div>
</template>

<script>

export default {
  data () {
    return {
      name: '',
      loading: false,
      deviceList: []
    }
  },

  components: {

  },

  methods: {
    openBluetooth () {
      wx.openBluetoothAdapter({
        success: res => {
          console.log(res)
        },
        fail: err => {
          console.log(err)
        }
      })
    },
    bluetoothState () {
      wx.getBluetoothAdapterState({
        success: res => {
          console.log('状态', res)
        },
        fail: err => {
          console.log('err', err)
        }
      })
    },
    search () {
      this.loading = true
      wx.openBluetoothAdapter({
        success: res => {
          console.log('init success', res)
          wx.startBluetoothDevicesDiscovery({
            success: res => {
              this.onBluetoothDeviceFound()
            }
          })
        },
        fail: err => {
          this.loading = false
          console.log('init error', err)
        }
      })
    },
    stopSearch () {
      this.loading = false
      wx.stopBluetoothDevicesDiscovery()
    },
    allDevices () {
      wx.getBluetoothDevices({
        success: res => {
          console.log(res)
        }
      })
    },
    onBluetoothDeviceFound () {
      wx.onBluetoothDeviceFound(res => {
        const device = res.devices[0]
        const list = [...this.deviceList]
        if (list.findIndex(e => e.deviceId === device.deviceId) < 0) {
          const _device = Object.assign({}, device, {
            name: device.name || 'unknown',
            end: device.deviceId.slice(0, 6)
          })
          list.push(_device)
          this.deviceList = list
          console.log('list: ', list)
        }
      })
    },
    connectDevice (e) {
      const { deviceId, name } = e.currentTarget.dataset
      const that = this
      wx.createBLEConnection({
        deviceId,
        success: (res) => {
          that.name = name
          that.getBLEDeviceServices(deviceId)
        }
      })
      this.stopSearch()
    },
    getBLEDeviceServices (deviceId) {
      const that = this
      wx.getBLEDeviceServices({
        deviceId,
        success: (res) => {
          console.log(res)
          for (let i = 0; i < res.services.length; i++) {
            if (res.services[i].isPrimary) {
              that.getBLEDeviceCharacteristics(deviceId, res.services[i].uuid)
              return
            }
          }
        }
      })
    },
    getBLEDeviceCharacteristics (deviceId, serviceId) {
      wx.getBLEDeviceCharacteristics({
        deviceId,
        serviceId,
        success: (res) => {
          console.log('getBLEDeviceCharacteristics success', res.characteristics)
          for (let i = 0; i < res.characteristics.length; i++) {
            let item = res.characteristics[i]
            if (item.properties.read) {
              wx.readBLECharacteristicValue({
                deviceId,
                serviceId,
                characteristicId: item.uuid,
                success: res => {
                  console.log('read success')
                }
              })
            }
            // if (item.properties.write) {
            //   this.setData({
            //     canWrite: true
            //   })
            //   this._deviceId = deviceId
            //   this._serviceId = serviceId
            //   this._characteristicId = item.uuid
            //   this.writeBLECharacteristicValue()
            // }
            // if (item.properties.notify || item.properties.indicate) {
            //   wx.notifyBLECharacteristicValueChange({
            //     deviceId,
            //     serviceId,
            //     characteristicId: item.uuid,
            //     state: true
            //   })
            // }
          }
        },
        fail (res) {
          console.error('getBLEDeviceCharacteristics', res)
        }
      })
      wx.onBLECharacteristicValueChange((characteristic) => {
        console.log(characteristic)
        console.log(this.ab2hex(characteristic.value))
      })
    },
    ab2hex (buffer) {
      var hexArr = Array.prototype.map.call(
        new Uint8Array(buffer),
        function (bit) {
          return ('00' + bit.toString(16)).slice(-2)
        }
      )
      return hexArr.join('')
    },
    reSearch () {
      this.deviceList = []
      this.search()
    }
  },

  created () {
    // init bluetooth
    // this.openBluetooth()
  }
}
</script>

<style scoped>
button {
  width: 90%;
  margin-bottom: 10px;
}
.scroll-view {
  height: 60vh;
  width: 96%;
  border: 1px solid #333;
}
.device-item {
  height: 30px;
  border: 1px solid skyblue;
  border-radius: 4px;
  width: 90%;
  margin-bottom: 6px;
  padding-left: 10px;
}
</style>
