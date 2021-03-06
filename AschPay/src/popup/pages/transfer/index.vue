<template>
  <section class="app-container">
    <navigation :title="$t('title.transfer')" />
    <el-form class="mt20" ref="form" :model="formData" :rules="formRules" label-position="top">
      <el-form-item :label="$t('label.ownerAccount')" prop="from">
        <el-select class="no-border" v-model="formData.from" style="width: 100%;">
          <el-option v-for="(item, index) in accounts"
                     :value="item.address"
                     :key="index"
                     :label="item.name"></el-option>
        </el-select>
      </el-form-item>
      <el-form-item :label="$t('label.toAddress')" prop="to">
        <el-input class="no-border" v-model="formData.to"></el-input>
      </el-form-item>
      <el-form-item :label="$t('label.tokenType')" prop="token">
        <el-select class="no-border" v-model="formData.token" style="width: 100%;">
          <el-option v-for="(item, index) in tokens"
                     :value="item.key"
                     :key="index"
                     :label="item.key"></el-option>
        </el-select>
      </el-form-item>
      <el-form-item :label="$t('label.amount')" prop="amount">
        <el-input class="no-border" v-model="formData.amount" type="text"></el-input>
      </el-form-item>
      <el-form-item>
        <el-button class="full-btn" type="primary" @click="onSubmit('form')">{{$t('button.send')}}</el-button>
      </el-form-item>
    </el-form>
    <!-- transfer info -->
    <div class="popup" v-show="popup">
      <div class="popup-container">
        <div class="close" @click="popup = false">
          <v-icon name="times" scale="1.4"></v-icon>
        </div>
        <div class="title">{{$t('title.sendDetail')}}</div>
        <div class="popup-body">
          <div class="item">
            <div class="label">{{$t('label.ownerAccount')}}</div>
            <div class="content">{{payName}}</div>
          </div>
          <div class="item">
            <div class="label">{{$t('label.toAddress')}}</div>
            <div class="content">{{formData.to}}</div>
          </div>
          <div class="item">
            <div class="label money-label">{{$t('label.amount')}}</div>
            <div class="content money-content"><span>{{formData.amount}}</span> {{this.formData.token}}</div>
          </div>
          <el-button class="full-btn" type="primary" @click="surePay">{{$t('button.surePay')}}</el-button>
        </div>
      </div>
    </div>
  </section>
