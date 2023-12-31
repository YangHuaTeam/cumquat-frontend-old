<template>
  <div>
    <v-scroll-x-transition>
      <v-form v-if="!isSubmitted" ref="infoForm" elevation="0" lazy-validation>
        <v-text-field
          ref="usrField"
          v-model="usernameData"
          label="用户名"
          hint="大小写A-z, 数字0-9, 下划线_, ."
          :rules="userNameRule"
          :error="usernameErr"
          :error-count="usrErrCount"
          :error-messages="usrErrMsg"
          maxlength="35"
          outlined
        />
        <v-text-field
          ref="pwdField"
          v-model="originPassword"
          label="密码"
          hint="注册时设置的密码"
          :type="passwdShow ? 'text' : 'password'"
          :append-icon="passwdShow ? 'mdi-eye' : 'mdi-eye-off'"
          :rules="passwordRule"
          :error="passwordErr"
          :error-count="pwdErrCount"
          :error-messages="pwdErrMsg"
          maxlength="128"
          outlined
          @click:append="passwdShow = !passwdShow"
        />
        <v-btn
          id="submitBtn"
          ref="submitBtn"
          rounded
          block
          elevation="1"
          color="primary"
          :loading="isSubmitting"
          :disabled="isSubmitting"
          @click="submit"
        >
          登录
        </v-btn>
      </v-form>
    </v-scroll-x-transition>
    <v-scroll-x-transition>
      <v-form v-if="isSubmitted && transitionPlayed" ref="succeedForm" class="succeed-form" elevation="0">
        <v-icon size="120" color="grey darken-2">
          mdi-check
        </v-icon>
        <p>登录成功</p>
        <v-btn
          ref="backHomeBtn"
          rounded
          block
          elevation="1"
          color="primary"
          to="/table"
        >
          返回首页
        </v-btn>
      </v-form>
    </v-scroll-x-transition>
  </div>
</template>

