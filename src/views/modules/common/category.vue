<template>
  <div>

    <el-tree :data="data" :props="defaultProps" @node-click="handleNodeClick" :expand-on-click-node="false"
      node-key="catId" :default-expanded-keys="expandedKey" ref="categoryTree">

    </el-tree>

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
      expandedKey: []
    }
  },
  created () {
    this.getMenus()
  },
  methods: {

    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        // console.log("数据收到了", data.data);
        this.data = data.data
      })
    },
    handleNodeClick (data, node, vc) {
      // console.log(data, node, vc)
      this.$emit('tree-node-click', data, node, vc)
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