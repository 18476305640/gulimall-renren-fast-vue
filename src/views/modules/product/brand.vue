<template>
  <div class="mod-config">
    <el-form :inline="true" :model="dataForm" @keyup.enter.native="getDataList()">
      <el-form-item>
        <el-input v-model="dataForm.key" placeholder="参数名" clearable></el-input>
      </el-form-item>
      <el-form-item>
        <el-button @click="getDataList()">查询</el-button>
        <el-button v-if="isAuth('pms:brand:save')" type="primary" @click="addOrUpdateHandle()">新增</el-button>
        <el-button v-if="isAuth('pms:brand:delete')" type="danger" @click="deleteHandle()"
          :disabled="dataListSelections.length <= 0">批量删除</el-button>
      </el-form-item>
    </el-form>
    <el-table :data="dataList" border v-loading="dataListLoading" @selection-change="selectionChangeHandle"
      style="width: 100%;">
      <el-table-column type="selection" header-align="center" align="center" width="50">
      </el-table-column>
      <el-table-column prop="brandId" header-align="center" align="center" label="品牌id">
      </el-table-column>
      <el-table-column prop="name" header-align="center" align="center" label="品牌名">
      </el-table-column>
      <el-table-column prop="logo" header-align="center" align="center" label="品牌logo地址">
        <template slot-scope="scope">
          <img :src="scope.row.logo.indexOf('http') >= 0 ? scope.row.logo : defaultLogo"
            style="width: 60px; height: 60px" />
        </template>
      </el-table-column>
      <el-table-column prop="descript" header-align="center" align="center" label="介绍" width="170px">
      </el-table-column>
      <el-table-column prop="showStatus" header-align="center" align="center" label="显示状态">
        <template slot-scope="scope">
          <el-switch v-model="scope.row.showStatus" active-color="#13ce66" inactive-color="#ff4949" :active-value="1"
            :inactive-value="0" @change="statusChanged(scope.row)">
          </el-switch>
        </template>
      </el-table-column>
      <el-table-column prop="firstLetter" header-align="center" align="center" label="检索首字母">
      </el-table-column>
      <el-table-column prop="sort" header-align="center" align="center" label="排序">
      </el-table-column>
      <el-table-column fixed="right" header-align="center" align="center" width="170" label="操作">
        <template slot-scope="scope">
          <el-button type="text" size="small" @click="openRelation(scope.row.brandId)">关联分类</el-button>
          <el-button type="text" size="small" @click="addOrUpdateHandle(scope.row.brandId)">修改</el-button>
          <el-button type="text" size="small" @click="deleteHandle(scope.row.brandId)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination @size-change="sizeChangeHandle" @current-change="currentChangeHandle" :current-page="pageIndex"
      :page-sizes="[10, 20, 50, 100]" :page-size="pageSize" :total="totalPage"
      layout="total, sizes, prev, pager, next, jumper">
    </el-pagination>
    <!-- 弹窗, 新增 / 修改 -->
    <add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="getDataList"></add-or-update>

    <el-dialog title="品牌关联分类" :visible.sync="relationDialog" width="40%" :before-close="null">
      <el-cascader v-model="relationCategoryPathIds" :options="cascaderData" :props="cascaderProps"
        @change="cascaderChangeEven" placeholder="添加分类" :filterable="true"></el-cascader>

      <el-table :data="currentBrandRelationCategorys" border style="width: 100%;">
        <el-table-column prop="id" label="#">
        </el-table-column>
        <el-table-column prop="brandName" label="品牌名">
        </el-table-column>
        <el-table-column prop="categoryName" label="分类名">
        </el-table-column>
        <el-table-column fixed="right" label="操作" width="120">
          <template slot-scope="scope">
            <el-button @click.native.prevent="deleteRelation(scope)" type="text" size="small">
              移除
            </el-button>
          </template>
        </el-table-column>
      </el-table>

      <span slot="footer" class="dialog-footer">
        <el-button @click="relationDialog = false">关 闭</el-button>
      </span>
    </el-dialog>

  </div>
</template>

