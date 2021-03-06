<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">{{action}}</div>
    <div slot="body">
      <a-alert class="mb-2" type="warning" v-if="tips">
        <div slot="message">
          {{ tips }}
        </div>
      </a-alert>
      <dialog-selected-tips :count="params.data.length" :action="action" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 3)" />
      <a-form
        :form="form.fc">
        <a-form-item v-bind="formItemLayout" v-show="!imgHidden" label="操作系统" extra="操作系统会根据选择的虚拟化平台和可用区域的变化而变化，公共镜像的维护请联系管理员">
          <os-select
            :type="type"
            :hypervisor="hypervisor"
            :image-params="image"
            :ignoreOptions="ignoreImageOptions"
            :osType="osType"
            :cache-image-params="cacheImageParams"
            :decorator="decorators.imageOS"
            @updateImageMsg="updateImageMsgDebounce" />
        </a-form-item>
        <a-form-item v-bind="formItemLayout" v-show="imgHidden" label="操作系统">
          <div>{{ imgHidden.text }}</div>
        </a-form-item>
        <a-form-item label="管理员密码" v-bind="formItemLayout" v-if="!isZStack">
          <server-password :decorator="decorators.loginConfig" :loginTypes="loginTypes" :form="form" />
        </a-form-item>
        <a-form-item label="自动启动" v-bind="formItemLayout" extra="重装系统后是否自动启动">
          <a-switch v-decorator="decorators.autoStart" />
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
import _ from 'lodash'
import OsSelect from '@Compute/sections/OsSelect'
import ServerPassword from '@Compute/sections/ServerPassword'
import { SERVER_TYPE, LOGIN_TYPES_MAP } from '@Compute/constants'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'
import { HYPERVISORS_MAP } from '@/constants'
import { Manager } from '@/utils/manager'
import { IMAGES_TYPE_MAP } from '@/constants/compute'
import { isRequired, passwordValidator } from '@/utils/validate'
import { findPlatform } from '@/utils/common/hypervisor'

