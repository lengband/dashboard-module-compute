<template>
  <div class="data-disk">
    <template v-if="dataDisks.length === 0 && disabled"><span class="warning-color">无数据盘</span></template>
    <template v-else>
      <div class="d-flex" v-for="(item, i) in dataDisks" :key="item.key">
        <disk
          :max="max"
          :min="item.min || min"
          :snapshots-params="getSnapshotsParams(item)"
          :diskTypeLabel="i === 0 ? '' : diskTypeLabel"
          :decorator="genDecorator(item.key)"
          :hypervisor="hypervisor"
          :types-map="typesMap"
          :elements="elements"
          :disabled="getDisabled(item)"
          :size-disabled="item.sizeDisabled"
          @diskTypeChange="val => diskTypeChange(item, val)" />
        <a-button v-if="!getDisabled(item, 'minus')" shape="circle" icon="minus" size="small" @click="decrease(item.key)" class="mt-2" />
      </div>
      <div class="d-flex align-items-center" v-if="diskRemain > 0 && !disabled">
        <a-button type="primary" shape="circle" icon="plus" size="small" @click="add" />
        <a-button type="link" @click="add">添加新磁盘</a-button>
        <span class="count-tips">您还可以添加 <span class="remain-num">{{ diskRemain }}</span> 块</span>
      </div>
    </template>
  </div>
</template>

<script>
import _ from 'lodash'
import * as R from 'ramda'
import Disk from '@Compute/sections/Disk'
import { STORAGE_AUTO } from '@Compute/constants'
import { STORAGE_TYPES } from '@/constants/compute'
import { HYPERVISORS_MAP } from '@/constants'
import { uuid } from '@/utils/utils'

// 磁盘最小值
const DISK_MIN_SIZE = 10

