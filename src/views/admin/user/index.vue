<template>
  <d2-container>
    <!-- header 查询条件 -->
    <template slot="header">
      <el-form
        :inline="true"
        :model="listQuery"
        size="mini"
        style="margin-bottom: -18px;">
          <el-form-item label="用户名" prop="username">
            <el-input @keyup.enter.native="handleFilter" style="width: 200px;" placeholder="用户名" v-model="listQuery.username" clearable>
            </el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="default" @click="handleFilter" icon="el-icon-search">搜 索</el-button>
          </el-form-item>
          <el-form-item style="float: right">
            <el-button v-if="sys_user_add" @click="handleCreate" type="primary" icon="el-icon-plus">新 增</el-button>
          </el-form-item>
      </el-form>
    </template>
    <!-- table表格 -->
    <el-table :key='tableKey'
              :data="list"
              v-loading="listLoading"
              element-loading-text="拼命加载中..."
              highlight-current-row
              stripe
              style="width: 100%">

      <el-table-column align="center" label="序号" width="60">
        <template slot-scope="scope">
          <span>{{scope.row.userId}}</span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="用户名">
        <template slot-scope="scope">
          <span>
            <img v-if="scope.row.avatar" class="user-avatar" style="width: 20px; height: 20px; border-radius: 50%;" :src="scope.row.avatar+'?imageView2/1/w/20/h/20'"> {{scope.row.username}}
          </span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="手机号">
        <template slot-scope="scope">
          <span>{{scope.row.phone}}</span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="所属部门" show-overflow-tooltip>
        <template slot-scope="scope">
          <span>{{scope.row.deptName}} </span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="角色">
        <template slot-scope="scope">
          <span v-for="role in scope.row.roleList" :key="role.id">{{role.roleDesc}} </span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="创建时间">
        <template slot-scope="scope">
          <span>{{scope.row.createTime | parseTime('{y}-{m}-{d} {h}:{i}')}}</span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="标签">
        <template slot-scope="scope">
          <span>{{scope.row.label}}</span>
        </template>
      </el-table-column>

      <el-table-column align="center" label="状态">
        <template slot-scope="scope">
          <el-tag>{{scope.row.delFlag | statusFilter}}</el-tag>
        </template>
      </el-table-column>

      <el-table-column align="center" label="操作" width="200">
        <template slot-scope="scope">
          <el-button v-if="sys_user_upd" size="mini" type="primary" @click="handleUpdate(scope.row)" icon="el-icon-edit"></el-button>
          <el-button v-if="sys_user_del" size="mini" type="danger" @click="deletes(scope.row)" icon="el-icon-delete"></el-button>
        </template>
      </el-table-column>

    </el-table>
    <!-- footer 分页条 -->
    <template slot="footer">
        <el-pagination
          background
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page.sync="listQuery.page"
          :page-sizes="[10,20,30,50]"
          :page-size="listQuery.limit"
          layout="total, sizes, prev, pager, next, jumper"
          :total="total"
          style="margin: -10px;">
        </el-pagination>
    </template>
    <!-- 部门筛选 -->
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogDeptVisible" width="600px">
      <el-tree class="filter-tree" :data="treeDeptData" :default-checked-keys="checkedKeys" check-strictly node-key="id" highlight-current ref="deptTree" :props="defaultProps" @node-click="getNodeData" default-expand-all>
      </el-tree>
    </el-dialog>
    <!-- 新增用户弹框 -->
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible" width="600px">
      <el-form :model="form" :rules="rules" ref="form" label-width="80px" size="mini">
        <el-row>
          <el-col :span="12">
            <el-form-item label="用户名" prop="username">
              <el-input v-model="form.username" placeholder="请输用户名"></el-input>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="密码" placeholder="请输入密码" prop="newpassword1">
              <el-input :disabled="dialogStatus == 'update'" type="password" v-model="form.newpassword1"></el-input>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-col :span="12">
            <el-form-item label="所属部门" prop="deptName">
              <el-input v-model="form.deptName" placeholder="选择部门" @focus="handleDept()" readonly></el-input>
              <input type="hidden" v-model="form.deptId" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="角色" prop="roleList">
              <el-select v-model="form.roleList" value-key="roleId" placeholder="请选择" multiple collapse-tags style="width:100%;">
                <el-option v-for="item in rolesOptions" :key="item.roleId" :label="item.roleDesc" :value="item" :disabled="isDisabled[item.delFlag]">
                  <!-- <span style="float: left">{{ item.roleDesc }}</span> -->
                  <!-- <span style="float: right; color: #8492a6; font-size: 13px">{{ item.roleCode }}</span> -->
                </el-option>
              </el-select>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-col :span="12">
            <el-form-item label="手机号" prop="phone">
              <el-input v-model="form.phone" placeholder="验证码登录使用"></el-input>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="状态" prop="delFlag">
              <el-select v-model="form.delFlag" placeholder="请选择" :disabled="!sys_user_del" style="width:100%;">
                <el-option v-for="item in statusOptions" :key="item" :label="item | statusFilter" :value="item"> </el-option>
              </el-select>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-form-item label="标签" prop="label">
            <el-input v-model="form.label" placeholder="多个标签 ',' 隔开"></el-input>
          </el-form-item>
        </el-row>
        <el-row>
          <Treeselect v-model="value" :multiple="true" :options="options" />
        </el-row>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="cancel('form')" icon="el-icon-close" size="mini">取 消</el-button>
        <el-button v-if="dialogStatus=='create'" type="primary" @click="create('form')" icon="el-icon-check" size="mini">确 定</el-button>
        <el-button v-else type="primary" @click="update('form')" icon="el-icon-check" size="mini">修 改</el-button>
      </div>
    </el-dialog>
  </d2-container>
