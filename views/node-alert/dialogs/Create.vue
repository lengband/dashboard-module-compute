<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">新建报警</div>
    <div slot="body">
      <vxe-grid class="mb-2" v-if="params.data" :data="params.data" :columns="params.columns.slice(0, 2)" />
      <node-alert-form
        ref="nodeAlertFormRef"
        :metric-opts="metricOptsEnabled"
        :hypervisor="params.hypervisor"
        :alertType="params.alertType"
        :monitor-metric="params.metric" />
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import NodeAlertForm from '../components/Form'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'CreateNodeAlert',
  components: {
    NodeAlertForm,
  },
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
    }
  },
  computed: {
    metricOptsEnabled () {
      let opts = this.params.metricOpts
      if (this.params.list && this.params.list.data) {
        // 新建时要保证新建的报警不会和已有的重复，下面是过滤掉已有的metric
        let listDataArr = Object.values(this.params.list.data)
        if (listDataArr && listDataArr.length) {
          opts = opts.filter(val => {
            return !listDataArr.find(dataWrap => dataWrap.data.metric === val.key)
          })
        }
      }
      return opts
    },
  },
  methods: {
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.$refs.nodeAlertFormRef.validateForm()
        this.loading = false
        const recipients = values.recipients.join(',')
        const data = {
          metric: values.metric,
          window: `${values.window}m`,
          comparator: values.comparator,
          threshold: values.threshold + '',
          level: values.level,
          channel: values.channel,
          node_id: this.params.nodeId,
          type: this.params.alertType,
          recipients,
        }
        if (this.params.list && this.params.list.onManager) {
          await this.params.list.onManager('create', { managerArgs: { data } })
        } else {
          await this.params.alertManager.create({
            data,
          })
        }
        this.cancelDialog()
        this.$message.success('操作成功')
      } catch (error) {
        this.loading = false
        throw error
      }
    },
  },
}
</script>
