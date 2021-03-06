<template>
  <div v-loading="loading">
    <div v-empty="list.length === 0">
      <div class="log-items">
        <examine-cell v-for="(item, index) in list"
                      :key="index"
                      :data="item"
                      @on-handle="examineCellHandle"></examine-cell>
        <div class="load">
          <el-button type="text"
                     :loading="loadMoreLoading">{{loadMoreLoading ? '加载更多' : '没有更多了'}}</el-button>
        </div>
      </div>
    </div>
    <examine-handle :show="showExamineHandle"
                    @close="showExamineHandle = false"
                    @save="refreshList"
                    :id="rowID"
                    examineType="oa_examine"
                    status="2"></examine-handle>
    <examine-create-view v-if="isCreate"
                         :categoryId="createInfo.categoryId"
                         :categoryTitle="createInfo.title"
                         :action="createAction"
                         @save-success="refreshList"
                         @hiden-view="isCreate=false"></examine-create-view>
    <c-r-m-full-screen-detail :visible.sync="showFullDetail"
                              :crmType="detailCRMType"
                              :id="rowID"></c-r-m-full-screen-detail>
  </div>
</template>

<script>
import ExamineCell from '@/views/OAManagement/examine/components/examineCell' // 跟进记录
import ExamineCreateView from '@/views/OAManagement/examine/components/examineCreateView'
import { crmQueryExamineRelation } from '@/api/customermanagement/common'
import { oaExamineDelete } from '@/api/oamanagement/examine'
import { formatTimeToTimestamp } from '@/utils'

export default {
  /** 审批 跟进记录*/
  name: 'examine-log',
  components: {
    ExamineCell,
    CRMFullScreenDetail: () =>
      import('@/views/customermanagement/components/CRMFullScreenDetail.vue'),
    ExamineHandle: () => import('@/components/Examine/ExamineHandle'),
    ExamineCreateView
  },
  props: {
    /** 模块ID */
    id: [String, Number],
    /** 没有值就是全部类型 有值就是当个类型 */
    crmType: {
      type: String,
      default: ''
    }
  },
  watch: {
    id: function(val) {
      this.refreshList()
    }
  },
  data() {
    return {
      loading: false,
      loadMoreLoading: true,
      isPost: false,
      page: 1,
      list: [], // 跟进记录列表
      rowID: '', // 行信息
      // 撤回操作
      showExamineHandle: false,
      isCreate: false, //是编辑
      createAction: { type: 'save' },
      createInfo: {}, // 编辑所需要的id 标题名信息
      // 详情
      showFullDetail: false,
      detailCRMType: ''
    }
  },
  computed: {},
  mounted() {
    // 分批次加载
    let self = this
    let item = document.getElementById('follow-log-content')
    item.onscroll = function() {
      let scrollTop = item.scrollTop
      let windowHeight = item.clientHeight
      let scrollHeight = item.scrollHeight //滚动条到底部的条件

      if (
        scrollTop + windowHeight == scrollHeight &&
        self.loadMoreLoading == true
      ) {
        if (!self.isPost) {
          self.isPost = true
          self.page++
          self.getList()
        } else {
          self.loadMoreLoading = false
        }
      }
    }

    this.getList()
  },
  activated: function() {},
  deactivated: function() {},
  methods: {
    getList() {
      this.loading = true
      let params = { page: this.page, limit: 10 }
      params[this.crmType + 'Ids'] = this.id
      crmQueryExamineRelation(params)
        .then(res => {
          this.list = this.list.concat(res.data.list)
          if (res.data.list.length < 10) {
            this.loadMoreLoading = false
          } else {
            this.loadMoreLoading = true
          }
          this.loading = false
          this.isPost = false
        })
        .catch(() => {
          this.isPost = false
          this.loading = false
        })
    },
    refreshList() {
      this.page = 1
      this.list = []
      this.getList()
    },
    examineCellHandle(data) {
      // 编辑
      if (data.type == 'edit') {
        data.data.item.title = data.data.item.categoryName
        this.createInfo = data.data.item
        this.createAction = {
          type: 'update',
          id: data.data.item.examineId,
          data: data.data.item
        }
        this.isCreate = true
        // 删除
      } else if (data.type == 'delete') {
        this.$confirm('确定删除?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
          .then(() => {
            oaExamineDelete({
              examineId: data.data.item.examineId
            }).then(res => {
              this.refreshList()
              this.$message({
                type: 'success',
                message: '删除成功!'
              })
            })
          })
          .catch(() => {
            this.$message({
              type: 'info',
              message: '已取消删除'
            })
          })
        // 撤回
      } else if (data.type == 'withdraw') {
        this.rowID = data.data.item.examineId
        this.showExamineHandle = true
        // 详情
      } else if (data.type == 'view') {
        this.detailCRMType = 'examine'
        this.rowID = data.data.item.examineId
        this.showFullDetail = true
      } else if (data.type == 'related-detail') {
        this.rowID = data.data.item[data.data.type + 'Id']
        this.detailCRMType = data.data.type
        this.showFullDetail = true
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.log-items {
  min-height: 400px;
  position: relative;
  padding: 10px 20px;
}

.load {
  color: #999;
  font-size: 13px;
  margin: 0 auto 15px;
  text-align: center;
  .el-button,
  .el-button:focus {
    color: #ccc;
    cursor: auto;
  }
}
</style>
