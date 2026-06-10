<!-- eslint-disable vue/multi-word-component-names -->
<template>
  <div class="login-page">
    <!-- 语言切换 - 右上角醒目 -->
    <div class="lang-bar">
      <LangSwitcher variant="light" />
    </div>

    <div class="main-container">
      <div class="left-bg">
        <div class="bg-overlay">
          <div class="shopping-text">CASHIER</div>
        </div>
      </div>

      <div class="right-form">
        <div class="login-card">
          <h2 class="card-title">{{ $t('login.title') }}</h2>
          <p class="card-subtitle"></p>

          <el-form ref="formref" :model="form" :rules="rules" label-width="auto">
            <el-form-item prop="username">
              <el-input v-model="form.username" :placeholder="$t('login.usernamePlaceholder')">
                <template #prefix>
                  <el-icon><User /></el-icon>
                </template>
              </el-input>
            </el-form-item>

            <el-form-item prop="password">
              <el-input
                v-model="form.password"
                type="password"
                :placeholder="$t('login.passwordPlaceholder')"
                show-password
              >
                <template #prefix>
                  <el-icon><Lock /></el-icon>
                </template>
              </el-input>
            </el-form-item>

            <!-- 记住密码 + 天数 -->
            <el-form-item>
              <div style="display: flex; align-items: center; gap: 8px">
                <el-checkbox v-model="rememberMe">{{ $t('login.rememberMe') }}</el-checkbox>
                <el-select
                  v-model="rememberDays"
                  size="small"
                  style="width: 100px"
                  :disabled="!rememberMe"
                >
                  <el-option :label="$t('login.days7')" :value="7"></el-option>
                  <el-option :label="$t('login.days15')" :value="15"></el-option>
                  <el-option :label="$t('login.days30')" :value="30"></el-option>
                </el-select>
              </div>
            </el-form-item>

            <el-form-item>
              <el-button
                type="primary"
                class="login-btn"
                :loading="captchaLoading"
                :disabled="!captchaReady"
                @click="onSubmit"
              >{{ $t('login.loginBtn') }}</el-button>
            </el-form-item>
          </el-form>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref, onMounted } from 'vue'
import { useI18n } from 'vue-i18n'
import { User, Lock } from '@element-plus/icons-vue'
import { ElNotification } from 'element-plus'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores/user'
import request from '@/utils/request'
import LangSwitcher from '@/components/LangSwitcher.vue'

const { t } = useI18n()

const router = useRouter()
const userStore = useUserStore()

// 👇 只加了这两个变量
const rememberMe = ref(false)
const rememberDays = ref(7)

// 腾讯云验证码状态
const captchaReady = ref(false)
const captchaLoading = ref(false)
let captchaInstance = null
let pendingCaptchaResolve = null

const form = reactive({
  username: 'admin',
  password: '123456',
})

const rules = {
  username: [{ required: true, message: t('login.usernamePlaceholder'), trigger: 'blur' }],
  password: [{ required: true, message: t('login.passwordPlaceholder'), trigger: 'blur' }],
}

const formref = ref(null)

// 后端实际角色码 → 前端内部角色标识映射
const ROLE_MAP = {
  admin_global: 'ADMIN',
  admin_store: 'STORE_ADMIN',
  cashier: 'CASHIER',
  waiter: 'WAITER',
  purchaser: 'PURCHASER',
  buyer: 'PURCHASER',
  chef: 'CHEF',
}

// 初始化腾讯云验证码实例
// 开发模式默认跳过验证码，需要测试真实验证码时改为 false
const isDev = import.meta.env.DEV
const SKIP_CAPTCHA_IN_DEV = true   // 开发跳过验证码
const CAPTCHA_APP_ID = '195210184'

// 容灾：JS 加载失败时生成容灾票据（trerror_ 前缀），后端校验时可放行
const loadErrorCallback = () => {
  const ticket = 'trerror_1001_' + CAPTCHA_APP_ID + Math.floor(new Date().getTime() / 1000)
  const randstr = '@' + Math.random().toString(36).substr(2)
  console.warn('验证码 JS 加载失败，使用容灾票据')
  if (pendingCaptchaResolve) {
    pendingCaptchaResolve({ ticket, randstr, errorCode: 1001 })
    pendingCaptchaResolve = null
  }
  captchaLoading.value = false
}

// 动态加载 TJCaptcha.js
const loadCaptchaScript = () => {
  return new Promise((resolve) => {
    if (window.TencentCaptcha) {
      resolve()
      return
    }
    const script = document.createElement('script')
    script.src = 'https://turing.captcha.qcloud.com/TJCaptcha.js'
    script.onload = () => resolve()
    script.onerror = () => {
      console.error('TJCaptcha.js 加载失败')
      resolve() // 不阻塞流程，后续走容灾
    }
    document.head.appendChild(script)
  })
}

const initCaptcha = async () => {
  if (isDev && SKIP_CAPTCHA_IN_DEV) {
    captchaReady.value = true
    return
  }

  await loadCaptchaScript()
  if (!window.TencentCaptcha) {
    console.warn('验证码 SDK 不可用，将使用容灾票据')
    captchaReady.value = true
    return
  }

  captchaReady.value = true
}

