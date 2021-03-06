<template>
  <div>
    <bread-crumb>
      <span slot="left">首页</span>
      <span slot="center">用户管理</span>
      <span slot="right">用户列表</span>
    </bread-crumb>

    <card>
      <!-- 搜索与添加 -->
      <el-row :gutter="20" slot="add">
        <el-col :span="8">
          <el-input
            placeholder="请输入内容"
            clearable
            @clear="this.getUserList"
            class="input-with-select"
            v-model="queryInfo.query"
          >
            <el-button slot="append" @click="this.getUserList" icon="el-icon-search"></el-button>
          </el-input>
        </el-col>
        <el-col :span="4">
          <div class="grid-content bg-purple">
            <el-button type="primary" @click="addDialogVisible=true">添加用户</el-button>
          </div>
        </el-col>
      </el-row>

      <!-- 用户列表 -->
      <el-table :data="userlist" border stripe slot="content">
        <el-table-column type="index"></el-table-column>
        <el-table-column label="姓名" prop="username"></el-table-column>
        <el-table-column label="邮箱" prop="email"></el-table-column>
        <el-table-column label="电话" prop="mobile"></el-table-column>
        <el-table-column label="角色" prop="role_name"></el-table-column>
        <el-table-column label="状态">
          <template slot-scope="scope">
            <el-switch v-model="scope.row.mg_state" @change="userStateChanged(scope.row)"></el-switch>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="180px">
          <template slot-scope="scope">
            <!-- 编辑 -->
            <el-button
              type="primary"
              @click="showEditDialog(scope.row.id)"
              icon="el-icon-edit"
              size="mini"
            ></el-button>
            <!-- 删除 -->
            <el-button
              type="danger"
              @click="removeUserById(scope.row.id)"
              icon="el-icon-delete"
              size="mini"
            ></el-button>
            <!-- 分配角色 -->
            <el-tooltip effect="dark" content="分配角色" :enterable="false" placement="top">
              <el-button
                type="warning"
                icon="el-icon-setting"
                size="mini"
                @click="setRole(scope.row)"
              ></el-button>
            </el-tooltip>
          </template>
        </el-table-column>
      </el-table>

      <!-- 分页 -->
      <el-pagination
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        :current-page="queryInfo.pagenum"
        :page-sizes="[1, 2, 5, 10]"
        :page-size="queryInfo.pagesize"
        layout="total, sizes, prev, pager, next, jumper"
        :total="total"
        slot="pagination"
      ></el-pagination>
    </card>

    <!-- 添加用户的对话框 -->
    <el-dialog title="添加用户" :visible.sync="addDialogVisible" width="50%" @close="addDialogClosed">
      <el-form :model="addForm" :rules="addFormRules" ref="addFormRef" label-width="70px">
        <el-form-item label="用户名" prop="username">
          <el-input v-model="addForm.username"></el-input>
        </el-form-item>
        <el-form-item label="密码" prop="password">
          <el-input v-model="addForm.password"></el-input>
        </el-form-item>
        <el-form-item label="邮箱" prop="email">
          <el-input v-model="addForm.email"></el-input>
        </el-form-item>
        <el-form-item label="手机" prop="mobile">
          <el-input v-model="addForm.mobile"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="addDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addUser">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 修改用户的对话框 -->
    <el-dialog title="修改用户" :visible.sync="editDialogVisible" width="50%" @close="editDialogClosed">
      <el-form :model="editForm" :rules="editFormRules" ref="editFormRef" label-width="70px">
        <el-form-item label="用户名" prop="username">
          <el-input v-model="editForm.username" disabled></el-input>
        </el-form-item>
        <el-form-item label="邮箱" prop="email">
          <el-input v-model="editForm.email"></el-input>
        </el-form-item>
        <el-form-item label="手机" prop="mobile">
          <el-input v-model="editForm.mobile"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="editDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editUserInfo">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 分配角色的对话框 -->
    <el-dialog
      title="修改用户"
      :visible.sync="setRoleDialogVisible"
      width="50%"
      @close="setRoleDialogClosed"
    >
      <div>
        <p>当前的用户：{{userInfo.username}}</p>
        <p>当前的角色：{{userInfo.role_name}}</p>
        <p>
          分配的新角色：
          <el-select v-model="selectdRoleId" placeholder="请选择">
            <el-option
              v-for="item in rolesList"
              :key="item.id"
              :label="item.roleName"
              :value="item.id"
            ></el-option>
          </el-select>
        </p>
      </div>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setRoleDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="saveRoleInfo">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import { showDataMixin } from "common/mixin";
import Card from "components/Card";
import BreadCrumb from "components/BreadCrumb";

