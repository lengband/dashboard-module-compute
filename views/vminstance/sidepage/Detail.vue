<template>
  <detail
    :list="list"
    :data="data"
    :extra-info="extraInfo"
    :base-info="baseInfo"
    status-module="server" />
</template>

<script>
import { ALL_STORAGE } from '@Compute/constants/index'
import {
  getCopyWithContentTableColumn,
  getBrandTableColumn,
  getSwitchTableColumn,
} from '@/utils/common/tableColumn'

export default {
  name: 'VmInstanceDetail',
  props: {
    list: {
      type: Object,
      required: true,
    },
    data: {
      type: Object,
      required: true,
    },
  },
  data () {
    return {
      serverDetail: {},
      baseInfo: [
        {
          field: 'keypair',
          title: '关联密钥',
        },
        getBrandTableColumn(),
      ],
    }
  },
  computed: {
    diskInfos () {
      const disksInfo = this.serverDetail.disks_info
      if (!disksInfo) return {}
      const dataDisk = {}
      const sysDisk = {}
      let image = '-'
      let sysDisks = disksInfo.filter(v => v.disk_type === 'sys')
      if (sysDisks && sysDisks.length === 0) {
        sysDisks = disksInfo.filter(v => v.index === 0)
      }
      const dataDisks = disksInfo.filter(v => v.index !== 0)
      if (sysDisks && sysDisks.length > 0) {
        const sysKey = sysDisks[0].storage_type
        image = sysDisks[0].image || '-'
        sysDisk[sysKey] = this._dealSize(sysDisks)
      }
      if (dataDisks && dataDisks.length > 0) {
        for (const k in ALL_STORAGE) {
          const e = ALL_STORAGE[k]
          const sameType = dataDisks.filter(v => v.storage_type === e.value)
          if (sameType && sameType.length) {
            dataDisk[k] = this._dealSize(sameType)
          }
        }
      }
      if (this.data.cdrom && dataDisks.length > 0) {
        image = dataDisks[0].image
      }
      return {
        sysDisk: this._diskStringify(sysDisk) || '-',
        dataDisk: this._diskStringify(dataDisk) || '-',
        image,
      }
    },
    extraInfo () {
      return [
        {
          title: '配置信息',
          items: [
            {
              field: 'os_type',
              title: '操作系统',
              formatter: ({ row }) => {
                const distribution = (row.metadata && row.metadata.os_distribution) ? row.metadata.os_distribution : row.os_type
                const { os_version: version = '' } = row.metadata || {}
                return distribution + version
              },
            },
            getCopyWithContentTableColumn({ field: 'ips', title: 'IP' }),
            getCopyWithContentTableColumn({
              field: 'image',
              title: '系统镜像',
              hideField: true,
              message: this.diskInfos.image,
              slotCallback: row => {
                return [<span>{ this.diskInfos.image }</span>]
              },
            }),
            {
              field: 'isolated_devices',
              title: 'GPU',
              formatter: ({ row }) => {
                if (!row.isolated_devices) return '-'
                let gpuArr = row.isolated_devices
                let obj = {}
                gpuArr.forEach(val => {
                  if (!obj[val.model]) {
                    obj[val.model] = 1
                  } else {
                    obj[val.model] += 1
                  }
                })
                let str = ''
                for (const k in obj) {
                  const n = obj[k]
                  str += `、${n}颗（${k}）`
                }
                return str.slice(1)
              },
            },
            {
              field: 'vcpu_count',
              title: 'CPU',
              formatter: ({ row }) => {
                return row.vcpu_count + '核'
              },
            },
            {
              field: 'vmem_size',
              title: '内存',
              formatter: ({ row }) => {
                return (row.vmem_size / 1024) + 'GB'
              },
            },
            {
              field: 'dataDisk',
              title: '数据盘',
              formatter: ({ row }) => {
                return this.diskInfos.dataDisk
              },
            },
            {
              field: 'sysDisk',
              title: '系统盘',
              formatter: ({ row }) => {
                return this.diskInfos.sysDisk
              },
            },
            getCopyWithContentTableColumn({
              field: 'cdrom',
              title: 'ISO',
              hideField: true,
              slotCallback: row => {
                const idx = row.cdrom.indexOf('(')
                return row.cdrom.substring(0, idx) || '-'
              },
            }),
          ],
        },
        {
          title: '其他设置',
          items: [
            {
              field: 'backup_host_name',
              title: '备份机的宿主机',
            },
            getSwitchTableColumn({
              field: 'disable_delete',
              title: '删除保护',
              change: val => {
                this.list.onManager('update', {
                  id: this.data.id,
                  managerArgs: {
                    data: { disable_delete: val },
                  },
                })
              },
            }),
          ],
        },
      ]
    },
  },
  created () {
    const manager = new this.$Manager('servers')
    manager.get({ id: this.data.id }).then(res => {
      this.serverDetail = res.data || {}
    })
  },
  methods: {
    _diskStringify (diskObj) {
      let str = ''
      const storageArr = Object.values(ALL_STORAGE)
      for (const k in diskObj) {
        const num = diskObj[k]
        const disk = storageArr.find(v => v.value === k)
        if (disk) {
          str += `、${parseInt(num / 1024)}GB（${disk.label}）`
        }
      }
      return str.slice(1)
    },
    _dealSize (sameType) {
      let sameType1 = sameType.map(v => {
        let size = +v.size
        return size
      })
      return sameType1.reduce((a, b) => {
        return a + b
      })
    },
  },
}
</script>
