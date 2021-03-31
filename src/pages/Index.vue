<template>
  <q-page class="flex flex-center">
    <div class="q-pa-md">
    <q-table
      title="Device Info"
        v-show="claimed"
      :data="data"
      :columns="columns"
      hide-bottom
      row-key="name"
    />
  </div>
  <div>
    <q-btn label="Connect" @click="reqUSB"></q-btn>
  </div>
  </q-page>
</template>

<script>
export default {
  name: 'PageIndex',
  data: () => ({
    columns: [
      { name: 'key', label: 'Key', field: 'key' },
      { name: 'value', label: 'Values', field: 'value', align: 'left' }
    ],
    data: [],
    interfaceStats: [],
    claimed: false,
    device: null
  }),

  methods: {
    async reqUSB () {
      let rawdevice

      if (this.device === null) {
      // request to get all connected usb devices
        rawdevice = await navigator.usb.requestDevice({ filters: [] })

        // do the setup procedure on the connected device
        await this.setup(rawdevice)

        // populating all received data in table
        this.displayDeviceStats()

        this.checkDevice()
        this.sendId()
        this.activateAOAMode()
        this.transferOut()
        this.transferIn()
      }
    },

    async setup (rawdevice) {
      // open the device (initiate communication)
      await rawdevice.open()
      // select the devices configuration descriptor
      await rawdevice.selectConfiguration(1)
      // claim the device interfaces
      rawdevice = await this.claimInterface(rawdevice)
      this.device = rawdevice
      console.log(rawdevice)

      return rawdevice
    },

    async claimInterface (d) {
      for (const config of d.configurations) {
        for (const iface of config.interfaces) {
          this.interfaceStats.push(iface)
          if (!iface.claimed) {
            await d.claimInterface(iface.interfaceNumber)
            this.claimed = true
            return d
          }
        }
      }
      return d
    },
    displayDeviceStats () {
      this.data = [
        {
          key: 'Manufacturer Name : ',
          value: this.device.manufacturerName
        },
        {
          key: 'Product Name : ',
          value: this.device.productName
        },
        {
          key: 'Serial Number :',
          value: this.device.serialNumber
        },
        {
          key: 'Product ID : ',
          value: this.device.productId
        },
        {
          key: 'Vendor ID : ',
          value: this.device.vendorId
        },
        {
          key: 'Devices Found : ',
          value: this.interfaceStats.length
        }
      ]
    },
    async checkDevice () {
      this.device.controlTransferIn(({
        requestType: 'vendor',
        recipient: 'device',
        request: 51, // ACCESSORY_GET_PROTOCOL
        value: 0,
        index: 0
      }), 2).then(response => {
        if (response.data.getInt8(0) === 1) {
          console.log('AoA V1 available')
        }
        if (response.data.getInt8(0) === 2) {
          console.log('AoA V2 available')
        }
      }).catch(e => {
        console.log("Can't send " + e)
      })
    },

    str2ab (str) {
      var buf = new ArrayBuffer(str.length * 2)
      var bufView = new Uint16Array(buf)
      for (var i = 0, strLen = str.length; i < strLen; i++) {
        bufView[i] = str.charCodeAt(i)
      }
      return buf
    },
    sendId () {
      const identifiers = ['manufacturer', 'model', 'description', 'version', 'uri', 'serialnumber']

      for (var i = 0; i < identifiers.length; i++) {
        this.device.controlTransferOut(({
          requestType: 'vendor',
          recipient: 'device',
          request: 52, // ACCESSORY_SEND_STRING
          value: 0,
          index: i
        }), this.str2ab(identifiers[i]))
          .then(response => {
            console.log('ready')
          }).catch(e => {
            console.log(e)
          })
      }
    },
    activateAOAMode () {
      this.device.controlTransferOut(({
        requestType: 'vendor',
        recipient: 'device',
        request: 53, // ACCESSORY_START
        value: 0,
        index: 0
      }), new ArrayBuffer(0)).then(response => {
        console.log(response)
        if (response.status === 'ok') {
          // this.device.close()
        }
      }).catch(e => {
        console.log("Can't send " + e)
      })
    },

    transferOut () {
      const ep1 = 1
      const text = 'Hello'
      this.device.transferOut(ep1, this.str2ab(text)).then(response => console.log(response))
    },
    transferIn () {
      const ep1 = 1
      this.device.transferIn(ep1, 64).then(response => console.log(response))
    }

  }

}
</script>