export default {
  name: 'dataDisk',
  inject: ['form'],
  components: {
    Disk,
  },
  props: {
    type: {
      type: String,
      required: true,
      validator: val => ['idc', 'private', 'public'].includes(val),
    },
    hypervisor: {
      type: String,
      required: true,
    },
    sku: {
      type: Object,
    },
    capabilityData: {
      type: Object,
      required: true,
    },
    decorator: {
      type: Object,
      required: true,
      validator: val => {
        const fields = ['type', 'size', 'schedtag', 'policy', 'snapshot', 'filetype', 'mountPath']
        return fields.every(f => R.is(Function, val[f]))
      },
    },
    disabled: {
      type: Boolean,
      default: false,
    },
    isHostImageType: {
      type: Boolean,
      default: false,
    },
    isSnapshotImageType: {
      type: Boolean,
      default: false,
    },
    domain: {
      type: String,
      default: 'default',
    },
  },
  data () {
    return {
      dataDisks: [],
    }
  },
  computed: {
    isPublic () {
      return this.type === 'public'
    },
    isPrivate () {
      return this.type === 'private'
    },
    isIDC () {
      return this.type === 'idc'
    },
    elements () {
      let ret = []
      if (this.isSnapshotImageType) return ret
      if (this.isHostImageType) return ['snapshot', 'schedtag']
      if (this.hypervisor === HYPERVISORS_MAP.kvm.key) {
        ret.push('mount-point')
        ret.push('snapshot')
      }
      if (!this.isPublic) {
        ret.push('schedtag')
      }
      return ret
    },
    typesMap () {
      const ret = {}
      const hyper = this.getHypervisor()
      const hypervisorDisks = { ...STORAGE_TYPES[hyper] } || {}
      if (!this.capabilityData || !this.capabilityData.data_storage_types2) return ret
      let currentTypes = this.capabilityData.data_storage_types2[hyper] || []
      if (!R.isNil(this.sku) && !R.isEmpty(this.sku)) {
        for (let obj in hypervisorDisks) {
          if (hypervisorDisks[obj].skuFamily && !hypervisorDisks[obj].skuFamily.includes(this.sku.instance_type_family)) {
            delete hypervisorDisks[obj]
          }
        }
      } else {
        if (this.isPublic) {
          currentTypes = []
        }
      }
      for (let i = 0, len = currentTypes.length; i < len; i++) {
        const type = currentTypes[i].split('/')[0]
        let opt = hypervisorDisks[type] || this.getExtraDiskOpt(type)
        if (opt) {
          let min = Math.max(DISK_MIN_SIZE, opt.min)
          if (opt) {
            ret[opt.key] = {
              ...opt,
              min,
            }
          }
        }
      }
      if (this.isIDC && this.hypervisor !== HYPERVISORS_MAP.kvm.key) {
        ret[STORAGE_AUTO.key] = STORAGE_AUTO
      }
      return ret
    },
    currentTypeObj () {
      // 数据盘仅第一块盘的磁盘类型可以修改
      const diskTypeKey = _.get(this.dataDisks, '[0].diskType.key')
      if (diskTypeKey) {
        return this.typesMap[diskTypeKey]
      }
      if (!R.isNil(this.typesMap) && !R.isEmpty(this.typesMap)) {
        const firstKey = Object.keys(this.typesMap)[0]
        return this.typesMap[firstKey]
      }
      return {}
    },
    max () {
      return this.currentTypeObj.max || DISK_MIN_SIZE
    },
    min () {
      return this.currentTypeObj.min || DISK_MIN_SIZE
    },
    diskRemain () {
      const remain = this.capabilityData.max_data_disk_count - this.dataDisks.length
      return Math.max(remain, 0)
    },
    diskTypeLabel () {
      return _.get(this.dataDisks, '[0].diskType.label')
      // if (this.dataDisks.length) {
      //   if (this.dataDisks[0]) {
      //     if (this.dataDisks[0].diskType) {
      //       return this.dataDisks[0].diskType.label || ''
      //     }
      //   }
      // }
      // return ''
    },
  },
  methods: {
    getDisabled (item, itemName) {
      if (item.disabled) return true
      if (itemName && item[`${itemName}Disabled`]) {
        return true // 这里目前仅针对 minus 按钮
      }
      return this.disabled
    },
    genDecorator (uid) {
      const ret = {}
      R.forEachObjIndexed((item, key) => {
        ret[key] = item(uid)
      }, this.decorator)
      return ret
    },
    decrease (key) {
      const index = this.dataDisks.findIndex(val => val.key === key)
      this.dataDisks.splice(index, 1)
      this.$nextTick(() => {
        if (index === 0 && this.dataDisks.length > 0) {
          const key = `dataDiskTypes[${this.dataDisks[0].key}]`
          const defaultKey = Object.keys(this.typesMap)[0]
          if (defaultKey) {
            const dataDiskTypes = {
              key: this.typesMap[defaultKey].key,
              label: this.typesMap[defaultKey].label,
            }
            this.form.fc.setFieldsValue({
              [key]: dataDiskTypes,
            })
          }
        }
        const formValue = this.form.fc.getFieldsValue()
        if (this.form.fd) { // 如果上层表单有fd时，需要在此同步数据(外层监听不到减少表单的情况)
          this.form.fd.dataDiskSizes = formValue.dataDiskSizes
        }
      })
    },
    add ({ size, diskType, policy, schedtag, snapshot, filetype, mountPath, min, disabled = false, sizeDisabled = false, ...ret } = {}) {
      const key = uuid()
      const typeObj = this.typesMap[diskType]
      let dataDiskTypes = {
        key: _.get(this.dataDisks, '[0].diskType.key'),
        label: _.get(this.dataDisks, '[0].diskType.label'),
      }
      if (R.is(Object, typeObj)) { // 传入diskType，回填
        dataDiskTypes = {
          key: typeObj.key || diskType,
          label: typeObj.label || diskType,
        }
      } else if (!diskType && !_.get(this.dataDisks, '[0].diskType')) { // 表单中数据盘无第一项，需要 set 磁盘类型默认值
        const defaultKey = Object.keys(this.typesMap)[0]
        if (defaultKey) {
          dataDiskTypes = {
            key: this.typesMap[defaultKey].key,
            label: this.typesMap[defaultKey].label,
          }
        }
      }
      console.log(sizeDisabled, 'sizeDisabled')
      const dataDiskItem = {
        key,
        disabled,
        sizeDisabled,
        diskType: dataDiskTypes,
        ...ret, // 目前仅用于 minus 按钮
      }
      if (min) {
        dataDiskItem.min = Math.max(min, this.min, DISK_MIN_SIZE)
      }
      this.dataDisks.push(dataDiskItem)
      this.$nextTick(() => {
        const value = {
          [`dataDiskSizes[${key}]`]: R.is(Number, size) ? size : (min || this.min),
        }
        value[`dataDiskTypes[${key}]`] = dataDiskTypes
        if (schedtag) { // 磁盘调度标签
          value[`dataDiskSchedtags[${key}]`] = schedtag
        }
        if (policy) { // 磁盘调度策略
          value[`dataDiskPolicys[${key}]`] = policy
        }
        if (snapshot && (filetype || mountPath)) {
          console.error('DataDisk组件报错：磁盘的快照和挂载点不能共存')
        }
        if (snapshot) { // 磁盘快照
          value[`dataDiskSnapshots[${key}]`] = snapshot
        }
        if (filetype) { // 磁盘文件系统
          value[`dataDiskFiletypes[${key}]`] = filetype
        }
        if (mountPath) { // 磁盘挂载路径
          value[`dataDiskMountPaths[${key}]`] = mountPath
        }
        this.form.fc.setFieldsValue(value)
      })
    },
    getExtraDiskOpt (type) {
      // 腾讯云过滤掉local_basic和local_ssd类型的盘
      if (this.getHypervisor() === HYPERVISORS_MAP.qcloud.key) {
        if (['local_basic', 'local_ssd'].includes(type)) {
          return
        }
      }
      // VMware过滤掉rbd类型的盘
      if (this.getHypervisor() === HYPERVISORS_MAP.esxi.key) {
        if (['rbd'].includes(type)) {
          return
        }
      }
      return {
        label: `${type}`,
        key: `${type}`,
        min: 1,
        max: 3 * 1024,
        sysMin: 10,
        sysMax: 500,
      }
    },
    getHypervisor () {
      let ret = this.hypervisor
      if (this.isPublic) {
        if (this.sku && this.sku.provider) {
          ret = this.sku.provider.toLowerCase()
        }
      }
      return ret
    },
    getSnapshotsParams (item) {
      const staticParams = {
        with_meta: true,
        cloud_env: 'onpremise',
        limit: 0,
        disk_type: 'data',
      }
      const modeParams = {}
      if (this.$store.getters.isAdminMode) {
        modeParams.project_domain = this.domain
      } else {
        modeParams.scope = this.$store.getters.scope
      }
      if (this.diskTypeLabel) {
        staticParams['joint_filter.0'] = `storages.id(storage_id).storage_type.equals(${_.get(this.form, 'fd.systemDiskType.key')})`
      }
      if (item.diskType && item.diskType.key) {
        staticParams['joint_filter.0'] = `storages.id(storage_id).storage_type.equals(${item.diskType.key})`
      }
      return {
        ...staticParams,
        ...modeParams,
      }
    },
    diskTypeChange (item, val) {
      // 仅有第一块盘可以更改磁盘类型
      console.log(item, 'item')
      this.$nextTick(() => {
        const dataDiskItem = {
          ...item,
          diskType: val,
        }
        if (item.min) {
          dataDiskItem.min = Math.max(item.min, this.min)
        }
        console.log(dataDiskItem, 'dataDiskItem')
        this.$set(this.dataDisks, 0, dataDiskItem)
      })
      // this.dataDisks.forEach((disk, i) => {
      //   console.log(i, {
      //     ...item,
      //     min: Math.max(item.min || DISK_MIN_SIZE, this.min),
      //     diskType: val,
      //   })
      //   this.$set(this.dataDisks, i, {
      //     ...item,
      //     min: Math.max(item.min || DISK_MIN_SIZE, this.min),
      //     diskType: val,
      //   })
      // })
      // 将
    },
  },
}
</script>

<style lang="scss" scoped>
@import '../../../../src/styles/variables';

.data-disk {
  .count-tips {
    .remain-num {
      color: $primary-color;
    }
  }
}
</style>