<script>
import AddOrUpdate from './brand-add-or-update'
export default {
  data () {
    return {
      currentBrandId: null,
      cascaderProps: {
        value: 'catId',
        label: 'name'
      },
      cascaderData: [],
      currentBrandRelationCategorys: [],
      dataForm: {
        key: ''
      },
      relationCategoryPathIds: [],
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalPage: 0,
      dataListLoading: false,
      dataListSelections: [],
      addOrUpdateVisible: false,
      defaultLogo: '',
      relationDialog: false

    }
  },
  components: {
    AddOrUpdate
  },
  activated () {
    this.getDataList()
  },
  methods: {
    deleteRelation (scope) {
      console.log(scope.row)
      this.$confirm(`删除“${scope.row.brandName}”（品牌）->“${scope.row.categoryName}”（分类）关联, 是否继续？`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/categorybrandrelation/delete'),
          method: 'post',
          data: [scope.row.id]
        }).then(({ data }) => {
          if (data && data.code === 0) {
            this.$message({
              message: '关联删除成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getBrandRelationCategorys()
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      })
    },
    cascaderChangeEven (catelogPathIds) {
      let categoryId = catelogPathIds[catelogPathIds.length - 1]
      let brandId = this.currentBrandId
      // 清除选择记录
      this.relationCategoryPathIds = []
      for (let i = 0; i < this.currentBrandRelationCategorys.length; i++) {
        let item = this.currentBrandRelationCategorys[i]
        console.log(item.categoryId === categoryId, item.brandId === brandId)
        if (item.categoryId === categoryId && item.brandId === brandId) {
          this.$message.error('已关联')
          return
        }
      }
      this.$http({
        url: this.$http.adornUrl('/product/categorybrandrelation/save'),
        method: 'post',
        data: { 'brandId': brandId, 'categoryId': categoryId }
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: '关联分类成功',
            type: 'success',
            duration: 1500,
            onClose: () => {
              this.getBrandRelationCategorys()
            }
          })
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    getBrandRelationCategorys () {
      // 请求当前品牌关联的分类
      this.$http({
        url: this.$http.adornUrl(`/product/categorybrandrelation/list?brandId=${this.currentBrandId}`),
        method: 'get'
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.currentBrandRelationCategorys = data.data
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    openRelation (brandId) {
      this.currentBrandId = brandId
      this.relationDialog = true
      this.getBrandRelationCategorys()
    },
    statusChanged (rowData) {
      this.$http({
        url: this.$http.adornUrl('/product/brand/update'),
        method: 'post',
        data: this.$http.adornData(rowData, false)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: '状态修改成功',
            type: 'success',
            duration: 1500,
            onClose: () => {
              this.getDataList()
            }
          })
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    // 获取数据列表
    getDataList () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/product/brand/list'),
        method: 'get',
        params: this.$http.adornParams({
          'page': this.pageIndex,
          'limit': this.pageSize,
          'key': this.dataForm.key
        })
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.dataList = data.page.list
          this.totalPage = data.page.totalCount
        } else {
          this.dataList = []
          this.totalPage = 0
        }
        this.dataListLoading = false
      })
    },
    // 每页数
    sizeChangeHandle (val) {
      this.pageSize = val
      this.pageIndex = 1
      this.getDataList()
    },
    // 当前页
    currentChangeHandle (val) {
      this.pageIndex = val
      this.getDataList()
    },
    // 多选
    selectionChangeHandle (val) {
      this.dataListSelections = val
    },
    // 新增 / 修改
    addOrUpdateHandle (id) {
      this.addOrUpdateVisible = true
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(id)
      })
    },
    // 删除
    deleteHandle (id) {
      var ids = id ? [id] : this.dataListSelections.map(item => {
        return item.brandId
      })
      this.$confirm(`确定对[id=${ids.join(',')}]进行[${id ? '删除' : '批量删除'}]操作?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/brand/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({ data }) => {
          if (data && data.code === 0) {
            this.$message({
              message: '操作成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getDataList()
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      })
    }
  },
  created () {
    this.$http({
      url: this.$http.adornUrl('/product/category/list/tree'),
      method: 'get'
    }).then(({ data }) => {
      // 使用递归，清除长度为0的children属性
      function clearNullChildrenArray (childArr) {
        for (let i = 0; i < childArr.length; i++) {
          if (childArr[i].children == null || childArr[i].children.length === 0) {
            delete childArr[i].children
          } else {
            clearNullChildrenArray(childArr[i].children)
          }
        }
      }
      // 开始递归
      console.log(data.data)
      clearNullChildrenArray(data.data)
      this.cascaderData = data.data
    })
  }
}
</script>
