<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">绑定密钥</div>
    <div slot="body">
      <dialog-selected-tips :count="params.data.length" action="绑定密钥" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 3)" />
      <a-form :form="form.fc" hideRequiredMark>
        <a-form-item label="密钥对" v-bind="formItemLayout">
          <base-select
            v-decorator="decorators.keypair"
            resource="keypairs"
            :select-props="{ allowClear: true, placeholder: '请选择关密钥对' }" />
        </a-form-item>
        <a-form-item label="自动启动" v-bind="formItemLayout" extra="绑定密钥成功后自动启动">
          <a-switch v-decorator="decorators.auto_start" :disabled="form.fi.disableAutoStart" />
        </a-form-item>
      </a-form>
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'
import { typeClouds } from '@/utils/common/hypervisor'

const hypervisorMap = typeClouds.hypervisorMap

export default {
  name: 'VmBindKeypairDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    let autoStartInitialValue = true
    let disableAutoStart = false
    const firstData = this.params.data && this.params.data[0]
    if (firstData && (firstData.status === 'running' || firstData.hypervisor === hypervisorMap.azure.key)) {
      autoStartInitialValue = false
      disableAutoStart = true
    }
    return {
      loading: false,
      form: {
        fc: this.$form.createForm(this),
        fi: {
          disableAutoStart,
        },
      },
      decorators: {
        keypair: [
          'keypair',
          {
            rules: [
              { required: true, message: '请选择关联密钥', trigger: 'blur' },
            ],
          },
        ],
        auto_start: [
          'auto_start',
          {
            initialValue: autoStartInitialValue,
            valuePropName: 'checked',
          },
        ],
      },
      formItemLayout: {
        wrapperCol: {
          span: 21,
        },
        labelCol: {
          span: 3,
        },
      },
    }
  },
  methods: {
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.form.fc.validateFields()
        const ids = this.params.data.map(item => item.id)
        await this.params.list.onManager('batchPerformAction', {
          id: ids,
          steadyStatus: ['running', 'ready'],
          managerArgs: {
            action: 'deploy',
            data: values,
          },
        })
        this.$bus.$emit('VMInstanceListSingleUpdate', [this.params.data[0]['id']])
        this.cancelDialog()
      } finally {
        this.loading = false
      }
    },
  },
}
</script>
