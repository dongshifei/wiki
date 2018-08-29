<template lang="pug">
  v-app
    .login
      .login-container(:class='{ "is-expanded": strategies.length > 1, "is-loading": isLoading }')
        .login-mascot
          img(src='/svg/henry-reading.svg', alt='Henry')
        .login-providers(v-show='strategies.length > 1')
          button(v-for='strategy in strategies', :class='{ "is-active": strategy.key === selectedStrategy }', @click='selectStrategy(strategy.key, strategy.useForm)', :title='strategy.title')
            em(v-html='strategy.icon')
            span {{ strategy.title }}
          .login-providers-fill
        .login-frame(v-show='screen === "login"')
          h1.text-xs-center.display-1 {{ siteTitle }}
          h2.text-xs-center.subheading {{ $t('auth:loginRequired') }}
          v-text-field(solo, hide-details, ref='iptEmail', v-model='username', :placeholder='$t("auth:fields.emailUser")')
          v-text-field.mt-2(
            solo
            hide-details
            ref='iptPassword'
            v-model='password'
            :append-icon='hidePassword ? "visibility" : "visibility_off"'
            :append-icon-cb='() => (hidePassword = !hidePassword)'
            :type='hidePassword ? "password" : "text"'
            :placeholder='$t("auth:fields.password")'
            @keyup.enter='login'
          )
          v-btn.mt-3(block, large, color='primary', @click='login') {{ $t('auth:actions.login') }}
        .login-frame(v-show='screen === "tfa"')
          .login-frame-icon
            svg.icons.is-48(role='img')
              title {{ $t('auth:tfa.title') }}
              use(xlink:href='#nc-key')
          h2 {{ $t('auth:tfa.subtitle') }}
          input(type='text', ref='iptTFA', v-model='securityCode', :placeholder='$t("auth:tfa.placeholder")', @keyup.enter='verifySecurityCode')
          button.button.is-blue.is-fullwidth(@click='verifySecurityCode')
            span {{ $t('auth:tfa.verifyToken') }}
    nav-footer(altbg)
</template>

<script>
/* global siteConfig */

import _ from 'lodash'
import { mapState } from 'vuex'

import strategiesQuery from 'gql/login/login-query-strategies.gql'
import loginMutation from 'gql/login/login-mutation-login.gql'
import tfaMutation from 'gql/login/login-mutation-tfa.gql'

