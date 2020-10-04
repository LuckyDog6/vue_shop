<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>商品管理</el-breadcrumb-item>
      <el-breadcrumb-item>商品列表</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片试图区域 -->
    <el-card>
      <el-row>
        <el-col>
          <el-button type="primary" @click="showAddCateDialog">添加分类</el-button>
        </el-col>
      </el-row>
      <!-- 表格 -->
      <tree-table class="treeTable" :data="catelist" :columns="columns" :selection-type="false"
      :expand-type="false" :show-index="true" index-text="#" border
      :show-row-hover="false">
      <!-- 是否有效 -->
        <template slot="isok" slot-scope="scope">
          <i class="el-icon-success" v-if="scope.row.cat_deleted===false"
          style="color:lightgreen"></i>
          <i class="el-icon-error" v-else
          style="color:red"></i>
        </template>
        <!-- 排序 -->
        <template slot="order" slot-scope="scope">
          <el-tag v-if="scope.row.cat_level===0">一级</el-tag>
          <el-tag type="success" v-else-if="scope.row.cat_level===1">二级</el-tag>
          <el-tag type="warning" v-else>三级</el-tag>
        </template>
        <!-- 操作 -->
        <template slot="opt" slot-scope="scope">
          <el-button type="primary" icon="el-icon-edit" size="mini" @click="showEditDialog(scope.row.cat_id)">编辑</el-button>
          <el-button type="danger" icon="el-icon-delete" size="mini" @click="removeUserById(scope.row.cat_id)">删除</el-button>
        </template>
      </tree-table>
      <!-- 分页区域 -->
      <el-pagination
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="queryInfo.pagenum"
      :page-sizes="[3, 5, 10, 15]"
      :page-size="queryInfo.pagesize"
      layout="total, sizes, prev, pager, next, jumper"
      :total="total">
    </el-pagination>
    </el-card>
    <!-- 添加分类的对话框 -->
    <el-dialog
    title="添加分类"
    :visible.sync="addCateDialogVisible"
    width="50%"
    @close="addCateDialogClosed"
    >
    <!-- 添加分类表单 -->
    <el-form :model="addCateForm" :rules="addCateFormRules" ref="addCateFormRef" label-width="100px">
      <el-form-item label="分类名称" prop="cat_name">
        <el-input v-model="addCateForm.cat_name"></el-input>
      </el-form-item>
      <el-form-item label="父级分类">
        <!-- options 用来指定数据源 -->
        <!-- props 用来指定配置对象 -->
        <el-cascader
        v-model="selectedKeys"
        :options="parentCateList"
        :props="cascaderProps"
        @change="parentCartChange"
        clearable change-on-select>
        </el-cascader>
      </el-form-item>
    </el-form>
    <span slot="footer" class="dialog-footer">
      <el-button @click="addCateDialogVisible = false">取 消</el-button>
      <el-button type="primary" @click="addCate">确 定</el-button>
    </span>
    </el-dialog>
    <!-- 修改角色对话框 -->
    <el-dialog
    title="修改分类"
    :visible.sync="editDialogVisible"
    width="50%"
    @close="editDialogClosed">
    <el-form :model="editForm" :rules="addFormRules" ref="editFormRef" label-width="70px">
      <el-form-item label="商品id">
        <el-input v-model="editForm.cat_id" disabled></el-input>
      </el-form-item>
      <el-form-item label="商品名">
        <el-input v-model="editForm.cat_name"></el-input>
      </el-form-item>
    </el-form>
    <span slot="footer" class="dialog-footer">
      <el-button @click="editDialogVisible = false">取 消</el-button>
      <el-button type="primary" @click="editUserInfo">确 定</el-button>
    </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 查询条件
      queryInfo: {
        type: 3,
        pagenum: 1,
        pagesize: 5
      },
      // 商品分类的数据列表,默认为空
      catelist: [],
      // 总数据条数
      total: 0,
      // 为table指定列的定义
      columns: [{
        label: '分类名称',
        prop: 'cat_name'
      },
      {
        label: '是否有效',
        // 表示将当前列定义为模板列
        type: 'template',
        template: 'isok'
      },
      {
        label: '排序',
        // 表示将当前列定义为模板列
        type: 'template',
        template: 'order'
      },
      {
        label: '操作',
        // 表示将当前列定义为模板列
        type: 'template',
        template: 'opt'
      }
      ],
      // 控制添加分类对话框的影藏
      addCateDialogVisible: false,
      // 添加分类的表单数据对象
      addCateForm: {
        // 将要添加的分类名称
        cat_name: '',
        // 腹肌分类的id
        cat_pid: 0,
        // 分类等级,默认是一级分类
        cat_level: 0
      },
      // 添加分类表单的验证规则对象
      addCateFormRules: {
        cat_name: [
          { required: true, message: '请输入分类名称', trigger: 'blur' }
        ]
      },
      // 父级分类列表
      parentCateList: [],
      // 指定级联选择器的配置对象
      cascaderProps: {
        value: 'cat_id',
        label: 'cat_name',
        children: 'children',
        expandTrigger: 'hover'
      },
      // 选中的父级分类的id数组
      selectedKeys: [],
      // 查询到的角色信息对象
      editForm: {},
      // 添加表单的验证规则对象
      addFormRules: {},
      editDialogVisible: false
    }
  },
  created() {
    this.getCateList()
  },
  methods: {
    // 获取商品数据分类数据
    async getCateList() {
      const { data: res } = await this.$http.get('categories', { params: this.queryInfo })
      if (res.meta.status !== 200) {
        return this.$message.error('获取商品分类数据失败')
      }
      // 数据列表赋值
      this.catelist = res.data.result
      // 总数据条数赋值
      this.total = res.data.total
      console.log(res.data.result)
    },
    // 监听pagesize 改变的事件
    handleSizeChange(newSize) {
      this.queryInfo.pagesize = newSize
      this.getCateList()
    },
    // 监听pagenum的改变
    handleCurrentChange(newPage) {
      this.queryInfo.pagenum = newPage
      this.getCateList()
    },
    // 点击按钮,展示对话框
    showAddCateDialog() {
      // 先获取父级分类的列表
      this.getParentCateList()
      // 在展示出对话框
      this.addCateDialogVisible = true
    },
    // 获取父级分类的数据列表
    async getParentCateList() {
      const { data: res } = await this.$http.get('categories', { params: { type: 2 } })
      if (res.meta.status !== 200) {
        return this.$message.error('获取父级分类数据列表失败')
      }
      this.parentCateList = res.data
      console.log(res.data)
    },
    // 选择项发生这个变化触发这个函数
    parentCartChange() {
      console.log(this.selectedKeys)
      // 如果selectedKeys数组长度大于0，选中了数据
      // 反之，没有选中数据
      if (this.selectedKeys.length > 0) {
        // 父级分类的id
        this.addCateForm.cat_pid = this.selectedKeys[this.selectedKeys.length - 1]
        // 为当前分类的等级赋值
        this.addCateForm.cat_level = this.selectedKeys.length
        return false
      } else {
        // 父级分类的id
        this.addCateForm.cat_pid = 0
        // 为当前分类的等级赋值
        this.addCateForm.cat_level = 0
      }
    },
    // 点击按钮，添加分类
    addCate() {
      this.$refs.addCateFormRef.validate(async valid => {
        if (!valid) return
        const { data: res } = await this.$http.post('categories', this.addCateForm)
        if (res.meta.status !== 201) {
          return this.$message.error('添加分类失败')
        }
        this.$message.success('添加分类成功')
        this.addCateDialogVisible = false
        this.getCateList()
      })
    },
    // 监听添加分类对话框的关闭事件,重置表单数据
    addCateDialogClosed() {
      this.$refs.addCateFormRef.resetFields()
      this.selectedKeys = []
      this.addCateForm.cat_level = 0
      this.addCateForm.cat_pid = 0
    },
    async showEditDialog(id) {
      this.editDialogVisible = true
      const { data: res } = await this.$http.get('categories/' + id)
      if (res.meta.status !== 200) {
        return this.$message.error('查询角色信息失败')
      }
      this.editForm = res.data
      console.log(this.editForm)
      this.editDialogVisible = true
    },
    // 监听添加修改对话框的关闭事件
    editDialogClosed() {
      this.$refs.editFormRef.resetFields()
    },
    // 修改角色信息并提交
    editUserInfo() {
      this.$refs.editFormRef.validate(async valid => {
        if (!valid) return
        // 发起修改角色信息的请求
        const { data: res } = await this.$http.put('categories/' + this.editForm.cat_id, {
          cat_name: this.editForm.cat_name
        })
        if (res.meta.status !== 200) {
          return this.$message.error('修改分类信息失败')
        }
        // 关闭对话框
        this.editDialogVisible = false
        // 刷新数据列表
        this.getCateList()
        // 提示修改成功
        this.$message.success('修改分类信息成功')
      })
    },
    // 删除分类
    async removeUserById(id) {
      // 弹框询问角色是否删除
      const confirmResult = await this.$confirm('此操作将永久删除该分类, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).catch(err => err)
      // 如果角色确认了删除，返回confirm
      // 如果角色取消了删除，返回cancle
      console.log(confirmResult)
      if (confirmResult !== 'confirm') {
        return this.$message.info('已取消删除')
      }
      const { data: res } = await this.$http.delete('categories/' + id)
      if (res.meta.status !== 200) {
        return this.$message.error('删除分类失败')
      }
      this.getCateList()
      this.$message.success('删除分类成功')
    }
  }
}
</script>

<style lang="less" scoped>
.treeTable{
  margin-top: 15px;
}

.el-cascader {
  width: 100%;
}
</style>
