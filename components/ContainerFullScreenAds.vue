<template>
  <client-only>
    <div v-if="hasFullScreenAd" class="full-screen-ads">
      <UIFullScreenAd
        v-if="hasAdFirst"
        v-show="isAdFirstVisible"
        :hasDefaultStyle="true"
        :isClosedBtnVisible="isAdFirstClosedBtnVisible"
      >
        <GPTAD
          key="ad-first"
          :adUnit="globalAdUnits.MB_FULL_SCREEN_FIRST.adUnitCode"
          :adSize="globalAdUnits.MB_FULL_SCREEN_FIRST.adSize"
          @slotRequested="setTimerForClosedBtn"
          @slotRenderEnded="handleAdRenderEndedFirst"
        />
      </UIFullScreenAd>
      <UIFullScreenAd
        v-if="hasAdSecondOrThird"
        :hasModifiedStyle="hasModifiedStyle"
      >
        <GPTAD
          v-if="hasAdSecond"
          key="ad-second"
          :adUnit="globalAdUnits.MB_FULL_SCREEN_SECOND.adUnitCode"
          :adSize="globalAdUnits.MB_FULL_SCREEN_SECOND.adSize"
          @slotRenderEnded="disableModifiedStyle"
        />
        <GPTAD
          v-if="hasAdThird"
          key="ad-third"
          :adUnit="globalAdUnits.MB_FULL_SCREEN_THIRD.adUnitCode"
          :adSize="globalAdUnits.MB_FULL_SCREEN_THIRD.adSize"
          @slotRenderEnded="disableModifiedStyle"
        />
      </UIFullScreenAd>
    </div>
  </client-only>
</template>

<script>
import UIFullScreenAd from '~/components/UIFullScreenAd.vue'
import gptUnits from '~/constants/gptUnits'

/*
  蓋板廣告有三層，分別為：第一層 FS、第二層 AD2、第三層 Innity
  
  第一層 FS（AdFirst）：
  帶有自定義的樣式和關閉按鈕，關閉按鈕在發送廣告請求後，延遲 3 秒才顯示
  因為帶有自定義的透明灰底樣式，所以在廣告載入結束後才顯示（isAdFirstVisible）
  若沒有 FS 則顯示下一層 AD2 廣告

  第二層 AD2（AdSecond）:
  AD2 為不額外帶有樣式的廣告，但因為 GPT Lazy loading 需要知道何時載入的位置
  因此需要設置修正用樣式（hasModifiedStyle），來觸發廣告載入
  當廣告載入完成後，就將該樣式移除

  而 AD2 無法透過 GPT 的事件來確認是否有廣告，跟廠商協調後
  當 AD2 沒廣告時，由廠商傳遞 noad2 事件，由我們監聽來顯示下一層 Innity 廣告

  第三層 Innity（AdThird）:
  同 AD2 一樣為不額外帶有樣式的廣告，因此需要先設置修正用樣式

  ＊ APP 用頁面沒有蓋板廣告
*/

export default {
  name: 'ContainerFullScreenAds',
  components: {
    UIFullScreenAd,
  },
  data() {
    return {
      hasAdFirst: true,
      hasAdSecond: false,
      hasAdThird: false,
      hasModifiedStyle: true,
      isAdFirstVisible: false,
      isAdFirstClosedBtnVisible: false,
      globalAdUnits: gptUnits.global,
      timerClosedBtn: null,
    }
  },
  computed: {
    hasFullScreenAd() {
      const isNotApp = this.$route.query.layout !== 'app'
      return (
        isNotApp && (this.hasAdFirst || this.hasAdSecond || this.hasAdThird)
      )
    },
    hasAdSecondOrThird() {
      return this.hasAdSecond || this.hasAdThird
    },
  },
  mounted() {
    // AD2 無法透過 GPT 的事件來確認是否有廣告，需要透過此事件來確認
    // 因為 this 綁定對象關係，使用 =>
    window.addEventListener('noad2', () => {
      this.hasModifiedStyle = true
      this.hasAdSecond = false
      this.hasAdThird = true
    })
  },
  methods: {
    setTimerForClosedBtn() {
      this.timerClosedBtn = setTimeout(() => {
        this.isAdFirstClosedBtnVisible = true
      }, 3000)
    },
    handleAdRenderEndedFirst(event) {
      if (event.isEmpty) {
        clearTimeout(this.timerClosedBtn)
      }
      this.isAdFirstVisible = true
      this.hasAdFirst = !event.isEmpty
      this.hasAdSecond = event.isEmpty
    },
    disableModifiedStyle() {
      this.hasModifiedStyle = false
    },
  },
}
</script>