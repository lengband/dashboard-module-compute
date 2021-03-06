<template>
  <page-list
    :list="list"
    :columns="columns"
    :group-actions="groupActions"
    :single-actions="singleActions"
    :export-data-options="exportDataOptions" />
</template>

<script>
import { weekOptions, timeOptions } from '../constants'
import {
  getNameDescriptionTableColumn,
  getStatusTableColumn,
  getProjectTableColumn,
  getTimeTableColumn,
} from '@/utils/common/tableColumn'
import { getTenantFilter, getStatusFilter } from '@/utils/common/tableFilter'
import expectStatus from '@/constants/expectStatus'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'SnapshotPolicyList',
  mixins: [WindowsMixin],
  props: {
    id: String,
    getParams: {
      type: Object,
      default: () => ({}),
    },
  },
  data () {
    return {
      list: this.$list.createList(this, {
        id: this.id,
        resource: 'snapshotpolicies',
        steadyStatus: Object.values(expectStatus.snapshotpolicy).flat(),
        getParams: this.getParam,
        filterOptions: {
          name: {
            label: '名称',
            filter: true,
            formatter: val => {
              return `name.contains("${val}")`
            },
          },
          status: getStatusFilter('snapshotpolicy'),
          tenant: getTenantFilter(),
        },
      }),
      exportDataOptions: {
        items: [
          { label: 'ID', key: 'id' },
          { label: '名称', key: 'name' },
          { label: '关联硬盘数量', key: 'binding_disk_count' },
          { label: '策略详情', key: 'repeat_weekdays' },
          { label: '创建时间', key: 'create_at' },
          { label: '状态', key: 'status' },
          { label: this.$t('dictionary.project'), key: 'tenant' },
        ],
      },
      columns: [
        getNameDescriptionTableColumn({
          vm: this,
          hideField: true,
          slotCallback: row => {
            return (
              <side-page-trigger onTrigger={ () => this.sidePageTriggerHandle(row.id, 'SnapshotPolicySidePage', { type: this.type }) }>{ row.name }</side-page-trigger>
            )
          },
        }),
        getStatusTableColumn({ statusModule: 'snapshotpolicy' }),
        {
          filed: 'binding_disk_count',
          title: '关联硬盘数量',
          width: 120,
          formatter: ({ row }) => {
            return row.binding_disk_count
          },
        },
        {
          filed: 'repeat_weekdays',
          title: '策略详情',
          minWidth: 180,
          showOverflow: 'ellipsis',
          slots: {
            default: ({ row }, h) => {
              let text = ''
              if (row.repeat_weekdays && row.repeat_weekdays.length) {
                text += '每' + row.repeat_weekdays.map(item => weekOptions[item - 1]).join('、')
              }
              if (row.time_points && row.time_points.length) {
                text += '; ' + row.time_points.map(item => timeOptions[item]).join('、')
              }
              if (text) {
                text += '自动创建快照'
              }
              return [
                <list-body-cell-wrap copy field='repeat_weekdays' hideField row={row} message={text}>
                  {{ text }}
                </list-body-cell-wrap>,
              ]
            },
          },
        },
        getTimeTableColumn(),
        getProjectTableColumn(),
      ],
      groupActions: [
        {
          label: '新建',
          action: () => {
            this.createDialog('CreateSnapshotPolicyDialog', {
              data: this.list.selectedItems,
              columns: this.columns,
              title: '新建',
              list: this.list,
            })
          },
          meta: () => {
            return {
              buttonType: 'primary',
            }
          },
        },
        {
          label: '删除',
          permission: 'snapshotpolicies_delete',
          action: () => {
            this.createDialog('DeleteResDialog', {
              data: this.list.selectedItems,
              columns: this.columns,
              title: '删除',
              list: this.list,
            })
          },
          meta: () => this.$getDeleteResult(this.list.selectedItems),
        },
      ],
      singleActions: [
        {
          label: '关联硬盘',
          action: obj => {
            this.createDialog('AttachDiskDialog', {
              data: [obj],
              columns: this.columns,
              title: '关联硬盘',
              list: this.list,
            })
          },
          meta: obj => {
            return {
              validate: true,
            }
          },
        },
        {
          label: '删除',
          action: obj => {
            this.createDialog('DeleteResDialog', {
              data: [obj],
              columns: this.columns,
              title: '删除',
              list: this.list,
              success: () => {
                this.destroySidePages()
              },
            })
          },
          meta: obj => this.$getDeleteResult(obj),
        },
      ],
    }
  },
  created () {
    this.initSidePageTab('snapshot-policy-detail')
    this.list.fetchData()
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
