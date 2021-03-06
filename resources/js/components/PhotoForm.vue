<template>
  <div v-show="value" class="p-photo-form">
    <div v-show="loading" class="p-panel">
      <!-- Loader.vueテンプレートを当て込み、<slot>Loading...<slot>に上書き。「送信中...」の文言を表示 -->
      <Loader>送信中...</Loader>
    </div>
      <h2 class="p-title">１.写真を投稿</h2>
      <form v-show="! loading" class="p-form" @submit.prevent="submit"> <!-- デフォルトアクション（ブラウザ本来の挙動）のみ停止 -->
        <div class="u-errors" v-if="errors">
          <ul v-if="errors.photo">
            <li v-for="msg in errors.photo" :key="msg">{{ msg }}</li>
          </ul>
        </div>
          <label class="c-button c-button__report p-form-file-btn">
            <span>ファイルを選択</span>
            <input class="p-form__item" type="file" @change="onFileChange">
          </label>
          <output class="p-form__output" v-if="preview">
            <img :src="preview" alt="">
          </output>
      <h2 class="p-title">２.コメントを入力</h2>
      <textarea class="p-form__item" v-model="commentContent"></textarea>
      <div class="p-form__button">
        <button type="submit" class="c-button c-button--inverse">送信</button>
      </div>
    </form>
  </div>
</template>

<script>
// レスポンスコードの定義をインポート。エラー処理のため
import { CREATED, UNPROCESSABLE_ENTITY } from '../util'
// Loaderコンポーネントをインポート
import Loader from './Loader.vue'

export default {
  components: {
    // Loaderコンポーネントを登録
    Loader
  },
  props: {
    value: {
      type: Boolean,
      required: true
    }
  },
  data () {
    return {
      loading: false, // ローディングを表示させるかどうか
      preview: null,
      photo: null, // 選択中のファイルを格納
      commentContent: '', // 画像の説明
      errors: null // エラーメッセージを格納
    }
  },
  methods: {
    // フォームでファイルが選択されたら実行
    onFileChange (event) {
      // 何も選択されていなかったら処理中断
      if (event.target.files.length === 0) {
        this.reset() // 入力欄の値とプレビュー表示をクリア
        return false
      }

      // ファイルが画像ではなかったら処理中断
      if (! event.target.files[0].type.match('image.*')) {
        this.reset() // 入力欄の値とプレビュー表示をクリア
        return false
      }

      // FileReaderクラスのインスタンスを取得
      const reader = new FileReader()

      // 画像ファイルを読み込み終わったタイミングで実行する処理
      reader.onload = e => {
        // previewに読み込み結果（データURL）を代入
        // previewに値が入ると<output>につけたv-ifがtrueと判定される
        // また<output>内部の<img>のsrc属性は、previewの値を参照しているので
        // 結果として画像が表示される
        this.preview = e.target.result
      }

      // 画像ファイルを読み込む
      // 読み込まれたファイルはデータURL形式で受け取れる（その後の処理は、上記onloadへ）
      reader.readAsDataURL(event.target.files[0])

      this.photo = event.target.files[0] // 画像ファイルを代入
    },
    // 入力欄の値とプレビュー表示をクリアするメソッド
    reset () {
      this.preview = ''
      this.photo = null
      this.commentContent = '' // 画像投稿と同時に投稿した、画像説明コメント
      // this.$elは、コンポーネントそのもののDOM要素
      this.$el.querySelector('input[type="file"]').value = null
      this.errors = '' // バリデーションエラー後、送信成功したら、エラーメッセージを消去
    },
    async submit () {
      // ローディングを表示
      this.loading = true

      const formData = new FormData() // FormData APIを使用。Ajaxでファイルを送るため
      formData.append('photo', this.photo)
      // LaravelでAPIを作成する場合、routes/api.phpにルートを記載するが、
      // その場合、APIにアクセスする側は/api/article/1のようにルートの先頭にapiをつける必要がある
      const response = await axios.post('/api/photos', formData)

      // 通信が終わったら、ローディングを非表示
      this.loading = false

      // 処理中断後に、バリデーションチェック。エラーメッセージを表示させるため
      if (response.status === UNPROCESSABLE_ENTITY) {
        this.errors = response.data.errors
        return false
      }

      // 画像投稿と同時に、画像説明コメント投稿
      const responseComment = await axios.post(`/api/photos/${response.data.id}/comments`, {
        content: this.commentContent
      })

      this.reset()
      // 自動的にフォームが閉じるよう、inputイベントを発行。それに伴い、falseを発行
      this.$emit('input', false)

      if (response.status !== CREATED) {
        this.$store.commit('error/setCode', response.status)
        return false
      }

      // 投稿完了後、メッセージ登録。message.jsモジュールのcontentを更新
      this.$store.commit('message/setContent', {
        content: '送信しました！',
        timeout: 2500
      })

      // 画像投稿後、作成した画像の詳細画面に遷移
      this.$router.push(`/photos/${response.data.id}`)
    }
  }
}
</script>