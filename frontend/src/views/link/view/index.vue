<template>
  <div
    v-loading="dataLoading"
    style="width: 100%;height: 100vh;background-color: #f7f8fa"
    :element-loading-text="$t('panel.data_loading')"
    element-loading-spinner="el-icon-loading"
    element-loading-background="rgba(220,220,220,1)"
  >
    <Preview
      v-if="show"
      :component-data="mainCanvasComponentData"
      :canvas-style-data="canvasStyleData"
      :panel-info="panelInfo"
      :user-id="user"
      @change-load-status="setLoading"
    />
  </div>
</template>

<script>
import { loadResource, viewLinkLog } from '@/api/link'
import { isMobile } from '@/utils/index'
import { uuid } from 'vue-uuid'
import Preview from '@/components/canvas/components/editor/Preview'
import { getPanelAllLinkageInfo } from '@/api/panel/linkage'
import { queryPanelJumpInfo, queryTargetPanelJumpInfo } from '@/api/panel/linkJump'
import { getNowCanvasComponentData, panelInit } from '@/components/canvas/utils/utils'
import { getOuterParamsInfo } from '@/api/panel/outerParams'
import { mapState } from 'vuex'

export default {
  name: 'LinkView',
  components: { Preview },
  props: {
    resourceId: {
      type: String,
      default: null
    },
    user: {
      type: String,
      default: null
    }
  },
  data() {
    return {
      canvasId: 'canvas-main',
      show: false,
      panelInfo: {},
      dataLoading: false
    }
  },
  computed: {
    mainCanvasComponentData() {
      return getNowCanvasComponentData(this.canvasId)
    },
    ...mapState([
      'canvasStyleData'
    ])
  },
  created() {
    this.show = false
    this.setPanelInfo()
    this.viewLog()
  },
  methods: {
    setLoading(status) {
      this.dataLoading = !!status
    },
    viewLog() {
      const param = {
        panelId: this.resourceId,
        userId: this.user,
        mobile: !!isMobile()
      }
      viewLinkLog(param).then(res => {

      })
    },
    setPanelInfo() {
      loadResource(this.resourceId).then(res => {
        this.show = false
        let loadingCount = 0
        const watermarkInfo = {
          ...res.data.watermarkInfo,
          settingContent: JSON.parse(res.data.watermarkInfo.settingContent)
        }
        this.panelInfo = {
          id: res.data.id,
          name: res.data.name,
          privileges: res.data.privileges,
          status: res.data.status,
          createBy: res.data.createBy,
          createTime: res.data.createTime,
          updateBy: res.data.updateBy,
          updateTime: res.data.updateTime,
          watermarkOpen: res.data.watermarkOpen,
          watermarkInfo: watermarkInfo
        }
        this.$store.dispatch('panel/setPanelInfo', this.panelInfo)

        panelInit(JSON.parse(res.data.panelData), JSON.parse(res.data.panelStyle))
        // ???????????????title????????????????????????
        document.title = res.data.name
        // ??????????????????
        getPanelAllLinkageInfo(this.resourceId).then(rsp => {
          this.$store.commit('setNowPanelTrackInfo', rsp.data)
        })
        // ??????????????????
        queryPanelJumpInfo(this.resourceId).then(rsp => {
          this.$store.commit('setNowPanelJumpInfo', rsp.data)
        })

        // ???????????????????????? ????????????
        const tempParam = localStorage.getItem('jumpInfoParam')
        // ??????????????????
        const attachParamsEncode = this.$route.query.attachParams

        tempParam && loadingCount++
        attachParamsEncode && loadingCount++

        if (attachParamsEncode) {
          try {
            const Base64 = require('js-base64').Base64
            const attachParam = JSON.parse(decodeURIComponent(Base64.decode(attachParamsEncode)))
            getOuterParamsInfo(this.resourceId).then(rsp => {
              if (--loadingCount === 0) {
                this.show = true
              }
              this.$store.commit('setNowPanelOuterParamsInfo', rsp.data)
              this.$store.commit('addOuterParamsFilter', attachParam)
            })
          } catch (e) {
            if (--loadingCount === 0) {
              this.show = true
            }

            this.$message({
              message: this.$t('panel.outer_param_decode_error'),
              type: 'error'
            })
            console.error('outerParams Decode error???', e)
          }
        }

        if (tempParam) {
          try {
            localStorage.removeItem('jumpInfoParam')
            const jumpParam = JSON.parse(tempParam)
            const jumpRequestParam = {
              sourcePanelId: jumpParam.sourcePanelId,
              sourceViewId: jumpParam.sourceViewId,
              sourceFieldId: jumpParam.sourceFieldId,
              targetPanelId: this.resourceId
            }
            if (jumpParam.sourceType && jumpParam.sourceType === 'table-pivot') {
              jumpRequestParam.sourceFieldId = null
            }
            // ???????????????????????????????????????
            queryTargetPanelJumpInfo(jumpRequestParam).then(rsp => {
              this.show = true
              this.$store.commit('setNowTargetPanelJumpInfo', rsp.data)
              this.$store.commit('addViewTrackFilter', jumpParam)
            })
          } catch (e) {
            if (--loadingCount === 0) {
              this.show = true
            }
            console.error('tempParam error???', e)
          }
        }

        if (loadingCount === 0) {
          this.show = true
        }
      })
    },
    resetID(data) {
      if (data) {
        data.forEach(item => {
          item.type !== 'custom' && (item.id = uuid.v1())
          // item.user = this.user
          item.fromSource = 'link'
        })
      }
      return data
    }

  }
}
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
}
</style>