export default {
  i18nOptions: { namespaces: 'auth' },
  data () {
    return {
      error: false,
      strategies: [],
      selectedStrategy: 'local',
      screen: 'login',
      username: '',
      password: '',
      hidePassword: true,
      securityCode: '',
      loginToken: '',
      isLoading: false
    }
  },
  computed: {
    ...mapState(['notification']),
    notificationState: {
      get() { return this.notification.isActive },
      set(newState) { this.$store.commit('updateNotificationState', newState) }
    },
    siteTitle () {
      return siteConfig.title
    }
  },
  mounted () {
    this.$refs.iptEmail.focus()
  },
  methods: {
    /**
     * SELECT STRATEGY
     */
    selectStrategy (key, useForm) {
      this.selectedStrategy = key
      this.screen = 'login'
      if (!useForm) {
        this.isLoading = true
        window.location.assign(this.$helpers.resolvePath('login/' + key))
      } else {
        this.$refs.iptEmail.focus()
      }
    },
    /**
     * LOGIN
     */
    async login () {
      if (this.username.length < 2) {
        this.$store.commit('showNotification', {
          style: 'red',
          message: 'Enter a valid email / username.',
          icon: 'warning'
        })
        this.$refs.iptEmail.focus()
      } else if (this.password.length < 2) {
        this.$store.commit('showNotification', {
          style: 'red',
          message: 'Enter a valid password.',
          icon: 'warning'
        })
        this.$refs.iptPassword.focus()
      } else {
        this.isLoading = true
        try {
          let resp = await this.$apollo.mutate({
            mutation: loginMutation,
            variables: {
              username: this.username,
              password: this.password,
              strategy: this.selectedStrategy
            }
          })
          if (_.has(resp, 'data.authentication.login')) {
            let respObj = _.get(resp, 'data.authentication.login', {})
            if (respObj.responseResult.succeeded === true) {
              if (respObj.tfaRequired === true) {
                this.screen = 'tfa'
                this.securityCode = ''
                this.loginToken = respObj.tfaLoginToken
                this.$nextTick(() => {
                  this.$refs.iptTFA.focus()
                })
              } else {
                this.$store.commit('showNotification', {
                  message: 'Login Successful! Redirecting...',
                  style: 'success',
                  icon: 'check'
                })
                _.delay(() => {
                  window.location.replace('/') // TEMPORARY - USE RETURNURL
                }, 1000)
              }
              this.isLoading = false
            } else {
              throw new Error(respObj.responseResult.message)
            }
          } else {
            throw new Error('Authentication is unavailable.')
          }
        } catch (err) {
          console.error(err)
          this.$store.commit('showNotification', {
            style: 'red',
            message: err.message,
            icon: 'warning'
          })
          this.isLoading = false
        }
      }
    },
    /**
     * VERIFY TFA CODE
     */
    verifySecurityCode () {
      if (this.securityCode.length !== 6) {
        this.$store.commit('showNotification', {
          style: 'red',
          message: 'Enter a valid security code.',
          icon: 'warning'
        })
        this.$refs.iptTFA.focus()
      } else {
        this.isLoading = true
        this.$apollo.mutate({
          mutation: tfaMutation,
          variables: {
            loginToken: this.loginToken,
            securityCode: this.securityCode
          }
        }).then(resp => {
          if (_.has(resp, 'data.authentication.loginTFA')) {
            let respObj = _.get(resp, 'data.authentication.loginTFA', {})
            if (respObj.responseResult.succeeded === true) {
              this.$store.commit('showNotification', {
                message: 'Login successful!',
                style: 'success',
                icon: 'check'
              })
              _.delay(() => {
                window.location.replace('/') // TEMPORARY - USE RETURNURL
              }, 1000)
              this.isLoading = false
            } else {
              throw new Error(respObj.responseResult.message)
            }
          } else {
            throw new Error('Authentication is unavailable.')
          }
        }).catch(err => {
          console.error(err)
          this.$store.commit('showNotification', {
            style: 'red',
            message: err.message,
            icon: 'warning'
          })
          this.isLoading = false
        })
      }
    }
  },
  apollo: {
    strategies: {
      query: strategiesQuery,
      update: (data) => data.authentication.strategies,
      watchLoading (isLoading) {
        this.isLoading = isLoading
        this.$store.commit(`loading${isLoading ? 'Start' : 'Stop'}`, 'login-strategies-refresh')
      }
    }
  }
}
</script>

