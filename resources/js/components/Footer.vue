<template>
  <div id="footer">
    <footer id="footer__inner">
      <button v-if="isLogin" class="c-button c-button--link" @click="logout">
        ログアウト
      </button>
      <RouterLink v-else class="c-button c-button--link c-button__LoginRegister--color c-button__LoginRegister--footerPosition" to="/login">
        ログイン / 登録
      </RouterLink>
    </footer>
  </div>
</template>

<script>
import { mapState, mapGetters } from 'vuex'// VuexのmapState関数、mapGetters関数
export default {
  computed: {
    ...mapState({ // VuexのmapState関数。当算出プロパティとstore/auth.jsのステートを、ひも付け
      apiStatus: state => state.auth.apiStatus
    }),
    ...mapGetters({
    // 冗長な記述を避けるため、直接テンプレートで$storeを参照するのではなく、computedからストアのゲッターを参照
      isLogin: 'auth/check'
    })
  },
  methods: {
    async logout () {
      // authストアのlogoutアクションを呼び出す

      // await で非同期なアクションの処理の完了（Promiseの解決）を待つ
      // this.$store でストアを参照。stores/index.js の Vue.use(Vuex) によって使える
      // dispatchメソッドでアクションを呼び出す。auth/logoutでアクションを指定。store/auth.jsにて、namespaced: trueにしてあるため
      // 第二引数は、ログアウトのため無し
      await this.$store.dispatch('auth/logout')
      if (this.apiStatus) {
        // トップページに移動。this にルーターオブジェクトを表す $router が追加されている。router.jsにて、 Vue.use(VueRouter) と記述したため
        this.$router.push('/login')
      }
    }
  }
}
</script>