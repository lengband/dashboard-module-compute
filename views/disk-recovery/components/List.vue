<template>
  <page-list
    :list="list"
    :columns="columns"
    :group-actions="groupActions"
    :single-actions="singleActions"
    :export-data-options="exportDataOptions"
    :showSearchbox="showSearchbox"
    :showGroupActions="showGroupActions" />
</template>

<script>
import expectStatus from '@/constants/expectStatus'
import { getNameFilter, getStatusFilter, getBrandFilter, getTenantFilter, getFilter } from '@/utils/common/tableFilter'
import { getBrandTableColumn, getStatusTableColumn, getCopyWithContentTableColumn, getProjectTableColumn, getTimeTableColumn } from '@/utils/common/tableColumn'
import WindowsMixin from '@/mixins/windows'
import globalSearchMixins from '@/mixins/globalSearch'

export default {
  name: 'DiskRecoveryList',
  mixins: [WindowsMixin, globalSearchMixins],
  props: {
    id: String,
  },
  data () {
    return {
      list: this.$list.createList(this, {
        id: this.id,
        resource: 'disks',
        getParams: this.getParams,
        steadyStatus: Object.values(expectStatus.server).flat(),
        filterOptions: {
          name: getNameFilter(),
          brand: getBrandFilter(),
          status: getStatusFilter('disk'),
          tenant: getTenantFilter(),
          // guest: getFilter({ field: 'guest', title: '云服务器' }),
          disk_type: getFilter({
            field: 'guest',
            title: '类型',
            items: [
              { label: '系统盘', key: 'sys' },
              { label: '数据盘', key: 'data' },
            ] }),
        },
        responseData: this.responseData,
      }),
      exportDataOptions: {
        items: [
          { label: 'ID', key: 'id' },
          { label: '名称', key: 'name' },
          { label: '容量', key: 'disk_size' },
          { label: '格式', key: 'disk_format' },
          { label: '磁盘类型', key: 'disk_type' },
          { label: '是否挂载', key: 'unused' },
          { label: '主机', key: 'guest' },
          { label: '主存储', key: 'storage' },
          { label: '创建时间', key: 'created_at' },
          { label: '状态', key: 'status' },
          { label: this.$t('dictionary.project'), key: 'tenant' },
          { label: '平台', key: 'provider' },
          { label: '区域', key: 'region' },
          { label: '可用区', key: 'zone' },
          { label: '介质类型', key: 'medium_type' },
        ],
      },
      columns: [
        getCopyWithContentTableColumn({ field: 'name', title: '名称' }),
        getCopyWithContentTableColumn({ field: 'guest', title: '云服务器' }),
        {
          field: 'disk_type',
          title: '类型',
          width: 70,
          formatter: ({ cellValue }) => {
            return cellValue === 'sys' ? '系统盘' : '数据盘'
          },
        },
        getStatusTableColumn({ statusModule: 'disk' }),
        getProjectTableColumn(),
        getBrandTableColumn(),
        getTimeTableColumn({ field: 'auto_delete_at', title: '自动清除时间' }),
      ],
      groupActions: [
        {
          label: '清除',
          permission: 'disks_delete',
          action: () => {
            this.createDialog('DeleteResDialog', {
              data: this.list.selectedItems,
              columns: this.columns,
              title: '清除',
              list: this.list,
              requestParams: { override_pending_delete: true },
            })
          },
          meta: () => {
            return {
              validate: this.list.allowDelete(),
            }
          },
        },
        {
          label: '恢复',
          permission: 'disks_perform_cancel_delete',
          action: () => {
            this.createDialog('DiskRestoreDialog', {
              title: '恢复',
              data: this.list.selectedItems,
              columns: this.columns,
              list: this.list,
            })
          },
          meta: () => {
            return {
              validate: this.list.selectedItems.length > 0,
            }
          },
        },
      ],
      singleActions: [
        {
          label: '清除',
          permission: 'disks_delete',
          action: obj => {
            this.createDialog('DeleteResDialog', {
              data: [obj],
              columns: this.columns,
              title: '清除',
              list: this.list,
              requestParams: { override_pending_delete: true },
            })
          },
          meta: obj => {
            return {
              validate: obj.can_delete,
            }
          },
        },
        {
          label: '恢复',
          permission: 'disks_perform_cancel_delete',
          action: (obj) => {
            this.createDialog('DiskRestoreDialog', {
              data: [obj],
              columns: this.columns,
              list: this.list,
            })
          },
          meta: obj => {
            return {
              validate: obj.status !== 'deleting',
            }
          },
        },
      ],
    }
  },
  created () {
    this.list.fetchData()
  },
  methods: {
    getParams () {
      return {
        details: true,
        with_meta: true,
        pending_delete: true,
      }
    },
  },
}
</script>