export default {
  name: 'VmRebuildRootDialog',
  components: {
    OsSelect,
    ServerPassword,
  },
  provide: function () {
    return {
      form: this.form,
    }
  },
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
      action: '重装系统',
      form: {
        fc: this.$form.createForm(this),
        fd: {},
      },
      image: {
        limit: 0,
        scope: '',
        details: true,
        status: 'active',
        is_standard: true,
      },
      ignoreImageOptions: [
        IMAGES_TYPE_MAP.iso.key,
        IMAGES_TYPE_MAP.host.key,
        IMAGES_TYPE_MAP.snapshot.key,
      ],
      formItemLayout: {
        wrapperCol: {
          span: 21,
        },
        labelCol: {
          span: 3,
        },
      },
      detailData: {},
      firstLoad: true,
    }
  },
  computed: {
    type () {
      const brand = this.params.data[0].brand
      return findPlatform(brand)
    },
    imageType () {
      if (this.type === SERVER_TYPE.public) {
        return IMAGES_TYPE_MAP.public.key
      } else if (this.type === SERVER_TYPE.private) {
        return IMAGES_TYPE_MAP.private.key
      } else {
        return IMAGES_TYPE_MAP.standard.key
      }
    },
    hypervisor () {
      return this.params.data[0].hypervisor
    },
    isZStack () {
      return this.hypervisor === HYPERVISORS_MAP.zstack.key
    },
    decorators () {
      const validateToImage = (isZStack) => {
        return (rule, value, callback) => {
          if (isZStack) {
            callback()
          } else {
            isRequired()(rule, value, callback)
          }
        }
      }
      return {
        imageOS: {
          os: [
            'os',
            {
              initialValue: '',
              rules: [
                { required: !this.isZStack, message: '请选择操作系统' },
              ],
            },
          ],
          image: [
            'image',
            {
              initialValue: { key: '', label: '' },
              rules: [
                { validator: validateToImage(this.isZStack), message: '请选择镜像' },
              ],
            },
          ],
          imageType: [
            'imageType',
            {
              initialValue: this.imageType,
            },
          ],
        },
        loginConfig: {
          loginType: [
            'loginType',
            {
              initialValue: 'random',
            },
          ],
          keypair: [
            'loginKeypair',
            {
              initialValue: undefined, // { key: '', label: '' }
              rules: [
                { validator: isRequired(), message: '请选择关联密钥' },
              ],
            },
          ],
          password: [
            'loginPassword',
            {
              initialValue: '',
              rules: [
                { required: true, message: '请输入密码' },
                { validator: passwordValidator, trigger: 'blur' },
              ],
            },
          ],
        },
        autoStart: [
          'autoStart',
          {
            valuePropName: 'checked',
            initialValue: true,
          },
        ],
      }
    },
    cacheImageParams () {
      let params = {
        details: false,
        order_by: 'ref_count',
        order: 'desc',
        zone: this.params.data[0].zone_id,
      }
      return params
    },
    isFreezeImg () {
      // if (this.hypervisor) {
      //   return [HYPERVISORS_MAP.openstack.key].includes(this.hypervisor.toLowerCase())
      // }
      return false
    },
    imgHidden () {
      if (this.hypervisor) {
        if (this.isFreezeImg) {
          const pdata = this.detailData[0]
          if (pdata && pdata.disks_info && pdata.disks_info[0] && pdata.disks_info[0].image) {
            return {
              text: pdata.disks_info[0].image,
              isImage: true,
            }
          } else {
            return {
              text: '未找到系统盘镜像，无法重装系统',
              isImage: false,
            }
          }
        } else {
          return false
        }
      }
      return false
    },
    tips () {
      if (this.hypervisor === HYPERVISORS_MAP.openstack.key) {
        return '由于OpenStack本身原因，重装系统时可能会出现新密码没有生效现象，建议在重装系统后重置密码或使用原密码登录'
      }
      if (this.hypervisor === HYPERVISORS_MAP.zstack.key) {
        return '由于ZStack/DStack本身不支持重装系统设定新密码，您可以在重装系统完成后在主机列表进行密码重置'
      }
      return ''
    },
    osType () {
      return this.params.data[0].os_type
    },
    loginTypes () {
      const loginTypes = { ...LOGIN_TYPES_MAP }
      const hypervisor = this.hypervisor
      if (HYPERVISORS_MAP.ucloud.key === hypervisor) {
        delete loginTypes[LOGIN_TYPES_MAP.image.key]
        delete loginTypes[LOGIN_TYPES_MAP.keypair.key]
      }
      if (HYPERVISORS_MAP.aws.key === hypervisor) {
        delete loginTypes[LOGIN_TYPES_MAP.random.key]
        delete loginTypes[LOGIN_TYPES_MAP.password.key]
      }
      if ([HYPERVISORS_MAP.azure.key, HYPERVISORS_MAP.huawei.key].includes(hypervisor)) {
        delete loginTypes[LOGIN_TYPES_MAP.image.key]
      }
      if (this.osType === 'Windows') {
        // 以下平台在选择 windows 镜像时禁用关联密钥
        const disableKeypairHyper = [
          HYPERVISORS_MAP.azure.key,
          HYPERVISORS_MAP.aliyun.key,
          HYPERVISORS_MAP.qcloud.key,
          HYPERVISORS_MAP.esxi.key,
        ]
        if (disableKeypairHyper.includes(hypervisor)) {
          delete loginTypes[LOGIN_TYPES_MAP.keypair.value]
        }
      }
      return Object.keys(loginTypes)
    },
  },
  // watch: {
  //   type: {
  //     handler (val) {
  //       if (val === SERVER_TYPE.public) {
  //         this.$nextTick(() => {
  //           this.form.fc.setFieldsValue({ 'imageType': IMAGES_TYPE_MAP.public.key })
  //         })
  //       }
  //     },
  //     immediate: true,
  //   },
  // },
  created () {
    this.serversManager = new Manager('servers', 'v2')
    this.fetchData()
    this.updateImageMsgDebounce = _.debounce(this.updateImageMsg, 600)
  },
  methods: {
    async doRebuildRootSubmit (data) {
      const { autoStart, image, loginType, loginKeypair, loginPassword } = data
      const ids = this.params.data.map(item => item.id)
      const params = {
        reset_password: true,
        auto_start: autoStart,
        image_id: image.key,
      }
      // if (this.isZStack) {
      //   params.image_id = this.detailData[0].disks_info[0].image_id
      // }
      if (loginType === 'keypair') {
        params['keypair'] = loginKeypair.key
        params['reset_password'] = false
      } else if (loginType === 'image') {
        params['reset_password'] = false // 如果登录方式为创建后设置, 则增加参数 reset_password = false
      } else if (loginType === 'password') {
        params['password'] = loginPassword
      }
      return this.params.list.onManager('batchPerformAction', {
        steadyStatus: ['running', 'ready'],
        id: ids,
        managerArgs: {
          action: 'rebuild-root',
          data: params,
        },
      })
    },
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.form.fc.validateFields()
        await this.doRebuildRootSubmit(values)
        this.loading = false
        this.cancelDialog()
      } catch (error) {
        this.loading = false
      }
    },
    fetchData () {
      let ids = []
      this.params.data.forEach((item) => {
        ids.push(item.id)
      })
      this.fetchdone = false
      this.serversManager.batchGet({ id: ids })
        .then((res) => {
          this.fetchdone = true
          this.detailData = res.data.data
        })
        .catch(() => {
          this.fetchdone = true
        })
    },
    updateImageMsg (imageMsg) {
      if (this.firstLoad) {
        const detailData = this.detailData || []
        const diskInfo = (detailData[0] && detailData[0].disks_info) || []
        if (diskInfo.length > 0) {
          const image = diskInfo[0] || {}
          if (imageMsg.name !== image.image) return
          this.$nextTick(() => {
            this.form.fc.setFieldsValue({
              image: {
                key: image.image_id,
                label: image.image,
              },
            })
          })
        }
        this.firstLoad = false
      }
    },
  },
}
</script>