<style lang="scss">
  .login {
    background-color: mc('blue', '800');
    background-image: url('../static/svg/motif-blocks.svg');
    background-repeat: repeat;
    background-size: 200px;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    animation: loginBgReveal 20s linear infinite;

    @include keyframes(loginBgReveal) {
      0% {
        background-position-y: 0;
      }
      100% {
        background-position-y: -800px;
      }
    }

    &::before {
      content: '';
      position: absolute;
      background-color: #0d47a1;
      background-image: url('../static/svg/motif-overlay.svg');
      background-attachment: fixed;
      background-size: cover;
      opacity: .5;
      top: 0;
      left: 0;
      width: 100vw;
      height: 100vh;
    }

    &::after {
      content: '';
      position: absolute;
      background-image: linear-gradient(to bottom, rgba(mc('blue', '800'), .9) 0%, rgba(mc('blue', '800'), 0) 100%);
      top: 0;
      left: 0;
      width: 100vw;
      height: 25vh;
      z-index: 1;
    }

    &-mascot {
      width: 200px;
      height: 200px;
      position: absolute;
      top: -180px;
      left: 50%;
      margin-left: -100px;
      z-index: 10;

      @include until($tablet) {
        display: none;
      }
    }

    &-container {
      position: relative;
      display: flex;
      width: 400px;
      align-items: stretch;
      box-shadow: 0 14px 28px rgba(0,0,0,0.2);
      border-radius: 6px;
      animation: zoomIn .5s ease;
      z-index: 2;

      &::after {
        position: absolute;
        top: 1rem;
        right: 1rem;
        content: " ";
        @include spinner(mc('blue', '500'),0.5s,16px);
        display: none;
      }

      &.is-expanded {
        width: 650px;

        .login-frame {
          border-radius: 0 6px 6px 0;
          border-left: none;

          @include until($tablet) {
            border-radius: 0;
          }
        }
      }

      &.is-loading::after {
        display: block;
      }

      @include until($tablet) {
        width: 95vw;
        border-radius: 0;

        &.is-expanded {
          width: 95vw;
        }
      }
    }

    &-providers {
      display: flex;
      flex-direction: column;
      width: 250px;

      border-right: none;
      border-radius: 6px 0 0 6px;
      z-index: 1;
      overflow: hidden;

      @include until($tablet) {
        width: 50px;
        border-radius: 0;
      }

      button {
        flex: 0 1 50px;
        padding: 5px 15px;
        border: none;
        color: #FFF;
        // background: linear-gradient(to right, rgba(mc('light-blue', '800'), .7), rgba(mc('light-blue', '800'), 1));
        // border-top: 1px solid rgba(mc('light-blue', '900'), .5);
        background: linear-gradient(to right, rgba(0,0,0, .5), rgba(0,0,0, .7));
        border-top: 1px solid rgba(0,0,0, .2);
        font-weight: 600;
        text-align: left;
        min-height: 40px;
        display: flex;
        justify-content: flex-start;
        align-items: center;
        transition: all .4s ease;

        &:focus {
          outline: none;
        }

        @include until($tablet) {
          justify-content: center;
        }

        &:hover {
          background-color: rgba(0,0,0, .4);
        }

        &:first-child {
          border-top: none;

          &.is-active {
            border-top: 1px solid rgba(255,255,255, .5);
          }
        }

        &.is-active {
          background-image: linear-gradient(to right, rgba(255,255,255,1) 0%,rgba(255,255,255,.77) 100%);
          color: mc('grey', '800');
          cursor: default;

          &:hover {
            background-color: transparent;
          }

          svg path {
            fill: mc('grey', '800');
          }
        }

        i {
          margin-right: 10px;
          font-size: 16px;

          @include until($tablet) {
            margin-right: 0;
            font-size: 20px;
          }
        }

        svg {
          margin-right: 10px;
          width: auto;
          height: 20px;
          max-width: 18px;
          max-height: 20px;

          path {
            fill: #FFF;
          }

          @include until($tablet) {
            margin-right: 0;
            font-size: 20px;
          }
        }

        em {
          height: 20px;
        }

        span {
          font-weight: 600;

          @include until($tablet) {
            display: none;
          }
        }
      }

      &-fill {
        flex: 1 1 0;
        background: linear-gradient(to right, rgba(mc('light-blue', '800'), .7), rgba(mc('light-blue', '800'), 1));
      }
    }

    &-frame {
      background-image: radial-gradient(circle at top center, rgba(255,255,255,1) 5%,rgba(255,255,255,.6) 100%);
      border: 1px solid rgba(255,255,255, .5);
      border-radius: 6px;
      width: 400px;
      padding: 1rem;
      color: mc('grey', '700');
      display: block;

      @include until($tablet) {
        width: 100%;
        border-radius: 0;
        border: none;
      }

      h1 {
        font-size: 2rem;
        font-weight: 400;
        color: mc('light-blue', '700');
        text-shadow: 1px 1px 0 #FFF;
        padding: 1rem 0 0 0;
        margin: 0;
      }

      h2 {
        font-size: 1.5rem;
        font-weight: 300;
        color: mc('grey', '700');
        text-shadow: 1px 1px 0 #FFF;
        padding: 0;
        margin: 0 0 25px 0;
      }
    }

    &-tfa {
      position: relative;
      display: flex;
      width: 400px;
      align-items: stretch;
      box-shadow: 0 14px 28px rgba(0,0,0,0.2);
      border-radius: 6px;
      animation: zoomIn .5s ease;
    }

    &-copyright {
      display: flex;
      align-items: center;
      justify-content: center;
      position: absolute;
      left: 0;
      bottom: 10vh;
      width: 100%;
      z-index: 2;
      color: mc('grey', '500');
      font-weight: 400;

      a {
        font-weight: 600;
        color: mc('blue', '500');
        margin-left: .25rem;

        @include until($tablet) {
          color: mc('blue', '200');
        }
      }

      @include until($tablet) {
        color: mc('blue', '50');
      }

    }
  }
</style>