<script>
import SessionUtils from '~/utils/SessionUtils.vue'
import HttpUtils from '~/utils/HttpUtils.vue'
export default {
  name: 'LoginForm',
  mixins: [SessionUtils, HttpUtils],
  data: () => ({
    isSubmitting: false,
    isSubmitted: false,
    passwdShow: false,
    usernameData: '',
    originPassword: '',
    usernameErr: false,
    passwordErr: false,
    transitionPlayed: false,
    usrErrCount: 1,
    pwdErrCount: 1,
    usrErrMsg: '',
    pwdErrMsg: '',
    userNameRule: [
      value => !!value || '用户名 为必填项', // 没名没姓
      (value) => {
        const emailPattern = /\w[-\w.+]*@([A-Za-z0-9][-A-Za-z0-9]+\.)+[A-Za-z]/
        const userNamePattern = /\w([A-Za-z0-9][-A-Za-z0-9]*)/
        if (emailPattern.test(value)) {
          return true
        } else if (userNamePattern.test(value)) {
          return true
        } else {
          return false || '用户名不合法, 请检查'
        }
      }
    ],
    passwordRule: [
      value => !!value || '密码 为必填项' // 门 户 开 放
    ]
  }),
  methods: {
    submit () {
      const cryptoInstance = require('crypto')
      const validateResult = this.$refs.infoForm.validate()
      this.$refs.submitBtn.$el.style = ''
      if (validateResult === true) {
        this.isSubmitting = !this.isSubmitting
      } else {
        this.$refs.submitBtn.$el.style = 'background-color: #EE6363 !important; border-color: #EE6363 !important;'
        setTimeout(this.clearBtnStyle, 1500)
        return false
      }
      const usernameData = this.$data.usernameData
      const originPassword = this.$data.originPassword
      const saltPassword = originPassword + '81262035b6d4babbcfd8fe71892edced'
      const sha512 = cryptoInstance.createHash('sha512')
      const saltPasswordSHA512 = sha512.update(saltPassword).digest('hex')
      const dataObj = {}
      dataObj.username = usernameData
      dataObj.password = saltPasswordSHA512
      this.sendPostToApi('account', '/login', dataObj, this.reqDataCallback, false)
    },
    reqDataCallback (requestDataReturn) {
      if (requestDataReturn.isError === false) { // If log-in is working
        if (requestDataReturn.data.code === 1000) { // 为了更好的观感以及便于修改, 此处没有使用 &&
          this.sendGetToApi('account', '/overview', '', this.initializeUserSession, false)
        }
      } else if (requestDataReturn.isError === true) { // If log-in isn't working
        if (requestDataReturn.code === 401) { // Code is 1001
          this.showResult('error', '服务异常, 可能是源代码逻辑错误, 或者后端出现问题', true, true, '服务异常, 请检查后端服务器')
        } else if (requestDataReturn.code === 403) { // Code is either 1001.1 or 1001.2
          if (requestDataReturn.data.code === 1001.1) { // Either user's pwd or usrname is wrong
            if (requestDataReturn.data.data.wrong === 'username') {
              this.showResult(undefined, undefined, true, false, '用户名不存在')
            } else if (requestDataReturn.data.data.wrong === 'password') {
              this.showResult(undefined, undefined, false, true, '密码错误')
            }
          } else if (requestDataReturn.data.code === 1001.2) { // Not suitable for concurrent situation
            this.showResult('error', 'API结构异常, 请检查后端', true, true, '结构异常, 请检查后端状态')
          }
        } else if (requestDataReturn.code === 500) { // Code is 1002
          this.showResult('error', '服务器内部错误, 请重试', true, true, '服务器内部错误, 请重试')
        } else if (requestDataReturn.code === 404) { // Code is 1003, or server is not running correctly
          if (requestDataReturn.data.code === 1003) {
            this.showResult('error', 'API结构异常, 请检查后端', true, true, '结构异常, 请检查后端状态')
          } else {
            this.showResult('error', 'API未正常运行, 请检查后端状态', true, true, 'API未正常运行, 请检查后端服务器')
          }
        } else if (requestDataReturn.code === 422) { // Code is either 1003 or 1003.1
          if (requestDataReturn.data.code === 1003) {
            this.showResult('error', '参数错误, 请检查后端, 是否与前端版本不匹配或发生其他问题?', true, true, '参数错误, 请检查后端状态')
          } else if (requestDataReturn.data.code === 1003.1) {
            this.showResult('error', '参数校验异常, 请检查输入内容合法性', true, true, '参数校验异常, 请检查输入内容合法性')
          }
        } else if (requestDataReturn.code === 423) { // Code is 1004
          this.showResult('warning', '请求过于频繁, 您的IP已被记录, 请稍后再试', true, true, '请求过于频繁, 请稍后再试')
          setTimeout(() => {
            this.showResult('warning', '由于恶意请求, 您的IP已被记录, 继续尝试可能导致被封禁, 请慎重', true, true, '请求过于频繁, 请稍后再试')
          }, 5000)
        } else if (requestDataReturn.code === -1) {
          this.showResult('error', '网络错误, 请重试', true, true, '网络错误, 请重试')
        } else {
          this.showResult('error', '发生未知错误, 请重试', true, true, '发生未知错误, 请重试')
        }
      }
    },
    initializeUserSession (userInfoReturn) {
      const userInfoCookie = {}
      userInfoCookie.uid = userInfoReturn.data.data.id
      userInfoCookie.username = userInfoReturn.data.data.username
      userInfoCookie.email = userInfoReturn.data.data.email
      userInfoCookie.status = userInfoReturn.data.data.status
      userInfoCookie.avatarSrc = userInfoReturn.data.data.avatarSrc
      const userInfoCookieStr = JSON.stringify(userInfoCookie) // Stringify The Cookie Obj in order to storage normally
      this.editCookieValue('userInfo', userInfoCookieStr, 259200)
      const cryptoInstance = require('crypto')
      const md5 = cryptoInstance.createHash('md5')
      const generateFrontendTempSession = () => {
        const curDate = new Date()
        const timeStamp = curDate.getTime()
        const saltedSessionID = userInfoCookie.email + timeStamp
        const saltedSessionIDMD5 = md5.update(saltedSessionID).digest('hex')
        return saltedSessionIDMD5
      }
      const frontendTempSession = generateFrontendTempSession()
      this.editCookieValue('frontendTempSession', frontendTempSession, 259200)
      this.actSubmitSucceed()
    },
    actSubmitSucceed () {
      this.$data.isSubmitted = true
      setTimeout(() => {
        this.$data.transitionPlayed = true
      }, 500)
      this.$emit('submitSucceed')
    },
    clearBtnStyle () {
      this.$refs.submitBtn.$el.style = ''
    },
    showResult (snackBarType, snackBarMsg, usrErr, pwdErr, errMsg) {
      if (snackBarType !== undefined) {
        this.$parent.$parent.$parent.$emit('showSnackBar', snackBarType, snackBarMsg, '5000', true)
      }
      this.$data.usernameErr = usrErr
      this.$data.passwordErr = pwdErr
      if (usrErr && pwdErr) {
        this.$data.pwdErrMsg = errMsg
        this.$data.usrErrCount = 1
        this.$data.pwdErrCount = 1
      } else if (usrErr && !pwdErr) {
        this.$data.usrErrMsg = errMsg
        this.$data.usrErrCount = 1
        this.$data.pwdErrCount = 0
      } else if (!usrErr && pwdErr) {
        this.$data.pwdErrMsg = errMsg
        this.$data.usrErrCount = 0
        this.$data.pwdErrCount = 1
      }
      this.$data.isSubmitting = false
      this.$refs.submitBtn.$el.style = 'background-color: #EE6363 !important; border-color: #EE6363 !important;'
      setTimeout(this.clearBtnStyle, 1500)
    }
  }
}
</script>

<style>
.v-form {
  margin: 5px auto;
  max-width: 35%;
  min-width: 300px;
}

.v-btn {
  margin: 0;
  transition: all 0.5s;
}

.v-input {
  margin-top: 5px;
}

.succeed-form {
  text-align: center;
}

.succeed-form p {
  font-size: 22px;
  opacity: .5;
}
</style>
