<script setup lang='ts'>
import { fetchLogin, fetchRegister, fetchResetPassword, fetchSendResetMail, fetchVerify, fetchVerifyAdmin } from '@/api'
import { useAuthStore } from '@/store'

const props = defineProps<Props>()

const emit = defineEmits<Emit>()

const { t } = useI18n()

interface Props {
  visible: boolean
}

interface Emit {
  (e: 'update:visible', visible: boolean): void
}

const show = computed({
  get: () => props.visible,
  set: (visible: boolean) => emit('update:visible', visible),
})

const route = useRoute()
const router = useRouter()
const authStore = useAuthStore()

const ms = useMessage()

const loading = ref(false)
const username = ref('')
const password = ref('')
const sign = ref('')
const token = ref('')
const need2FA = ref(false)

const disabled = computed(() => !username.value.trim() || !password.value.trim() || loading.value)

const activeTab = ref('login')

const showConfirmPassword = ref(false)
const confirmPassword = ref('')

function handlePasswordInput() {
  showConfirmPassword.value = password.value.trim() !== ''
}

const confirmPasswordStatus = computed(() => {
  if (!password.value || !confirmPassword.value)
    return undefined
  return password.value !== confirmPassword.value ? 'error' : 'success'
})

onMounted(async () => {
  const verifytoken = route.query.verifytoken as string
  await handleVerify(verifytoken)
  const verifytokenadmin = route.query.verifytokenadmin as string
  await handleVerifyAdmin(verifytokenadmin)
  sign.value = route.query.verifyresetpassword as string
  if (sign.value) {
    username.value = sign.value.split('-')[0].split('|')[0]
    activeTab.value = 'resetPassword'
    show.value = true
  }
})

async function handleVerify(verifytoken: string) {
  if (!verifytoken)
    return
  const secretKey = verifytoken.trim()

  try {
    loading.value = true
    const result = await fetchVerify(secretKey)
    ms.success(result.message as string)
    router.replace('/')
  }
  catch (error: any) {
    ms.error(error.message ?? 'error')
    authStore.removeToken()
  }
  finally {
    loading.value = false
  }
}

async function handleVerifyAdmin(verifytoken: string) {
  if (!verifytoken)
    return
  const secretKey = verifytoken.trim()

  try {
    loading.value = true
    await fetchVerifyAdmin(secretKey)
    ms.success('开通成功 | Activate successfully')
    router.replace('/')
  }
  catch (error: any) {
    ms.error(error.message ?? 'error')
  }
  finally {
    loading.value = false
  }
}

function handlePress(event: KeyboardEvent) {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    handleLogin()
  }
}

async function handleLogin() {
  const name = username.value.trim()
  const pwd = password.value.trim()
  if (!name || !pwd)
    return

  try {
    loading.value = true
    const result = await fetchLogin(name, pwd, token.value)
    if (result.data.need2FA) {
      need2FA.value = true
      ms.warning(result.message as string)
      return
    }
    await authStore.setToken(result.data.token)
    ms.success(result.message as string)
    window.location.reload()
  }
  catch (error: any) {
    ms.error(error.message ?? 'error')
    password.value = ''
  }
  finally {
    loading.value = false
  }
}

async function handleRegister() {
  const name = username.value.trim()
  const pwd = password.value.trim()
  const confirmPwd = confirmPassword.value.trim()

  if (!name || !pwd || !confirmPwd || pwd !== confirmPwd) {
    ms.error('两次输入的密码不一致 | Passwords don\'t match')
    return
  }

  try {
    loading.value = true
    const result = await fetchRegister(name, pwd)
    ms.success(result.message as string)
  }
  catch (error: any) {
    ms.error(error.message ?? 'error')
  }
  finally {
    loading.value = false
  }
}

async function handleSendResetMail() {
  const name = username.value.trim()

  if (!name)
    return

  try {
    loading.value = true
    const result = await fetchSendResetMail(name)
    ms.success(result.message as string)
  }
  catch (error: any) {
    ms.error(error.message ?? 'error')
  }
  finally {
    loading.value = false
  }
}

async function handleResetPassword() {
  const name = username.value.trim()
  const pwd = password.value.trim()
  const confirmPwd = confirmPassword.value.trim()

  if (!name || !pwd || !confirmPwd || pwd !== confirmPwd) {
    ms.error('两次输入的密码不一致 | Passwords don\'t match')
    return
  }

  try {
    loading.value = true
    const result = await fetchResetPassword(name, pwd, sign.value)
    ms.success(result.message as string)
  }
  catch (error: any) {
    ms.error(error.message ?? 'error')
  }
  finally {
    loading.value = false
  }
}
</script>

<template>
  <NModal v-model:show="show" style="width: 90%; max-width: 440px">
    <div class="p-10 bg-white rounded-sm dark:bg-slate-800">
      <div class="space-y-4">
        <header class="space-y-2">
          <h2 class="text-2xl font-bold text-center text-slate-800 dark:text-neutral-200">
            {{ t('common.notLoggedIn') }}
          </h2>
        </header>

        <!-- Add Tabs -->
        <NTabs v-model:value="activeTab" type="line">
          <NTabPane name="login" :tab="t('common.login')">
            <NInput v-model:value="username" type="text" :placeholder="t('common.email')" class="mb-2" />
            <NInput v-model:value="password" type="password" :placeholder="t('common.password')" class="mb-2" @keypress="handlePress" />
            <NInput v-if="need2FA" v-model:value="token" type="text" :placeholder="t('common.twoFA')" class="mb-2" @keypress="handlePress" />
            <NButton block type="primary" :disabled="disabled" :loading="loading" @click="handleLogin">
              {{ t('common.login') }}
            </NButton>
          </NTabPane>

          <NTabPane v-if="authStore.session && authStore.session.allowRegister" name="register" :tab="t('common.register')">
            <NInput v-model:value="username" type="text" :placeholder="t('common.email')" class="mb-2" />
            <NInput v-model:value="password" type="password" :placeholder="t('common.password')" class="mb-2" @input="handlePasswordInput" />
            <NInput
              v-if="showConfirmPassword"
              v-model:value="confirmPassword"
              type="password"
              :placeholder="t('common.passwordConfirm')"
              class="mb-4"
              :status="confirmPasswordStatus"
            />

            <NButton block type="primary" :disabled="disabled || password !== confirmPassword" :loading="loading" @click="handleRegister">
              {{ t('common.register') }}
            </NButton>
          </NTabPane>

          <NTabPane name="resetPassword" :tab="t('common.resetPassword')">
            <NInput v-model:value="username" :disabled="sign !== undefined" type="text" :placeholder="t('common.email')" class="mb-2" />
            <NInput v-if="!!sign" v-model:value="password" type="password" :placeholder="t('common.password')" class="mb-2" @input="handlePasswordInput" />
            <NInput
              v-if="showConfirmPassword"
              v-model:value="confirmPassword"
              type="password"
              :placeholder="t('common.passwordConfirm')"
              class="mb-4"
              :status="confirmPasswordStatus"
            />
            <NButton v-if="!sign" block type="primary" :disabled="username.length <= 0" :loading="loading" @click="handleSendResetMail">
              {{ t('common.resetPasswordMail') }}
            </NButton>
            <NButton v-else block type="primary" :disabled="disabled || password !== confirmPassword" :loading="loading" @click="handleResetPassword">
              {{ t('common.resetPassword') }}
            </NButton>
          </NTabPane>
        </NTabs>
        <!-- End Tabs -->
      </div>
    </div>
  </NModal>
</template>
