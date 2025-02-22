<template>
  <NodeContainer :nodeModel="nodeModel">
    <el-card shadow="never" class="card-never" style="--el-card-padding: 12px">
      <el-form
        @submit.prevent
        :model="form_data"
        label-position="top"
        require-asterisk-position="right"
        label-width="auto"
        ref="rerankerNodeFormRef"
        hide-required-asterisk
      >
        <el-form-item
          label="重排内容"
          prop="reranker_reference_list"
          :rules="{
            type: 'array',
            message: '请选择重排内容',
            trigger: 'change',
            required: true
          }"
        >
          <template #label>
            <div class="flex-between">
              <span>重排内容<span class="danger">*</span></span>
              <el-button @click="add_reranker_reference" link type="primary">
                <el-icon class="mr-4"><Plus /></el-icon>
              </el-button>
            </div>
          </template>
          <el-row
            :gutter="8"
            style="margin-bottom: 8px"
            v-for="(reranker_reference, index) in form_data.reranker_reference_list"
            :key="index"
          >
            <el-col :span="22">
              <el-form-item
                :prop="'reranker_reference_list.' + index"
                :rules="{
                  type: 'array',
                  required: true,
                  message: '请选择变量',
                  trigger: 'change'
                }"
              >
                <NodeCascader
                  :key="index"
                  :nodeModel="nodeModel"
                  class="w-full"
                  placeholder="请选择重排内容"
                  v-model="form_data.reranker_reference_list[index]"
                />
              </el-form-item>
            </el-col>
            <el-col :span="1">
              <el-button link type="info" class="mt-4" @click="deleteCondition(index)">
                <el-icon><Delete /></el-icon>
              </el-button>
            </el-col>
          </el-row>
        </el-form-item>
        <el-form-item label="检索参数">
          <template #label>
            <div class="flex-between">
              <span>检索参数</span>
              <el-button type="primary" link @click="openParamSettingDialog">
                <el-icon><Setting /></el-icon>
              </el-button>
            </div>
          </template>
          <div class="w-full">
            <el-row>
              <el-col :span="12" class="color-secondary lighter"> Score 高于</el-col>
              <el-col :span="12" class="lighter">
                {{ form_data.reranker_setting.similarity?.toFixed(3) }}</el-col
              >
              <el-col :span="12" class="color-secondary lighter"> 引用分段 Top</el-col>
              <el-col :span="12" class="lighter"> {{ form_data.reranker_setting.top_n }}</el-col>
              <el-col :span="12" class="color-secondary lighter"> 最大引用字符数</el-col>
              <el-col :span="12" class="lighter">
                {{ form_data.reranker_setting.max_paragraph_char_number }}</el-col
              >
            </el-row>
          </div>
        </el-form-item>
        <el-form-item
          label="检索问题"
          prop="question_reference_address"
          :rules="{
            message: '请选择检索问题',
            trigger: 'blur',
            required: true
          }"
        >
          <template #label>
            <div class="flex-between">
              <span>检索问题<span class="danger">*</span></span>
            </div>
          </template>
          <NodeCascader
            ref="nodeCascaderRef"
            :nodeModel="nodeModel"
            class="w-full"
            placeholder="检索问题"
            v-model="form_data.question_reference_address"
          />
        </el-form-item>
        <el-form-item
          label="重排模型"
          prop="reranker_model_id"
          :rules="{
            required: true,
            message: '请选择重排模型',
            trigger: 'change'
          }"
        >
          <template #label>
            <div class="flex-between">
              <span>重排模型<span class="danger">*</span></span>
            </div>
          </template>
          <ModelSelect
            @wheel="wheel"
            :teleported="false"
            v-model="form_data.reranker_model_id"
            placeholder="请选择重排模型"
            :options="modelOptions"
            @submitModel="getModel"
            showFooter
          ></ModelSelect>
        </el-form-item>
      </el-form>
    </el-card>
    <ParamSettingDialog ref="ParamSettingDialogRef" @refresh="refreshParam" />
  </NodeContainer>
</template>
<script setup lang="ts">
import { set, cloneDeep, groupBy } from 'lodash'
import NodeContainer from '@/workflow/common/NodeContainer.vue'
import NodeCascader from '@/workflow/common/NodeCascader.vue'
import ParamSettingDialog from './ParamSettingDialog.vue'
import { ref, computed, onMounted } from 'vue'

import applicationApi from '@/api/application'
import useStore from '@/stores'
import { app } from '@/main'

const { model } = useStore()
const props = defineProps<{ nodeModel: any }>()

const ParamSettingDialogRef = ref<InstanceType<typeof ParamSettingDialog>>()
const {
  params: { id }
} = app.config.globalProperties.$route as any
const form = {
  reranker_reference_list: [[]],
  reranker_model_id: '',
  question_reference_address: [],
  reranker_setting: {
    top_n: 3,
    similarity: 0,
    max_paragraph_char_number: 5000
  }
}

const modelOptions = ref<any>(null)
const openParamSettingDialog = () => {
  ParamSettingDialogRef.value?.open(form_data.value.reranker_setting)
}
const deleteCondition = (index: number) => {
  const list = cloneDeep(props.nodeModel.properties.node_data.reranker_reference_list)
  list.splice(index, 1)
  set(props.nodeModel.properties.node_data, 'reranker_reference_list', list)
}
const wheel = (e: any) => {
  if (e.ctrlKey === true) {
    e.preventDefault()
    return true
  } else {
    e.stopPropagation()
    return true
  }
}
const form_data = computed({
  get: () => {
    if (props.nodeModel.properties.node_data) {
      return props.nodeModel.properties.node_data
    } else {
      set(props.nodeModel.properties, 'node_data', form)
    }
    return props.nodeModel.properties.node_data
  },
  set: (value) => {
    set(props.nodeModel.properties, 'node_data', value)
  }
})
function refreshParam(data: any) {
  set(props.nodeModel.properties.node_data, 'reranker_setting', data)
}
function getModel() {
  if (id) {
    applicationApi.getApplicationRerankerModel(id).then((res: any) => {
      modelOptions.value = groupBy(res?.data, 'provider')
    })
  } else {
    model.asyncGetModel({ model_type: 'RERANKER' }).then((res: any) => {
      modelOptions.value = groupBy(res?.data, 'provider')
    })
  }
}

const add_reranker_reference = () => {
  const list = cloneDeep(props.nodeModel.properties.node_data.reranker_reference_list)
  list.push([])
  set(props.nodeModel.properties.node_data, 'reranker_reference_list', list)
}
const rerankerNodeFormRef = ref()
const nodeCascaderRef = ref()
const validate = () => {
  return Promise.all([
    nodeCascaderRef.value ? nodeCascaderRef.value.validate() : Promise.resolve(''),
    rerankerNodeFormRef.value?.validate()
  ]).catch((err: any) => {
    return Promise.reject({ node: props.nodeModel, errMessage: err })
  })
}

onMounted(() => {

  getModel()
  set(props.nodeModel, 'validate', validate)
})
</script>
<style lang="scss" scoped>
.reply-node-editor {
  :deep(.md-editor-footer) {
    border: none !important;
  }
}
</style>