</template>

<script>
import { fetchList, getObj, addObj, putObj, delObj } from '@/api/user'
import { deptRoleList, fetchDeptTree } from '@/api/role'
import { mapGetters, mapState } from 'vuex'
import ElRadioGroup from 'element-ui/packages/radio/src/radio-group'
import ElOption from 'element-ui/packages/select/src/option'
import Treeselect from '@riophae/vue-treeselect'
export default {
  components: {
    ElOption,
    ElRadioGroup,
    Treeselect
  },
  name: 'table_user',
  data () {
    return {
      value: null,
      // define options
      options: [ {
        id: 'a',
        label: 'a',
        children: [ {
          id: 'aa',
          label: 'aa'
        }, {
          id: 'ab',
          label: 'ab'
        } ]
      }, {
        id: 'b',
        label: 'b'
      }, {
        id: 'c',
        label: 'c'
      } ],
      roletest: [],
      treeDeptData: [],
      checkedKeys: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
      list: null,
      total: null,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 10
      },
      form: {
        username: undefined,
        newpassword1: undefined,
        delFlag: undefined,
        deptId: undefined,
        phone: undefined,
        label: undefined,
        roleList: []
      },
      rules: {
        username: [
          {
            required: true,
            message: '请输入账户',
            trigger: 'blur'
          },
          {
            min: 3,
            max: 20,
            message: '长度在 3 到 20 个字符',
            trigger: 'blur'
          }
        ],
        newpassword1: [
          {
            min: 6,
            max: 20,
            message: '长度在 6 到 20 个字符',
            trigger: 'blur'
          }
        ],
        deptId: [
          {
            required: true,
            message: '请选择部门',
            trigger: 'blur'
          }
        ],
        roleList: [
          {
            type: 'array',
            required: true,
            message: '请选择角色',
            trigger: 'change'
          }
        ],
        phone: [
          {
            required: true,
            message: '手机号',
            trigger: 'blur'
          },
          {
            min: 11,
            max: 11,
            message: '长度在11 个字符',
            trigger: 'blur'
          }
        ]
      },
      statusOptions: ['0', '1'],
      rolesOptions: [],
      dialogFormVisible: false,
      dialogDeptVisible: false,
      userAdd: false,
      userUpd: false,
      userDel: false,
      dialogStatus: '',
      textMap: {
        update: '编辑用户',
        create: '新增用户'
      },
      isDisabled: {
        0: false,
        1: true
      },
      tableKey: 0
    }
  },
  computed: {
    ...mapGetters(['permissions']),
    ...mapState('d2admin/user', ['info'])
  },
  filters: {
    statusFilter (status) {
      const statusMap = {
        0: '有效',
        1: '无效',
        9: '锁定'
      }
      return statusMap[status]
    }
  },
  created () {
    this.getList()
    this.sys_user_add = this.permissions['sys_user_add']
    this.sys_user_upd = this.permissions['sys_user_upd']
    this.sys_user_del = this.permissions['sys_user_del']
  },
  methods: {
    getList () {
      this.listLoading = true
      this.listQuery.isAsc = false
      fetchList(this.listQuery).then(response => {
        this.list = response.data.records
        this.total = response.data.total
        this.listLoading = false
      })
    },
    getNodeData (data) {
      this.dialogDeptVisible = false
      this.form.deptId = data.id
      this.form.deptName = data.name
      deptRoleList(data.id).then(response => {
        this.rolesOptions = response.data
      })
    },
    handleDept () {
      fetchDeptTree().then(response => {
        this.treeDeptData = response.data
        this.dialogDeptVisible = true
      })
    },
    handleFilter () {
      this.listQuery.page = 1
      this.getList()
    },
    handleSizeChange (val) {
      this.listQuery.limit = val
      this.getList()
    },
    handleCurrentChange (val) {
      this.listQuery.page = val
      this.getList()
    },
    handleCreate () {
      this.resetTemp()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      // deptRoleList(this.info.deptId).then(response => {
      //   this.rolesOptions = response.data
      // })
    },
    handleUpdate (row) {
      getObj(row.userId).then(response => {
        this.form = response.data
        this.dialogFormVisible = true
        this.dialogStatus = 'update'
        deptRoleList(response.data.deptId).then(response => {
          this.rolesOptions = response.data
        })
      })
    },
    create (formName) {
      const set = this.$refs
      this.form.role = this.role
      set[formName].validate(valid => {
        if (valid) {
          addObj(this.form).then(() => {
            this.dialogFormVisible = false
            this.getList()
            this.$notify({
              title: '成功',
              message: '创建成功',
              type: 'success',
              duration: 2000
            })
          })
        } else {
          return false
        }
      })
    },
    cancel (formName) {
      this.dialogFormVisible = false
      this.$refs[formName].resetFields()
    },
    update (formName) {
      const set = this.$refs
      // this.form.role = this.role
      console.log(this.form)
      set[formName].validate(valid => {
        if (valid) {
          this.dialogFormVisible = false
          this.form.password = undefined
          putObj(this.form).then(() => {
            this.dialogFormVisible = false
            this.getList()
            this.$notify({
              title: '成功',
              message: '修改成功',
              type: 'success',
              duration: 2000
            })
          })
        } else {
          return false
        }
      })
    },
    deletes (row) {
      this.$confirm(
        '此操作将永久删除该用户(用户名:' + row.username + '), 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).then(() => {
        delObj(row.userId)
          .then(() => {
            this.getList()
            this.$notify({
              title: '成功',
              message: '删除成功',
              type: 'success',
              duration: 2000
            })
          })
          .cache(() => {
            this.$notify({
              title: '失败',
              message: '删除失败',
              type: 'error',
              duration: 2000
            })
          })
      })
    },
    resetTemp () {
      this.form = {
        id: undefined,
        username: '',
        password: '',
        roleList: [],
        delFlag: '',
        deptId: '',
        phone: ''
      }
    }
  }
}
</script>
