<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="名称" prop="planName">
        <el-input
          v-model="queryParams.planName"
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
        <el-button type="primary" @click="handleAdd" v-hasPermi="['nursing:plan:add']">新增护理计划</el-button>
      </el-col>
    </el-row>

    <el-table v-loading="loading" :data="planList">
      <el-table-column label="序号" type="index" width="80" align="center" :index="indexMethod" />
      <el-table-column label="名称" align="center" prop="planName" />
      <el-table-column label="创建时间" align="center" prop="createTime" width="200">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'" effect="light">
            {{ scope.row.status === 1 ? '启用' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="240" class-name="small-padding fixed-width">
        <template #default="scope">
          <span :title="scope.row.levelCount > 0 ? '该计划已绑定护理等级，无法操作' : ''" style="display: inline-block">
            <el-button
              link
              type="primary"
              :disabled="scope.row.levelCount > 0"
              @click="handleUpdate(scope.row)"
              v-hasPermi="['nursing:plan:edit']"
            >修改</el-button>
          </span>
          <span :title="scope.row.levelCount > 0 ? '该计划已绑定护理等级，无法操作' : ''" style="display: inline-block">
            <el-button
              link
              type="danger"
              :disabled="scope.row.levelCount > 0"
              @click="handleDelete(scope.row)"
              v-hasPermi="['nursing:plan:remove']"
            >删除</el-button>
          </span>
          <el-button link type="primary" @click="handleView(scope.row)">查看</el-button>
          <span :title="scope.row.levelCount > 0 ? '该计划已绑定护理等级，无法操作' : ''" style="display: inline-block">
            <el-button
              link
              :type="scope.row.status === 1 ? 'danger' : 'primary'"
              :disabled="scope.row.levelCount > 0"
              @click="handleStatusChange(scope.row)"
              v-hasPermi="['nursing:plan:edit']"
            >{{ scope.row.status === 1 ? '禁用' : '启用' }}</el-button>
          </span>
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

    <!-- 添加或修改护理计划对话框 -->
    <el-dialog :title="title" v-model="open" width="960px" append-to-body>
      <el-form ref="planRef" :model="form" :rules="rules" label-width="100px" :disabled="viewMode">
        <el-row>
          <el-col :span="12">
            <el-form-item label="排序号" prop="sortNo">
              <el-input-number v-model="form.sortNo" :min="0" controls-position="right" placeholder="请输入排序号" />
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
            <el-form-item label="名称" prop="planName">
              <el-input
                v-model="form.planName"
                placeholder="请输入名称"
                maxlength="10"
                show-word-limit
              />
            </el-form-item>
          </el-col>
        </el-row>
        <el-divider content-position="center">护理项目信息</el-divider>
        <el-row class="mb8">
          <el-col :span="1.5">
            <el-button
              v-if="!viewMode && canAddProject"
              type="primary"
              icon="Plus"
              @click="handleAddProjectPlan"
            >添加</el-button>
          </el-col>
        </el-row>
        <el-table :data="nursingProjectPlanList" border>
          <el-table-column label="序号" type="index" width="60" align="center" />
          <el-table-column label="护理项目" prop="projectId" min-width="180">
            <template #default="scope">
              <el-select v-model="scope.row.projectId" placeholder="请选择护理项目" style="width: 100%">
                <el-option
                  v-for="item in projectOptions(scope.row)"
                  :key="item.id"
                  :label="item.name"
                  :value="item.id"
                />
              </el-select>
            </template>
          </el-table-column>
          <el-table-column label="期望服务时间" prop="executeTime" width="170">
            <template #default="scope">
              <el-time-select
                v-model="scope.row.executeTime"
                start="00:00"
                end="23:30"
                step="00:30"
                placeholder="选择时间"
                style="width: 100%"
              />
            </template>
          </el-table-column>
          <el-table-column label="执行周期" prop="executeCycle" width="150">
            <template #default="scope">
              <el-select v-model="scope.row.executeCycle" placeholder="请选择执行周期" style="width: 100%">
                <el-option
                  v-for="dict in cycleOptions"
                  :key="dict.value"
                  :label="dict.label"
                  :value="dict.value"
                />
              </el-select>
            </template>
          </el-table-column>
          <el-table-column label="执行频次" prop="executeFrequency" width="160">
            <template #default="scope">
              <el-input-number
                v-model="scope.row.executeFrequency"
                :min="1"
                :max="7"
                :step="1"
                :precision="0"
                controls-position="right"
                style="width: 100%"
              />
            </template>
          </el-table-column>
          <el-table-column label="操作" width="80" align="center">
            <template #default="scope">
              <el-button
                v-if="!viewMode && nursingProjectPlanList.length > 1"
                link
                type="danger"
                icon="Delete"
                @click="handleDeleteProjectPlan(scope.$index)"
              >删除</el-button>
            </template>
          </el-table-column>
        </el-table>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button v-if="!viewMode" type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">{{ viewMode ? '关 闭' : '取 消' }}</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Plan">
import { listPlan, getPlan, delPlan, addPlan, updatePlan } from "@/api/nursing/plan"
import { listProject } from "@/api/nursing/project"

const { proxy } = getCurrentInstance()

// 状态：1 启用，0 禁用（字典 nursing_status 建好后可替换为 useDict）
const statusOptions = [
  { value: 1, label: "启用" },
  { value: 0, label: "禁用" }
]
// 执行周期：0 每日，1 每周，2 每月（字典建好后可替换为 useDict）
const cycleOptions = [
  { value: 0, label: "每日" },
  { value: 1, label: "每周" },
  { value: 2, label: "每月" }
]

const planList = ref([])
const projectList = ref([])
const nursingProjectPlanList = ref([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const viewMode = ref(false)
const total = ref(0)
const title = ref("")

// 是否还能继续添加项目行：未选项目的数量 > 0
const canAddProject = computed(() => {
  const used = nursingProjectPlanList.value
    .map(item => item.projectId)
    .filter(id => id !== undefined && id !== null)
  return used.length < projectList.value.length
})

// 当前行可选的项目：启用状态 + 未被其他行选择
function projectOptions(currentRow) {
  return projectList.value.filter(item => {
    const usedByOther = nursingProjectPlanList.value.some(
      row => row !== currentRow && row.projectId === item.id
    )
    return !usedByOther
  })
}

const data = reactive({
  form: {},
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    planName: undefined,
    status: undefined,
  },
  rules: {
    sortNo: [
      { required: true, message: "排序号不能为空", trigger: "blur" }
    ],
    planName: [
      { required: true, message: "名称不能为空", trigger: "blur" },
      { max: 10, message: "名称长度不能超过10个字符", trigger: "blur" },
      { validator: checkPlanNameUnique, trigger: "blur" }
    ],
    status: [
      { required: true, message: "请选择状态", trigger: "change" }
    ],
  }
})

const { queryParams, form, rules } = toRefs(data)

/** 分页序号连续 */
function indexMethod(index) {
  return (queryParams.value.pageNum - 1) * queryParams.value.pageSize + index + 1
}

/** 校验计划名称唯一（排除自身） */
function checkPlanNameUnique(rule, value, callback) {
  if (!value) {
    return callback()
  }
  listPlan({ planName: value, pageNum: 1, pageSize: 1000 }).then(response => {
    const duplicate = response.rows.some(item => item.planName === value && item.id !== form.value.id)
    if (duplicate) {
      callback(new Error("计划名称已存在"))
    } else {
      callback()
    }
  })
}

/** 查询护理计划列表 */
function getList() {
  loading.value = true
  listPlan(queryParams.value).then(response => {
    planList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

/** 查询启用的护理项目（下拉数据源） */
function getProjectList() {
  listProject({ pageNum: 1, pageSize: 1000 }).then(response => {
    projectList.value = response.rows.filter(item => item.status === 1)
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
    sortNo: 0,
    planName: null,
    status: 1,
    createTime: null,
    updateTime: null,
    createBy: null,
    updateBy: null,
    remark: null
  }
  nursingProjectPlanList.value = []
  proxy.resetForm("planRef")
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
  viewMode.value = false
  handleAddProjectPlan()
  open.value = true
  title.value = "新增护理计划"
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset()
  viewMode.value = false
  getPlan(row.id).then(response => {
    fillForm(response.data)
    open.value = true
    title.value = "修改护理计划"
  })
}

/** 查看按钮操作（只读） */
function handleView(row) {
  reset()
  viewMode.value = true
  getPlan(row.id).then(response => {
    fillForm(response.data)
    open.value = true
    title.value = "查看护理计划"
  })
}

/** 回填计划详情及子表数据 */
function fillForm(data) {
  form.value = data
  nursingProjectPlanList.value = data.nursingProjectPlanList || []
}

/** 启用/禁用操作：先查全量再提交，避免先删后建逻辑清空项目关联 */
function handleStatusChange(row) {
  const text = row.status === 1 ? "禁用" : "启用"
  const newStatus = row.status === 1 ? 0 : 1
  proxy.$modal.confirm('确认要' + text + '护理计划"' + row.planName + '"吗？').then(() => {
    return getPlan(row.id)
  }).then(response => {
    const data = response.data
    data.status = newStatus
    return updatePlan(data)
  }).then(() => {
    proxy.$modal.msgSuccess(text + "成功")
    getList()
  }).catch(() => {})
}

/** 新增一条项目关联行，按原型给默认值 */
function handleAddProjectPlan() {
  nursingProjectPlanList.value.push({
    projectId: undefined,
    executeTime: "08:00",
    executeCycle: 1,
    executeFrequency: 1
  })
}

/** 删除指定项目关联行 */
function handleDeleteProjectPlan(index) {
  nursingProjectPlanList.value.splice(index, 1)
}

/** 校验子表行是否填写完整 */
function validateProjectPlans() {
  const list = nursingProjectPlanList.value
  if (!list || list.length === 0) {
    return false
  }
  return list.every(item =>
    item.projectId !== undefined && item.projectId !== null &&
    item.executeTime &&
    item.executeCycle !== undefined && item.executeCycle !== null &&
    Number.isInteger(item.executeFrequency) &&
    item.executeFrequency >= 1 && item.executeFrequency <= 7
  )
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["planRef"].validate(valid => {
    if (!valid) {
      return
    }
    if (!validateProjectPlans()) {
      proxy.$modal.msgError("护理项目信息不完整，请重新设置")
      return
    }
    form.value.nursingProjectPlanList = nursingProjectPlanList.value
    if (form.value.id != null) {
      updatePlan(form.value).then(() => {
        proxy.$modal.msgSuccess("修改成功")
        open.value = false
        getList()
      })
    } else {
      addPlan(form.value).then(() => {
        proxy.$modal.msgSuccess("新增成功")
        open.value = false
        getList()
      })
    }
  })
}

/** 删除按钮操作 */
function handleDelete(row) {
  proxy.$modal.confirm('是否确认删除护理计划"' + row.planName + '"？').then(function() {
    return delPlan(row.id)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {})
}

getList()
getProjectList()
</script>
