<template>
  <div>
    <template v-if="isGlobalSearch">
      <snapshot-list :list="list" type="instance" v-bind="$props" />
    </template>
    <template v-else>
      <page-header title="主机快照" :tabs="cloudEnvOptions" :current-tab.sync="cloudEnv" />
      <page-body>
        <snapshot-list :list="list" type="instance" :cloud-env="cloudEnv" v-bind="$props" />
      </page-body>
    </template>
  </div>
</template>

<script>
import { steadyStatus } from './constants'
import SnapshotList from './components/List'
import { getTenantFilter, getStatusFilter } from '@/utils/common/tableFilter'
import { getCloudEnvOptions } from '@/utils/common/hypervisor'
import globalSearchMixins from '@/mixins/globalSearch'

export default {
  name: 'VmInstanceSnapshotsIndex',
  components: {
    SnapshotList,
  },
  mixins: [globalSearchMixins],
  data () {
    return {
      cloudEnvOptions: getCloudEnvOptions('compute_engine_brands'),
      cloudEnv: '',
      list: this.$list.createList(this, {
        id: 'VmInstanceSnapshotsList',
        resource: 'instance_snapshots',
        getParams: this.getParam,
        steadyStatus,
        filterOptions: {
          name: {
            label: '名称',
            filter: true,
            formatter: val => {
              return `name.contains("${val}")`
            },
          },
          status: getStatusFilter('snapshot'),
          tenant: getTenantFilter(),
          // guest: {
          //   label: '虚拟机',
          //   dropdown: true,
          //   multiple: true,
          //   distinctField: {
          //     type: 'field',
          //     key: 'guest',
          //   },
          // },
        },
        responseData: this.responseData,
      }),
    }
  },
  computed: {
    isGlobalSearch () {
      return this.responseData && this.showGroupActions === false && this.showSearchbox === false
    },
  },
  watch: {
    cloudEnv (val) {
      this.$nextTick(() => {
        this.list.fetchData(0)
      })
    },
  },
  methods: {
    getParam () {
      const ret = {
        details: true,
      }
      if (this.cloudEnv) ret.cloud_env = this.cloudEnv
      return ret
    },
  },
}
</script>
