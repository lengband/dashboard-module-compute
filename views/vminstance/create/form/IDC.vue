<template>
  <div>
    <a-form
      class="mt-3"
      :form="form.fc"
      @submit="submit">
      <servertemplate v-if="isServertemplate" :decorators="decorators.servertemplate" :formItemLayout="formItemLayout" />
      <a-divider orientation="left">基础配置</a-divider>
      <a-form-item v-show="!isServertemplate" :label="`指定${$t('dictionary.project')}`" v-bind="formItemLayout">
        <domain-project :fc="form.fc" :decorators="{ project: decorators.project, domain: decorators.domain }" />
      </a-form-item>
      <a-form-item label="区域" class="mb-0" v-bind="formItemLayout">
        <cloudregion-zone
          :zone-params="zoneParams"
          :cloudregion-params="cloudregionParams"
          :decorator="decorators.cloudregionZone" />
      </a-form-item>
      <a-form-item label="名称" v-if="!isServertemplate" v-bind="formItemLayout">
        <a-input v-decorator="decorators.name" :placeholder="$t('validator.serverCreateName')" />
        <name-repeated
          v-slot:extra
          res="servers"
          :name="form.fd.name"
          default-text="名称支持有序后缀占位符‘#’，用法举例，名称host##，数量2，创建后实例的名称依次为host01、host02，已有同名实例，序号顺延" />
      </a-form-item>
      <a-form-item label="申请原因" v-bind="formItemLayout" v-if="isOpenWorkflow">
        <a-input v-decorator="decorators.reason" placeholder="请输入主机申请原因" />
      </a-form-item>
      <a-form-item label="数量" v-show="!isServertemplate" v-bind="formItemLayout">
        <a-input-number v-decorator="decorators.count" :min="1" :max="10" />
      </a-form-item>
      <a-form-item label="平台" v-bind="formItemLayout" extra="根据选择的区域不同，平台的可用类型不同且目前只有OneCloud支持GPU云服务器、云硬盘">
        <hypervisor-radio :decorator="decorators.hypervisor" :type="form.fi.createType" :hypervisors="hypervisors" />
      </a-form-item>
      <a-form-item v-if="form.fd.hypervisor === 'kvm'" v-bind="formItemLayout" label="是否配置GPU" extra="目前只有OneCloud支持GPU云服务器">
        <gpu :decorators="decorators.gpu" :gpu-options="gpuOptions" />
      </a-form-item>
      <a-form-item v-bind="formItemLayout" label="操作系统" extra="操作系统会根据选择的虚拟化平台和可用区域的变化而变化，公共镜像的维护请联系管理员">
        <os-select
          :type="type"
          :uefi="uefi"
          :hypervisor="form.fd.hypervisor"
          :decorator="decorators.imageOS"
          :image-params="scopeParams"
          :cacheImageParams="cacheImageParams"
          @updateImageMsg="updateFi" />
      </a-form-item>
      <a-form-item label="CPU核数" v-bind="formItemLayout" class="mb-0">
        <cpu-radio :decorator="decorators.vcpu" :options="form.fi.cpuMem.cpus || []" @change="cpuChange" />
      </a-form-item>
      <a-form-item label="内存" v-bind="formItemLayout" class="mb-0">
        <mem-radio :decorator="decorators.vmem" :options="form.fi.cpuMem.mems_mb || []" />
      </a-form-item>
      <a-form-item label="套餐" v-bind="formItemLayout" v-if="showSku">
        <sku
          v-decorator="decorators.sku"
          :type="type"
          :sku-params="skuParam"
          :hypervisor="form.fd.hypervisor" />
      </a-form-item>
      <a-form-item label="系统磁盘" v-bind="formItemLayout" class="mb-0">
        <system-disk
          v-if="form.fd.hypervisor"
          :decorator="decorators.systemDisk"
          :type="type"
          :hypervisor="form.fd.hypervisor"
          :sku="form.fd.sku"
          :capability-data="form.fi.capability"
          :image="form.fi.imageMsg"
          :isHostImageType="isHostImageType"
          :disabled="form.fi.sysDiskDisabled" />
      </a-form-item>
      <a-form-item label="数据盘" v-bind="formItemLayout">
        <data-disk
          v-if="form.fd.hypervisor"
          ref="dataDiskRef"
          :decorator="decorators.dataDisk"
          :form="form"
          :type="type"
          :hypervisor="form.fd.hypervisor"
          :sku="form.fd.sku"
          :capability-data="form.fi.capability"
          :isSnapshotImageType="isSnapshotImageType"
          :isHostImageType="isHostImageType"
          :disabled="form.fi.dataDiskDisabled"
          :domain="project_domain" />
      </a-form-item>
      <a-form-item label="管理员密码" v-if="!isIso" v-bind="formItemLayout">
        <server-password :form="form" :login-types="loginTypes" :isSnapshotImageType="isSnapshotImageType" :decorator="decorators.loginConfig" />
      </a-form-item>
      <a-form-item label="网络" v-bind="formItemLayout" class="mb-0">
        <server-network
          :decorator="decorators.network"
          :network-list-params="networkParam"
          :schedtag-params="params.schedtag" />
      </a-form-item>
      <a-form-item label="标签" v-bind="formItemLayout" class="mb-0">
        <tag
          v-decorator="decorators.tag" />
      </a-form-item>
      <a-divider orientation="left" v-if="showAdvanceConfig">高级配置</a-divider>
      <a-form-item label="安全组" v-if="isKvm" v-bind="formItemLayout">
        <secgroup-config
          :form="form"
          :isSnapshotImageType="isSnapshotImageType"
          :decorators="decorators.secgroup"
          :secgroup-params="secgroupParams"
          :hypervisor="form.fd.hypervisor" />
      </a-form-item>
      <a-form-item v-show="!isServertemplate" label="调度策略" v-bind="formItemLayout" class="mb-0">
        <sched-policy
          :server-type="form.fi.createType"
          :disabled-host="policyHostDisabled"
          :policy-host-params="policyHostParams"
          :decorators="decorators.schedPolicy"
          :policy-schedtag-params="params.policySchedtag" />
      </a-form-item>
      <a-form-item label="引导方式" v-bind="formItemLayout" class="mb-0" v-if="isKvm">
        <bios :decorator="decorators.bios" :uefi="uefi" />
      </a-form-item>
      <a-form-item v-bind="formItemLayout" v-show="!isServertemplate" v-if="isKvm && isLocalDisk" label="高可用" extra="只有宿主机数量不少于2台时才可以使用该功能">
        <backup
          :decorator="decorators.backup"
          :disabled="form.fd.systemDiskType"
          :disabled-items="backupDisableds" />
      </a-form-item>
      <a-form-item v-bind="formItemLayout" v-show="!isServertemplate" label="到期释放">
        <duration :decorators="decorators.duration" :form="form" />
      </a-form-item>
      <a-form-item v-bind="formItemLayout" v-show="!isServertemplate" v-if="isKvm" label="主机组" extra="对资源的简单调度策略，组内的机器根据设置分布在不同的宿主机上，从而实现业务的高可用">
        <instance-groups :decorators="decorators.groups" :params="instanceGroupsParams" />
      </a-form-item>
      <bottom-bar
        :loading="submiting"
        :form="form"
        :type="type"
        :isOpenWorkflow="isOpenWorkflow"
        :errors.sync="errors"
        :isServertemplate="isServertemplate"
        :hasMeterService="hasMeterService" />
    </a-form>
  </div>
