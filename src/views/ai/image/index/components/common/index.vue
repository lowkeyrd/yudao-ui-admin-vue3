<!-- dall3 -->
<template>
  <div class="prompt">
    <el-text tag="b">画面描述</el-text>
    <el-text tag="p">建议使用“形容词 + 动词 + 风格”的格式，使用“，”隔开</el-text>
    <el-input
      v-model="prompt"
      maxlength="1024"
      :rows="5"
      class="w-100% mt-15px"
      input-style="border-radius: 7px;"
      placeholder="例如：童话里的小屋应该是什么样子？"
      show-word-limit
      type="textarea"
    />
  </div>
  <div class="flex flex-col mt-30px">
    <div>
      <el-text tag="b">随机热词</el-text>
    </div>
    <el-space wrap class="flex flex-row flex-wrap justify-start mt-15px">
      <el-button
        round
        class="m-0"
        :type="selectHotWord === hotWord ? 'primary' : 'default'"
        v-for="hotWord in ImageHotWords"
        :key="hotWord"
        @click="handleHotWordClick(hotWord)"
      >
        {{ hotWord }}
      </el-button>
    </el-space>
  </div>
  <div class="mt-30px">
    <div>
      <el-text tag="b">平台</el-text>
    </div>
    <el-space wrap class="mt-15px w-full">
      <el-select
        v-model="otherPlatform"
        placeholder="Select"
        size="large"
        class="!w-350px"
        @change="handlerPlatformChange"
      >
        <el-option
          v-for="item in OtherPlatformEnum"
          :key="item.key"
          :label="item.name"
          :value="item.key"
        />
      </el-select>
    </el-space>
  </div>
  <div class="mt-30px">
    <div>
      <el-text tag="b">模型</el-text>
    </div>
    <el-space wrap class="mt-15px w-full">
      <el-select v-model="modelId" placeholder="Select" size="large" class="!w-350px">
        <el-option
          v-for="item in platformModels"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </el-space>
  </div>
  <div class="mt-30px">
    <div>
      <el-text tag="b">图片尺寸</el-text>
    </div>
    <el-space wrap class="mt-15px w-full">
      <el-input v-model="width" type="number" class="w-170px" placeholder="图片宽度" />
      <el-input v-model="height" type="number" class="w-170px" placeholder="图片高度" />
    </el-space>
  </div>
  <div class="flex justify-center mt-50px">
    <el-button
      type="primary"
      size="large"
      round
      :loading="drawIn"
      :disabled="prompt.length === 0"
      @click="handleGenerateImage"
    >
      {{ drawIn ? '生成中' : '生成内容' }}
    </el-button>
  </div>
</template>
<script setup lang="ts">
import { ImageApi, ImageDrawReqVO, ImageVO } from '@/api/ai/image'
import { AiPlatformEnum, ImageHotWords, OtherPlatformEnum } from '@/views/ai/utils/constants'
import { ModelVO } from '@/api/ai/model/model'

const message = useMessage() // 消息弹窗

// 接收父组件传入的模型列表
const props = defineProps({
  models: {
    type: Array<ModelVO>,
    default: () => [] as ModelVO[]
  }
})
const emits = defineEmits(['onDrawStart', 'onDrawComplete']) // 定义 emits

// 定义属性
const drawIn = ref<boolean>(false) // 生成中
const selectHotWord = ref<string>('') // 选中的热词
// 表单
const prompt = ref<string>('') // 提示词
const width = ref<number>(512) // 图片宽度
const height = ref<number>(512) // 图片高度
const otherPlatform = ref<string>(AiPlatformEnum.TONG_YI) // 平台
const platformModels = ref<ModelVO[]>([]) // 模型列表
const modelId = ref<number>() // 选中的模型

/** 选择热词 */
const handleHotWordClick = async (hotWord: string) => {
  // 情况一：取消选中
  if (selectHotWord.value == hotWord) {
    selectHotWord.value = ''
    return
  }

  // 情况二：选中
  selectHotWord.value = hotWord // 选中
  prompt.value = hotWord // 替换提示词
}

/** 图片生成 */
const handleGenerateImage = async () => {
  // 二次确认
  await message.confirm(`确认生成内容?`)
  try {
    // 加载中
    drawIn.value = true
    // 回调
    emits('onDrawStart', otherPlatform.value)
    // 发送请求
    const form = {
      platform: otherPlatform.value,
      modelId: modelId.value, // 模型
      prompt: prompt.value, // 提示词
      width: width.value, // 图片宽度
      height: height.value, // 图片高度
      options: {}
    } as unknown as ImageDrawReqVO
    await ImageApi.drawImage(form)
  } finally {
    // 回调
    emits('onDrawComplete', otherPlatform.value)
    // 加载结束
    drawIn.value = false
  }
}

/** 填充值 */
const settingValues = async (detail: ImageVO) => {
  prompt.value = detail.prompt
  width.value = detail.width
  height.value = detail.height
}

/** 平台切换 */
const handlerPlatformChange = async (platform: string) => {
  // 根据选择的平台筛选模型
  platformModels.value = props.models.filter((item: ModelVO) => item.platform === platform)

  // 切换平台，默认选择一个模型
  if (platformModels.value.length > 0) {
    modelId.value = platformModels.value[0].id // 使用 model 属性作为值
  } else {
    modelId.value = undefined
  }
}

/** 监听 models 变化 */
watch(
  () => props.models,
  () => {
    handlerPlatformChange(otherPlatform.value)
  },
  { immediate: true, deep: true }
)
/** 暴露组件方法 */
defineExpose({ settingValues })
</script>