// 弹出验证码弹窗，返回 Promise<{ticket, randstr, errorCode?} | null>
const showCaptcha = () => {
  if (isDev && SKIP_CAPTCHA_IN_DEV) {
    return Promise.resolve({ ticket: 'dev-bypass', randstr: 'dev-bypass' })
  }

  return new Promise((resolve) => {
    pendingCaptchaResolve = resolve

    if (!window.TencentCaptcha) {
      loadErrorCallback()
      return
    }

    try {
      captchaInstance = new window.TencentCaptcha(CAPTCHA_APP_ID, (res) => {
        // res.ret === 0 : 验证通过
        // res.ret === 2 : 用户关闭弹窗
        // res.errorCode : 容灾错误码
        if (res.ret === 0 && pendingCaptchaResolve) {
          // 容灾票据也放行（errorCode 存在时）
          if (res.errorCode) {
            console.warn('容灾票据，errorCode=' + res.errorCode)
          }
          pendingCaptchaResolve({ ticket: res.ticket, randstr: res.randstr, errorCode: res.errorCode })
        } else if (res.ret === 2 && pendingCaptchaResolve) {
          pendingCaptchaResolve(null)
        }
        pendingCaptchaResolve = null
        captchaLoading.value = false
      })
      captchaInstance.show()
    } catch (error) {
      // JS 初始化异常，走容灾
      console.error('验证码初始化异常：', error)
      loadErrorCallback()
    }
  })
}

// 👇 只加了这一段
onMounted(() => {
  const data = localStorage.getItem('loginRemember')
  if (!data) {
    initCaptcha()
    return
  }
  try {
    const { username, password, expire } = JSON.parse(data)
    if (Date.now() > expire) {
      localStorage.removeItem('loginRemember')
    } else {
      form.username = username
      form.password = password
      // rememberMe.value = true
    }
  } catch (e) {
    // ignore
  }
  initCaptcha()
})

const onSubmit = async () => {
  // 表单校验（Element Plus validate 返回 Promise）
  try {
    await formref.value.validate()
  } catch {
    return
  }

  // 弹出腾讯云滑块验证码
  captchaLoading.value = true
  const captchaResult = await showCaptcha()

  if (!captchaResult) {
    // 用户关闭了验证码弹窗
    ElNotification.warning('请完成滑块验证')
    return
  }

  // 滑块验证通过，提交登录
  try {
    const res = await request.post('/users/login', {
      username: form.username,
      password: form.password,
      ticket: captchaResult.ticket,
      randstr: captchaResult.randstr,
    })

    if (res.code === 200) {
      ElNotification.success(t('login.loginSuccess'))

      // 👇 只加了这一段
      if (rememberMe.value) {
        const expire = Date.now() + rememberDays.value * 86400000
        localStorage.setItem(
          'loginRemember',
          JSON.stringify({
            username: form.username,
            password: form.password,
            expire,
          }),
        )
      } else {
        localStorage.removeItem('loginRemember')
      }

      const { token, roleCode, userId, username, realName, storeId } = res.data

      const internalRole = ROLE_MAP[roleCode] || roleCode

      userStore.setUserInfo(token, internalRole, {
        userId,
        username,
        realName,
        storeId,
      })

      const rolePathMap = {
        ADMIN: '/admin',
        STORE_ADMIN: '/manager',
        CASHIER: '/cashier',
        WAITER: '/waiter',
        PURCHASER: '/buyer',
        CHEF: '/chef',
      }
      const targetPath = rolePathMap[internalRole]
      if (targetPath) {
        router.push(targetPath)
      } else {
        ElNotification.error(`未知角色：${roleCode}`)
      }
    } else {
      ElNotification.error(res.msg || t('login.loginFailed'))
    }
  } catch (err) {
    ElNotification.error(t('login.loginFailed'))
    console.error(err)
  }
}
</script>

<style>
/* 全局取消边距 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
html,
body,
#app {
  width: 100%;
  height: 100%;
  overflow-x: hidden;
}
</style>

<style scoped>
.login-page {
  width: 100%;
  min-height: 100vh;
  background: linear-gradient(to right, rgba(173, 216, 230, 0.3), rgba(255, 192, 203, 0.3));
  overflow-x: hidden;
  position: relative;
}

.lang-bar {
  position: absolute;
  top: 20px;
  right: 30px;
  z-index: 200;
}

.nav-bar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  padding: 20px 40px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  z-index: 100;
  flex-wrap: wrap;
}
.logo {
  font-size: 24px;
  font-weight: bold;
}
.nav-links {
  display: flex;
  gap: 20px;
  flex-wrap: wrap;
}
.nav-links span {
  cursor: pointer;
  white-space: nowrap;
}
.nav-links span:hover {
  color: #409eff;
}

.main-container {
  display: flex;
  width: 100%;
  min-height: 100vh;
}

.left-bg {
  flex: 2;
  background: url('@/assets/登录首页.png') center/cover no-repeat;
  min-height: 100vh;
  position: relative;
}
.bg-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
.shopping-text {
  font-size: 64px;
  color: #fff;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
}

.right-form {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.login-card {
  width: 300px;
  padding: 30px;
  background: rgba(255, 255, 255, 0.8);
  border-radius: 12px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.05);
}

.card-title {
  font-size: 20px;
  margin-bottom: 5px;
  text-align: center;
  color: #333;
}
.card-subtitle {
  font-size: 13px;
  color: #666;
  margin-bottom: 30px;
  text-align: center;
}

.login-btn {
  width: 100%;
  background: linear-gradient(to right, #a8edea, #fed6e3);
  border: none;
  color: #333;
  font-weight: 500;
  transition: all 0.3s;
}
.login-btn:hover {
  opacity: 0.9;
  transform: scale(0.98);
}

@media screen and (max-width: 768px) {
  .nav-bar {
    padding: 15px 20px;
    background: rgba(255, 255, 255, 0.6);
    backdrop-filter: blur(4px);
  }
  .main-container {
    flex-direction: column;
  }
  .left-bg {
    min-height: 40vh;
    height: 40vh;
  }
  .shopping-text {
    font-size: 36px;
  }
  .right-form {
    align-items: flex-start;
    padding: 40px 20px;
  }
  .login-card {
    width: 100%;
    max-width: 320px;
    margin: 0 auto;
  }
}
</style>
