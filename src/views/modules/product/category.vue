<template>
  <div>
    <div class="drop-control">
      <span>
        {{ dropData.isDrop ? "可拖拽" : "不可拖拽" }}：
        <el-switch
          v-model="dropData.isDrop"
          active-color="#13ce66"
          inactive-color="#909399"
        ></el-switch>
        &nbsp;
      </span>
      <el-button type="primary" round v-show="dropData.isDrop" size="mini" :disabled="dropData.action.length == 0"
      @click="saveCategorys"
        >点击保存</el-button
      >
    </div>
    <el-tree
      :data="data"
      :props="defaultProps"
      @node-click="handleNodeClick"
      show-checkbox
      :expand-on-click-node="false"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="dropData.isDrop"
      :allow-drop="isDrag"
      @node-drop="droped"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            v-if="node.level <= 2"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            type="text"
            v-if="node.data.children.length == 0"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="dialogTitle == 'add' ? '添加' : '编辑'"
      :visible.sync="dialogVisible"
      width="30%"
      :show-close="false"
    >
      <el-form
        ref="form"
        :model="category"
        @submit.native.prevent="addCategory"
      >
        <el-form-item label="分类名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="分类图标">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="unSumbit()">取 消</el-button>
        <el-button type="primary" @click="sumbitController()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data () {
    return {
      data: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
      expandedKey: [],
      // 添加弹框,默认不显示
      dialogVisible: false,
      category: { name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: '', productUnit: '' },
      defaultCategory: { name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: '', productUnit: '' },
      dialogTitle: '', // add edit
      dropData: {
        action: [],   // 保存拖拽的数据
        isDrop: false // 是否可拖拽
      }

    }
  },
  created () {
    this.getMenus()
  },
  methods: {
    // 确保拖拽后的树深度 <=3
    isDrag (draggingNode, dropNode, type) {
      // 默认展开,第一个值
      if (draggingNode.parent) {
        this.expandedKey = [ draggingNode.parent.data.catId ]
      }

      // console.log("Drag=>", draggingNode, dropNode, type)
      let draggingLevel = this.countNodeLevel(draggingNode) - draggingNode.level + 1
      let dropLevel = dropNode.level
      // console.log("拖的node深度：", draggingLevel, dropLevel)
      if (type === 'inner' && draggingLevel + dropLevel <= 3) {
        return true
      }
      if ((type === 'prev' || type === 'next') && draggingLevel + dropNode.parent.level <= 3) {
        return true
      }
      return false
    },
    // 拖拽成功放置后的回调
    droped (draggingNode, dropNode, type) {
      // 计算出要修改的，把要保存的数据，push到全局数组中
      // let parseChildren = draggingNode
      console.log(draggingNode, dropNode, type)
      let parent = null // 父节点
      // 这里存放本次成功拖拽要修改的category数据
      let updateCategorys = []
      // 获取父节点, 最上面有个root节点，是数据根的父节点
      if (type === 'inner') {
        parent = dropNode
      } else {
        parent = dropNode.parent
      }
      // 默认展开,第二个值
      this.expandedKey.push(parent.data.catId)
      // 获取所有子孙节点要修改的category数据
      for (let i = 0; i < parent.childNodes.length; i++) {
        // 【1】如果是拖拽的节点，它的sort,parentCid,level都将改变，收集成一个category子集对象
        if (parent.childNodes[i].data.catId === draggingNode.data.catId) {
          updateCategorys.push({ catId: parent.childNodes[i].data.catId, sort: i, parentCid: (parent.data.catId === undefined ? 0 : parent.data.catId), catLevel: parent.level + 1 })
          updateCategorys.push(...this.collectChildrenUpdateData(parent.childNodes[i]))
          continue
        }
        updateCategorys.push({catId: parent.childNodes[i].data.catId, sort: i})
      }
      // 将本次收集要修改的category数据push到全局容器中
      this.dropData.action.push(...updateCategorys)
    },
    // droped函数使用，用于收集子节点的数据
    collectChildrenUpdateData (rootNode) {
      console.log('rootNode', rootNode)
      let collectNode = []
      for (let i = 0; i < rootNode.childNodes.length; i++) {
        collectNode.push({ catId: rootNode.childNodes[i].data.catId, catLevel: rootNode.level + 1 })
        collectNode.push(...this.collectChildrenUpdateData(rootNode.childNodes[i]))
      }
      return collectNode
    },
    // 保存拖拽的顺序修改
    saveCategorys () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.dropData.action, false)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: '保存成功',
            type: 'success',
            duration: 1000,
            onClose: () => {
              this.getMenus()
              // 清空要保存的数据
              this.dropData.action = []
            }
          })
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    // 查看指定节点的深度
    countNodeLevel (node) {
      let maxLevel = node.level
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > maxLevel) {
            maxLevel = node.childNodes[i].level
          }
          let childLevel = this.countNodeLevel(node.childNodes[i])
          if (childLevel > maxLevel) {
            maxLevel = childLevel
          }
        }
      }
      return maxLevel
    },
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        // console.log("数据收到了", data.data);
        this.data = data.data
      })
    },
    handleNodeClick () { },
    unSumbit () {
      console.log('unSumbit ')
      this.category = this.defaultCategory
      this.dialogVisible = false
    },
    sumbitController () {
      if (this.dialogTitle === 'add') {
        this.addCategory()
      } else if (this.dialogTitle === 'edit') {
        this.editCategory()
      }
    },
    append (data) {
      // 清除输入框
      this.category = { ...this.defaultCategory }
      // 绑定好添加的category的数据
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel + 1
      // 修改默认展开
      this.expandedKey = [data.catId]
      // 修改弹框标题
      this.dialogTitle = 'add'
      // 打开输入弹框
      this.dialogVisible = true
    },
    addCategory () {
      console.log('category:', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: '保存成功',
            type: 'success',
            duration: 1000,
            onClose: () => {
              // 关闭输入弹框
              this.dialogVisible = false
              this.getMenus()
              // 清除输入框
              this.category = { ...this.defaultCategory }
            }
          })
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    edit (data) {
      this.dialogTitle = 'edit'
      // 修改默认展开
      this.expandedKey = [data.parentCid]
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({ data }) => {
        if (data && data.code === 0) {
          // 进行数据回显
          this.category = data.category
          // 打开对话弹框
          this.dialogVisible = true
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    editCategory () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: '编辑成功',
            type: 'success',
            duration: 1000,
            onClose: () => {
              // 关闭输入弹框
              this.dialogVisible = false
              this.getMenus()
              // 清除输入框
              this.category = { ...this.defaultCategory }
            }
          })
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    remove (node, data) {
      console.log(node, data)
      let ids = [data.catId]
      this.$confirm(`是否删除【${data.name}】当前菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({ data }) => {
          if (data && data.code === 0) {
            this.$message({
              message: '操作成功',
              type: 'success',
              duration: 1000,
              onClose: () => {
                this.getMenus()
                this.expandedKey = [node.parent.data.catId]
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => { })
    }
  }
}
</script>

<style>
.drop-control {
  display: flex;
  align-items: center;
  height: 40px;
}
</style>