</template>
<script>
import _ from 'lodash'
import * as R from 'ramda'
import SecgroupConfig from '@Compute/sections/SecgroupConfig'
import mixin from './mixin'
import { HYPERVISORS_MAP } from '@/constants'
import { resolveValueChangeField } from '@/utils/common/ant'
import { IMAGES_TYPE_MAP, STORAGE_TYPES } from '@/constants/compute'

export default {
  name: 'VM_IDCCreate',
  components: {
    SecgroupConfig,
  },
  mixins: [mixin],
  computed: {
    isKvm () {
      return this.form.fd.hypervisor === HYPERVISORS_MAP.kvm.key
    },
    isIso () {
      return this.form.fd.imageType === IMAGES_TYPE_MAP.iso.key
    },
    isLocalDisk () {
      return _.get(this.form, 'fd.systemDiskType.key') === 'local'
    },
    hypervisors () {
      const { hypervisors = [] } = this.form.fi.capability
      return hypervisors
    },
    cloudregionParams () {
      return {
        cloud_env: 'onpremise',
        usable: true,
        show_emulated: true,
        ...this.scopeParams,
      }
    },
    zoneParams () {
      return {
        usable: true,
        show_emulated: true,
        order_by: 'created_at',
        order: 'asc',
        ...this.scopeParams,
      }
    },
    instanceGroupsParams () {
      return {
        ...this.scopeParams,
        enabled: true,
      }
    },
    cacheImageParams () {
      const params = {
        cloudregion_id: _.get(this.form.fd, 'cloudregion.key'),
      }
      if (!params.region) return {}
      return params
    },
    showSku () {
      if (this.form.fd.hypervisor && this.form.fd.vcpu && this.form.fd.vmem) {
        return true
      }
      return false
    },
    skuParam () {
      return {
        limit: 0,
        public_cloud: false,
        postpaid_status: 'available',
        cpu_core_count: this.form.fd.vcpu || this.decorators.vcpu[1].initialValue,
        memory_size_mb: this.form.fd.vmem,
        cloudregion: _.get(this.form, 'fd.cloudregion.key'),
        provider: 'OneCloud',
        ...this.scopeParams,
      }
    },
    policyHostParams () {
      const zone = _.get(this.form.fd, 'zone.key')
      if (zone) {
        return {
          enabled: 1,
          usable: true,
          zone,
          hypervisor: this.form.fd.hypervisor,
        }
      }
      return {}
    },
    networkParam () {
      if (!this.cloudregionZoneParams.cloudregion) return {}
      return {
        filter: 'server_type.notin(ipmi, pxe)',
        usable: true,
        ...this.cloudregionZoneParams,
        ...this.scopeParams,
      }
    },
    instanceSpecParmas () {
      return {
        usable: true,
        enabled: true,
        cloudregion: _.get(this.form.fd, 'cloudregion.key'),
      }
    },
    showAdvanceConfig () { // 是否展示高级配置
      return this.isKvm || !this.isServertemplate
    },
    uefi () {
      if (this.isKvm && this.form.fd.gpuEnable && this.form.fd.gpu && this.form.fd.os === 'Windows') {
        return true
      }
      return false
    },
  },
  watch: {
    'form.fi.imageMsg': {
      deep: true,
      handler (val, oldVal) {
        if (R.equals(val, oldVal)) return
        this.$nextTick(() => {
          this.form.fi.dataDiskDisabled = false
          this.form.fi.sysDiskDisabled = false
          if (this.form.fd.imageType === IMAGES_TYPE_MAP.host.key) {
            const { root_image: rootImage, data_images: dataImages } = this.form.fi.imageMsg
            const systemDiskSize = rootImage.min_disk_mb / 1024
            const systemDiskType = { // 这里写死即可，因为主机镜像仅 kvm 型机器，且和系统盘类型无瓜葛 @郑雨
              key: STORAGE_TYPES[HYPERVISORS_MAP.kvm.key].local.key,
              label: STORAGE_TYPES[HYPERVISORS_MAP.kvm.key].local.label,
            }
            this.form.fc.setFieldsValue({
              systemDiskSize,
              systemDiskType,
            })
            // 重置数据盘数据
            this._resetDataDisk()
            dataImages.forEach(val => {
              this.$refs.dataDiskRef.add({ size: val.min_disk_mb / 1024, min: val.min_disk_mb / 1024, minusDisabled: true })
            })
          } else if (this.form.fd.imageType === IMAGES_TYPE_MAP.snapshot.key) {
            // 镜像类型为主机快照的话要回填数据并禁用
            const snapshots = this.form.fi.imageMsg.server_config.disks
            if (!snapshots) return
            const sysDisk = snapshots.find(val => val.disk_type === 'sys')
            const dataDisks = snapshots.filter(val => val.disk_type !== 'sys')
            this.form.fc.setFieldsValue({
              systemDiskType: {
                key: sysDisk.backend,
                label: STORAGE_TYPES[HYPERVISORS_MAP.kvm.key][sysDisk.backend].label,
              },
              systemDiskSize: sysDisk.size / 1024,
            })
            // 重置数据盘数据
            this._resetDataDisk()
            dataDisks.forEach(val => {
              this.$refs.dataDiskRef.add({ size: val.size / 1024, sizeDisabled: true })
            })
            this.form.fi.dataDiskDisabled = true
            this.form.fi.sysDiskDisabled = true
          }
        })
      },
    },
    uefi (val, oldVal) {
      if (val) {
        this.form.fc.setFieldsValue({ bios: 'UEFI' })
      } else {
        this.form.fc.setFieldsValue({ bios: 'BIOS' })
      }
    },
  },
  methods: {
    onValuesChange (vm, changedFields) {
      this.$nextTick(() => {
        const formValue = this.form.fc.getFieldsValue()
        const newField = resolveValueChangeField(changedFields)
        this._setNewFieldToFd(newField, formValue)
        const keys = Object.keys(newField)
        if (keys.includes('zone') || keys.includes('cloudregion')) {
          this.fetchCapability()
        }
        if (keys.includes('cloudregion')) {
          this.$nextTick(this.fetchInstanceSpecs)
        }
      })
    },
    fetchCapability () {
      const params = {
        show_emulated: true,
        resource_type: 'shared',
        ...this.scopeParams,
      }
      let id = this.cloudregionZoneParams.cloudregion
      let resource = 'cloudregions'
      if (this.cloudregionZoneParams.zone) {
        id = this.cloudregionZoneParams.zone
        resource = 'zones'
      }
      const capabilityParams = { id, spec: 'capability', params }
      if (!id) return
      if (R.equals(this.capabilityParams, capabilityParams)) return // 和已有的参数一样则不发请求
      this.capabilityParams = capabilityParams
      new this.$Manager(resource).getSpecific(this.capabilityParams)
        .then(({ data }) => {
          this.form.fi.capability = {
            ...data,
            hypervisors: data.hypervisors.filter(val => val !== 'baremetal'),
          }
          this.form.fc.setFieldsValue({
            hypervisor: this.form.fi.capability.hypervisors[0], // 赋值默认第一个平台
          })
        })
    },
    fetchInstanceSpecs () {
      if (!this.instanceSpecParmas.cloudregion) return
      this.serverskusM.get({ id: 'instance-specs', params: this.instanceSpecParmas })
        .then(({ data }) => {
          this.form.fi.cpuMem = data
          const vcpuDecorator = this.decorators.vcpu
          const vcpuInit = vcpuDecorator[1].initialValue
          this.cpuChange(vcpuInit)
        })
    },
  },
}
</script>