</template>
<script>
import Navigation from '../../components/navigation'
import { mapState } from 'vuex'
import { NewTronWeb } from '../../utils/tools'
import utils from '../../../lib/utils'
import InternalMessage from '../../../messages/InternalMessage'
import * as InternalMessageTypes from '../../../messages/InternalMessageTypes'
export default {
  components: {
    Navigation
  },
  data () {
    const frommValidator = (rule, value, callback) => {
      if (value === '') {
        callback(new Error(this.$i18n.t('verify.ownerAccountNull')))
      } else {
        callback()
      }
    }
    const toValidator = (rule, value, callback) => {
      if (value === '') {
        callback(new Error(this.$i18n.t('verify.toAddressNull')))
      } else {
        callback()
      }
    }
    const tokenValidator = (rule, value, callback) => {
      if (value === '') {
        callback(new Error(this.$i18n.t('verify.tokenNull')))
      } else {
        callback()
      }
    }
    const amountValidator = (rule, value, callback) => {
      if (!(/^(-?\d+)(\.\d+)?$/.test(value))) {
        callback(new Error(this.$i18n.t('verify.number')))
      } else if (value * 1 > 0) {
        callback()
      } else {
        callback(new Error(this.$i18n.t('verify.numberGtZero')))
      }
    }
    return {
      popup: false,
      formData: {
        from: '',
        to: '',
        token: '',
        amount: 0
      },
      formRules: {
        from: { validator: frommValidator, trigger: 'blur' },
        to: { validator: toValidator, trigger: 'blur' },
        token: { validator: tokenValidator, trigger: 'blur' },
        amount: { validator: amountValidator, trigger: 'blur' }
      },
      tokens: [],
      assetKey: this.$route.params.assetKey ? this.$route.params.assetKey : ''
    }
  },
  computed: {
    ...mapState([
      'currentAccount'
    ]),
    ...mapState('wallet', [
      'accounts'
    ]),
    payName () {
      let account = this.accounts.find(ele => ele.address === this.formData.from)
      if (account) {
        return account.name
      } else {
        return ''
      }
    }
  },
  created () {
    this.formData.from = this.currentAccount.address
    this.loadTokens()
  },
  methods: {
    onSubmit (formName) {
      this.$refs[formName].validate(valid => {
        if (valid) {
          this.popup = true
        }
      })
    },
    surePay () {
      let amount = this.formData.amount
      if (this.formData.token === 'TRX') {
        // 转trx
        amount = utils.getTokenRawAmount(amount)
        this.transferTrx(amount)
      } else {
        // 转token
        this.transferToken(amount)
      }
    },
    async transferTrx (amount) {
      this.popup = false
      this.$store.commit('loading', true)
      try {
        let transaction = await NewTronWeb().transactionBuilder.sendTrx(this.formData.to, amount * 1, this.formData.from)
        const fromAccount = this.accounts.find(ele => ele.address === this.formData.from)
        if (!fromAccount) {
          this.$kalert({
            message: this.$i18n.t('alert.sendFail')
          })
          return false
        }
        const { result } = await InternalMessage.widthPayload(InternalMessageTypes.SIGNSENDTRANSACTION,
          {address: this.formData.from, transaction: transaction})
          .send()
        let success = result
        this.$store.commit('loading', false)
        if (success) {
          this.formData.to = ''
          this.formData.amount = 0
          this.$kalert({
            message: this.$i18n.t('alert.sendSuccess')
          })
        } else {
          this.$kalert({
            message: this.$i18n.t('alert.sendFail')
          })
        }
      } catch (e) {
        this.$store.commit('loading', false)
        this.$kalert({
          message: this.$i18n.t('alert.sendFail')
        })
      }
    },
    async transferToken (amount) {
      this.popup = false
      this.$store.commit('loading', true)
      try {
        let transaction = await NewTronWeb().transactionBuilder.sendToken(
          this.formData.to,
          amount * 1,
          this.formData.token,
          this.formData.from
        )
        const fromAccount = this.accounts.find(ele => ele.address === this.formData.from)
        if (!fromAccount) {
          this.$kalert({
            message: this.$i18n.t('alert.sendFail')
          })
          return false
        }
        const { result } = await InternalMessage.widthPayload(InternalMessageTypes.SIGNSENDTRANSACTION,
          {address: this.formData.from, transaction: transaction})
          .send()
        let success = result
        this.$store.commit('loading', false)
        if (success) {
          this.formData.to = ''
          this.formData.amount = 0
          this.$kalert({
            message: this.$i18n.t('alert.sendSuccess')
          })
        } else {
          this.$kalert({
            message: this.$i18n.t('alert.sendFail')
          })
        }
      } catch (err) {
        this.$store.commit('loading', false)
        this.$kalert({
          message: this.$i18n.t('alert.sendFail')
        })
      }
    },
    // 加载tokens
    async loadTokens () {
      this.$store.commit('loading', true)
      const accountData = await NewTronWeb().trx.getAccount(this.formData.from)
      if (accountData.asset) {
        this.tokens = [{ key: 'TRX', value: utils.getTokenAmount(accountData.balance) }].concat(accountData.asset)
      } else {
        this.tokens = [{ key: 'TRX', value: utils.getTokenAmount(accountData.balance) }]
      }
      this.$store.commit('loading', false)
      if (this.assetKey) {
        this.formData.token = this.assetKey
      } else {
        this.formData.token = this.tokens[0].key
      }
    }
  }
}
</script>
<style lang="scss" scoped>
@import "../../theme/v1/variable";
.popup{
  position: fixed;
  background-color: rgba(0, 0, 0, 0.5);
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 99;
}
.popup-container{
    width: 100%;
    position: absolute;
    bottom: 0;
    left: 0;
    padding: 20px;
    background-color: #fff;
    .close{
      position: absolute;
      right: 20px;
      top: 5px;
      font-size: 18px;
      cursor: pointer;
      color: $color-second;
    }
    .title{
      text-align: center;
      padding-bottom: 15px;
      font-size: 16px;
      border-bottom: 1px dashed #e6e6e6;
    }
    .item{
      display: flex;
      height: 50px;
      overflow: hidden;
      .label{
        width: 70px;
        white-space: nowrap;
        color: $color-second;
        display: flex;
        align-items: center;
      }
      .content{
        padding-left: 5px;
        flex: 1;
        overflow: hidden;
        display: flex;
        align-items: center;
      }
      .memo-content{
        max-height: 50px;
        line-height: 18px;
      }
      .money-label{
        color: $color-base;
      }
      .money-content{
        color: $primary-color;
        justify-content: flex-end;
        span{
          font-size: 20px;
          position: relative;
          top: -2px;
          padding-right: 5px;
        }
      }
    }
  }
</style>