export default {
  name: "Users",
  mixins: [showDataMixin],
  components: {
    Card,
    BreadCrumb,
  },
  data() {
    let checkEmail = (rule, value, cb) => {
      const regEmail = /^([a-zA-Z0-9_-])+@([a-zA-Z0-9_-])+(\.[a-zA-Z0-9_-])+/;
      if (regEmail.test(value)) {
        return cb();
      }
      cb(new Error("请输入合法的邮箱"));
    };
    let checkMobile = (rule, value, cb) => {
      const regMobile = /^(0|86|17951)?(13[0-9]|15[0123456789]|17[678]|18[0-9]|14[57])[0-9]{8}$/;
      if (regMobile.test(value)) return cb();
      cb(new Error("请输入合法的手机号"));
    };
    return {
      queryInfo: {
        query: "",
        pagenum: 1,
        pagesize: 2,
      },
      userlist: [],
      userInfo: {},
      rolesList: [],
      total: 0,
      selectdRoleId:'',
      addDialogVisible: false,
      editDialogVisible: false,
      setRoleDialogVisible: false,
      editForm: {},
      addForm: {
        username: "",
        password: "",
      },
      // 规则
      editFormRules: {
        email: [
          { required: true, message: "请输入邮箱", trigger: "blur" },
          { validator: checkEmail, trigger: "blur" },
        ],
        mobile: [
          { required: true, message: "请输入手机号", trigger: "blur" },
          { validator: checkMobile, trigger: "blur" },
        ],
      },
      addFormRules: {
        username: [
          { required: true, message: "请输入用户名", trigger: "blur" },
          {
            min: 3,
            max: 10,
            message: "用户名长度在3~10个字符之间",
            trigger: "blur",
          },
        ],
        password: [
          { required: true, message: "请输入密码", trigger: "blur" },
          {
            min: 6,
            max: 15,
            message: "密码长度在6~15个字符之间",
            trigger: "blur",
          },
        ],
        email: [
          { required: true, message: "请输入邮箱", trigger: "blur" },
          { validator: checkEmail, trigger: "blur" },
        ],
        mobile: [
          { required: true, message: "请输入手机号", trigger: "blur" },
          { validator: checkMobile, trigger: "blur" },
        ],
      },
    };
  },
  created() {
    this.getUserList();
  },
  methods: {
    // ---分页---
    // 监听pagesize改变
    handleSizeChange(newSize) {
      console.log(newSize);
      this.queryInfo.pagesize = newSize;
      this.getUserList();
    },
    // 监听页码改变
    handleCurrentChange(newPage) {
      console.log(newPage);
      this.queryInfo.pagenum = newPage;
      this.getUserList();
    },

    // 监听用户状态改变
    async userStateChanged(userinfo) {
      console.log(userinfo);
      const { data: res } = await this.$http.put(
        `users/${userinfo.id}/state/${userinfo.mg_state}`
      );
      if (res.meta.status !== 200) {
        userinfo.mg_state = !userinfo.mg_state;
        return this.$message.error("更新用户状态失败");
      }
      this.$message.success("更新用户状态成功");
    },

    // -----添加-----

    // 监听添加对话框关闭
    addDialogClosed() {
      this.$refs.addFormRef.resetFields();
    },

    // 点击添加新用户
    addUser() {
      this.$refs.addFormRef.validate(async (valid) => {
        if (!valid) return;
        console.log(valid);
        const { data: res } = await this.$http.post("users", this.addForm);
        if (res.meta.status !== 201) {
          this.$message.error("添加用户失败！");
        }
        this.$message.success("添加用户成功");
        this.addDialogVisible = false;
        this.getUserList();
      });
    },

    // -----编辑-----

    // 监听修改对话框关闭
    editDialogClosed() {
      this.$refs.editFormRef.resetFields();
    },

    // 编辑用户
    async showEditDialog(id) {
      this.editDialogVisible = true;
      const { data: res } = await this.$http.get("users/" + id);
      const message = "查询用户信息失败";
      this.showError(res, message);
      this.editForm = res.data;
      console.log(this.editForm);
    },

    // 修改用户信息并提交
    editUserInfo() {
      this.$refs.editFormRef.validate(async (valid) => {
        console.log(valid);
        if (!valid) return;
        const { data: res } = await this.$http.put(
          "users/" + this.editForm.id,
          {
            email: this.editForm.email,
            mobile: this.editForm.mobile,
          }
        );
        const message = "更新用户信息失败！";
        this.showError(res, message);
        this.editDialogVisible = false;
        this.getUserList();
        this.$message.success("更新用户信息成功");
      });
    },

    // 展示分配角色对话框
    async setRole(userInfo) {
      this.userInfo = userInfo;
      const { data: res } = await this.$http.get("roles");
      const message = "获取角色列表失败";
      this.showError(res, message);
      this.rolesList = res.data;
      console.log(this.rolesList);
      this.setRoleDialogVisible = true;
    },

    // 分配角色
    async saveRoleInfo(){
      if(!this.selectdRoleId){
        return this.$message.error('请选择要分配的角色！')
      }

      const {data:res} = await this.$http.put(`users/${this.userInfo.id}/role`,{
        rid:this.selectdRoleId
      })

      this.showError(res,'更新角色失败！')
      this.$message.success('更新角色成功！')
      this.getUserList()
      this.setRoleDialogVisible=false
    },

    // 监听分配框关闭
    setRoleDialogClosed(){
      this.selectdRoleId=''
      this.userInfo={}
    },

    // -----删除-----

    // 根据id删除信息
    async removeUserById(id) {
      const confirmResult = await this.$confirm(
        "此操作将永久删除该用户, 是否继续?",
        "提示",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning",
        }
      ).catch((err) => err);
      if (confirmResult == "confirm") {
        const { data: res } = await this.$http.delete("users/" + id);
        if (res.meta.status !== 200) {
          return this.$message.error("删除用户失败！");
        }
        this.$message.success("删除用户成功！");
        this.getUserList();
        return this.$message.success("已删除");
      }
    },
  },
};
</script>

<style scoped>
</style>