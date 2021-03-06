<template>
  <div>
    <page-header title="部署服务" />
    <page-card-detail
      class="mt-3"
      v-if="catalogData.name"
      :img="catalogData.icon_url"
      :page-title="catalogData.name"
      :description="catalogData.description" />
    <a-form :form="form.fc" class="mt-3">
      <a-form-item label="主机名称" v-bind="formItemLayout" extra="名称支持序号占位符‘#’，用法如下。 名称：host## 数量：2、实例为：host01、host02">
        <a-input v-decorator="decorators.name" :placeholder="$t('validator.serverCreateName')" />
      </a-form-item>
      <a-form-item label="申请原因" v-bind="formItemLayout" v-if="isOpenWorkflow">
        <a-input v-decorator="decorators.reason" placeholder="请输入主机申请原因" />
      </a-form-item>
      <page-footer>
        <template v-slot:right>
          <a-button
            size="large"
            type="primary"
            class="mr-2"
            @click="submit"
            style="width: 120px;"
            :loading="loading">{{ isOpenWorkflow ? '提交工单' : $t('dialog.ok') }}</a-button>
          <a-button size="large" style="width: 120px;" @click="goBack">取消</a-button>
          <side-errors error-title="创建主机失败" :errors.sync="errors" />
        </template>
      </page-footer>
    </a-form>
  </div>
</template>

<script>
import { GenCreateData } from '@Compute/utils/createServer'
import WindowsMixin from '@/mixins/windows'
import workflowMixin from '@/mixins/workflow'
import { WORKFLOW_TYPES } from '@/constants/workflow'
import SideErrors from '@/sections/SideErrors'

export default {
  name: 'ServicecatalogDeploy',
  components: {
    SideErrors,
  },
  mixins: [WindowsMixin, workflowMixin],
  data () {
    return {
      loading: false,
      catalogData: {},
      serverConfig: null,
      errors: [],
      form: {
        fc: this.$form.createForm(this),
      },
      decorators: {
        name: [
          'name',
          {
            validateFirst: true,
            rules: [
              { required: true, message: '请输入主机名称' },
              { validator: this.$validate('serverCreateName') },
            ],
          },
        ],
        reason: [
          'reason',
        ],
      },
      formItemLayout: {
        wrapperCol: {
          span: 20,
        },
        labelCol: {
          span: 4,
        },
      },
    }
  },
  computed: {
    isOpenWorkflow () {
      return this.checkWorkflowEnabled(WORKFLOW_TYPES.APPLY_MACHINE)
    },
  },
  created () {
    this.fetchServicecatalog()
  },
  methods: {
    fetchServicecatalog () {
      new this.$Manager('servicecatalogs', 'v2')
        .get({ id: this.$route.query.id })
        .then(({ data }) => {
          this.catalogData = data
          this.fetchServerConfig(data.guest_template_id)
        })
        .catch(() => {
          this.$message.error('获取主机模板数据失败，无法完成部署')
        })
    },
    fetchServerConfig (id) {
      new this.$Manager('servertemplates', 'v2')
        .get({ id })
        .then(({ data }) => {
          this.serverConfig = data.content
        })
        .catch(() => {
          this.$message.error('获取主机模板数据失败，无法完成部署')
        })
    },
    doCreateWorkflow (values) {
      const params = {
        ...this.serverConfig,
        generate_name: values.name,
      }
      delete params.reason
      const variables = {
        process_definition_key: WORKFLOW_TYPES.APPLY_MACHINE,
        initiator: this.$store.getters.userInfo.id,
        description: values.reason,
        'server-create-paramter': JSON.stringify(params),
        project: values.project_id,
      }
      // this._getProjectDomainInfo(variables) // !!! project_domain 暂时不加，因为后端可以从token里面获取
      return new this.$Manager('process-instances', 'v1')
        .create({ data: { variables } })
        .then(() => {
          this.$message.success(`主机 ${params.generate_name} 创建请求流程已提交`)
          this.goWorkflow()
        })
    },
    doForecast (fd) {
      const genCreateData = new GenCreateData()
      const params = { ...fd, ...this.serverConfig }
      return new this.$Manager('schedulers', 'v1').rpc({ methodname: 'DoForecast', params })
        .then(res => {
          if (res.data.can_create) {
            this.createServer(params)
          } else {
            this.errors = genCreateData.getForecastErrors(res.data)
          }
        })
        .catch(err => {
          this.$message.error(`创建失败: ${err}`)
        })
    },
    createServer (data) {
      delete data['vcpu_count']
      delete data['vmem_size']
      new this.$Manager('servers', 'v2').create({ data })
        .then(res => {
          this.$message.success('操作成功，开始创建')
          this.goVminstance()
        })
    },
    async submit () {
      this.loading = true
      try {
        let values = await this.form.fc.validateFields()
        if (this.isOpenWorkflow) {
          await this.doCreateWorkflow(values)
        } else {
          await this.doForecast(values)
        }
        this.loading = false
      } catch (error) {
        this.loading = false
        throw error
      }
    },
    goWorkflow () {
      window.location.href = this.$appConfig.v1Perfix + '/workflow?type=me-process'
    },
    goBack () {
      this.$router.push('/servicecatalog')
    },
    goVminstance () {
      this.$router.push('/vminstance')
    },
  },
}
</script>
