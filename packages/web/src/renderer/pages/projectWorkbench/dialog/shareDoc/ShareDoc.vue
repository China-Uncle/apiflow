<template>
  <el-dialog :model-value="modelValue" top="10vh" width="45%" :title="t('分享接口')" :before-close="handleClose" destroy-on-close>
    <div class="share-wrap">
      <SConfig :label="t('密码设置')" :has-check="false" :description="t('密码可不填写')">
        <el-input v-model="password" :placeholder="t('请输入密码')" class="w-100" type="password" show-password clearable />
      </SConfig>
      <SConfig :label="`${t('过期时间')}(${formatExpiry(expiry)})`" :has-check="false" :description="t('不填默认一个月后过期')">
        <el-radio-group v-model="expiry">
          <el-radio :value="86400000">{{ t('1天后') }}</el-radio>
          <el-radio :value="86400000 * 7">{{ t('1周后') }}</el-radio>
          <el-radio :value="86400000 * 30">{{ t('1个月后') }}</el-radio>
          <el-radio :value="86400000 * 365 * 5">{{ t('不过期') }}</el-radio>
        </el-radio-group>
      </SConfig>
      <div v-if="shareUrl" class="share-url-wrap">
        <SConfig :label="t('分享链接')" :has-check="false">
          <div class="d-flex a-center gap-2">
            <el-input v-model="shareUrl" readonly class="share-url-input" @click="copyShareUrl" />
            <el-button type="primary" @click="copyShareUrl">{{ t('复制') }}</el-button>
          </div>
        </SConfig>
      </div>
    </div>
    <template #footer>
      <el-button @click="handleClose">{{ t('取消') }}</el-button>
      <el-button :loading="loading" type="primary" @click="handleGenerate">{{ shareUrl ? t('关闭') : t('生成链接') }}</el-button>
    </template>
  </el-dialog>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import { request } from '@/api/api'
import { message } from '@/helper'
import { router } from '@/router'
import SConfig from '@/components/common/config/ClConfig.vue'

const { t } = useI18n()

const props = defineProps<{
  modelValue: boolean
  docId: string
}>()
const emit = defineEmits<{
  (e: 'update:modelValue', v: boolean): void
}>()

const projectId = router.currentRoute.value.query.id as string
const loading = ref(false)
const password = ref('')
const expiry = ref(86400000 * 30)
const shareUrl = ref('')

const handleClose = () => {
  emit('update:modelValue', false)
}

const formatExpiry = (val: number) => {
  const days = Math.max(0, Math.floor(val / 86400000))
  const hours = Math.max(0, Math.floor((val % 86400000) / 3600000))
  return `${days}${t('天')}${hours}${t('小时')}`
}

const copyShareUrl = async () => {
  try {
    await navigator.clipboard.writeText(shareUrl.value)
    message.success(t('链接已复制'))
  } catch {
    message.warning(t('复制失败，请手动复制'))
  }
}

const handleGenerate = async () => {
  if (shareUrl.value) {
    handleClose()
    return
  }
  loading.value = true
  try {
    const res = await request.post('/api/project/export/online', {
      shareName: `单接口分享-${props.docId.slice(-8)}`,
      projectId,
      maxAge: expiry.value,
      password: password.value,
      selectedDocs: [props.docId],
    })
    const { origin, hash } = location
    const base = origin + hash.replace(/\/$/, '')
    const query = `?share_id=${res.data}&id=${projectId}&docId=${props.docId}`
    shareUrl.value = `${base}/share${query}`
    message.success(t('分享链接生成成功'))
  } catch (err) {
    console.error(err)
  } finally {
    loading.value = false
  }
}
</script>

<style lang="scss" scoped>
.share-wrap {
  width: 100%;
}
.share-url-wrap {
  margin-top: 16px;
}
.share-url-input {
  :deep(.el-input__inner) {
    cursor: pointer;
    color: var(--el-color-primary);
    user-select: all;
  }
}
</style>
