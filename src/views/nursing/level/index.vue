<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="80px">
      <el-form-item label="等级名称" prop="name">
        <el-input
          v-model="queryParams.name"
          placeholder="请输入"
          clearable
          style="width: 240px"
          @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="queryParams.status" placeholder="请选择" clearable style="width: 280px">
          <el-option
            v-for="dict in statusOptions"
            :key="dict.value"
            :label="dict.label"
            :value="dict.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button @click="resetQuery">重 置</el-button>
        <el-button type="primary" @click="handleQuery">搜 索</el-button>
      </el-form-item>
    </el-form>

    <el-row class="mb8">
      <el-col :span="24" style="text-align: right">
        <el-button type="primary" @click="handleAdd" v-hasPermi="['nursing:level:add']">新增护理等级</el-button>
      </el-col>
    </el-row>

    <el-table v-loading="loading" :data="levelList">
      <el-table-column label="序号" type="index" width="80" align="center" :index="indexMethod" />
      <el-table-column label="护理等级名称" align="center" prop="name" />
      <el-table-column label="执行护理计划" align="center" prop="planName" />
      <el-table-column label="护理费用（元/月）" align="center" prop="fee">
        <template #default="scope">
          {{ Number(scope.row.fee || 0).toFixed(2) }}
        </template>
      </el-table-column>
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'" effect="light">
            {{ scope.row.status === 1 ? '启用' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="等级说明" align="center" prop="description" show-overflow-tooltip />
      <el-table-column label="创建时间" align="center" prop="createTime" width="170">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="200" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" @click="handleUpdate(scope.row)" v-hasPermi="['nursing:level:edit']">修改</el-button>
          <el-button link type="danger" @click="handleDelete(scope.row)" v-hasPermi="['nursing:level:remove']">删除</el-button>
          <el-button
            link
            :type="scope.row.status === 1 ? 'danger' : 'primary'"
            @click="handleStatusChange(scope.row)"
            v-hasPermi="['nursing:level:edit']"
          >{{ scope.row.status === 1 ? '禁用' : '启用' }}</el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
      v-show="total>0"
      :total="total"
      v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改护理等级对话框 -->
    <el-dialog :title="title" v-model="open" width="600px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="110px">
        <el-row>
          <el-col :span="24">
            <el-form-item label="等级名称" prop="name">
              <el-input v-model="form.name" placeholder="请输入等级名称" maxlength="20" />
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="执行护理计划" prop="planId">
              <el-select v-model="form.planId" placeholder="请选择护理计划" style="width: 100%">
                <el-option
                  v-for="item in planList"
                  :key="item.id"
                  :label="item.planName"
                  :value="item.id"
                />
              </el-select>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="护理费用" prop="fee">
              <el-input-number v-model="form.fee" :min="0" :precision="2" controls-position="right" placeholder="请输入护理费用" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="状态" prop="status">
              <el-radio-group v-model="form.status">
                <el-radio
                  v-for="dict in statusOptions"
                  :key="dict.value"
                  :value="dict.value"
                >{{ dict.label }}</el-radio>
              </el-radio-group>
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="等级说明" prop="description">
              <el-input v-model="form.description" type="textarea" :rows="3" placeholder="请输入等级说明" />
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Level">
import { listLevel, getLevel, delLevel, addLevel, updateLevel } from "@/api/nursing/level"
import { listPlan } from "@/api/nursing/plan"

const { proxy } = getCurrentInstance()

// 状态：1 启用，0 禁用（字典 nursing_status 建好后可替换为 useDict）
const statusOptions = [
  { value: 1, label: "启用" },
  { value: 0, label: "禁用" }
]

const levelList = ref([])
const planList = ref([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const total = ref(0)
const title = ref("")

const data = reactive({
  form: {},
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: undefined,
    status: undefined,
  },
  rules: {
    name: [
      { required: true, message: "等级名称不能为空", trigger: "blur" }
    ],
    planId: [
      { required: true, message: "请选择执行护理计划", trigger: "change" }
    ],
    fee: [
      { required: true, message: "护理费用不能为空", trigger: "blur" }
    ],
    status: [
      { required: true, message: "请选择状态", trigger: "change" }
    ],
    description: [
      { required: true, message: "等级说明不能为空", trigger: "blur" }
    ],
  }
})

const { queryParams, form, rules } = toRefs(data)

/** 分页序号连续 */
function indexMethod(index) {
  return (queryParams.value.pageNum - 1) * queryParams.value.pageSize + index + 1
}

/** 查询护理等级列表 */
function getList() {
  loading.value = true
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

/** 查询全部护理计划（下拉数据源，含禁用计划以保证回显） */
function getPlanList() {
  listPlan({ pageNum: 1, pageSize: 1000 }).then(response => {
    planList.value = response.rows
  })
}

/** 取消按钮 */
function cancel() {
  open.value = false
  reset()
}

/** 表单重置 */
function reset() {
  form.value = {
    id: null,
    name: null,
    planId: null,
    fee: 0,
    status: 1,
    description: null,
    remark: null
  }
  proxy.resetForm("levelRef")
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery()
}

/** 新增按钮操作 */
function handleAdd() {
  reset()
  open.value = true
  title.value = "新增护理等级"
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset()
  getLevel(row.id).then(response => {
    form.value = response.data
    open.value = true
    title.value = "修改护理等级"
  })
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["levelRef"].validate(valid => {
    if (!valid) {
      return
    }
    if (form.value.id != null) {
      updateLevel(form.value).then(() => {
        proxy.$modal.msgSuccess("修改成功")
        open.value = false
        getList()
      })
    } else {
      addLevel(form.value).then(() => {
        proxy.$modal.msgSuccess("新增成功")
        open.value = false
        getList()
      })
    }
  })
}

/** 删除按钮操作 */
function handleDelete(row) {
  proxy.$modal.confirm('是否确认删除护理等级"' + row.name + '"？').then(function() {
    return delLevel(row.id)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {})
}

/** 启用/禁用操作 */
function handleStatusChange(row) {
  const text = row.status === 1 ? "禁用" : "启用"
  const newStatus = row.status === 1 ? 0 : 1
  proxy.$modal.confirm('确认要' + text + '护理等级"' + row.name + '"吗？').then(function() {
    return updateLevel({ ...row, status: newStatus })
  }).then(() => {
    proxy.$modal.msgSuccess(text + "成功")
    getList()
  }).catch(() => {})
}

getList()
getPlanList()
</script>
