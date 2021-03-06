<template>
  <section class="section">
    <UiVideoIframeWithItems
      :items="latestData"
      textPositionInMdViewport="right"
      class="section__latest"
      @sendGa="handleSendGa"
    >
      <template #heading>
        <h1>最新影片</h1>
      </template>
    </UiVideoIframeWithItems>
    <div class="section__bottom-wrapper">
      <UiVideoPopular
        :items="popularData"
        class="section__popular"
        @sendGa="handleSendGa"
      />
      <UiVideoSubscriptions class="section__subscriptions" />

      <ContainerGptAd class="section__ad" pageKey="videohub" adKey="FT" />

      <div class="section__categories-wrapper">
        <UiVideoCategory
          v-for="category in categories"
          :key="`video-category-${category.id}`"
          :category="category"
          :items="getCategoryItems(category)"
          class="section__category"
          @sendGa="handleSendGa"
        />
      </div>
      <UiYoutubePolicies class="section__policies" />
    </div>

    <UiStickyAd pageKey="videohub" />
    <ContainerFullScreenAds />
  </section>
</template>

<script>
import { mapState } from 'vuex'
import _ from 'lodash'
import {
  SITE_TITLE,
  SITE_URL,
  VIDEOHUB_CATEGORIES_PLAYLIST_MAPPING as PLAYLIST_MAPPING,
} from '~/constants/index'
import { processResponseItems } from '~/utils/youtube'
import ContainerGptAd from '~/components/ContainerGptAd.vue'
import ContainerFullScreenAds from '~/components/ContainerFullScreenAds.vue'
import UiStickyAd from '~/components/UiStickyAd.vue'
import UiVideoCategory from '~/components/UiVideoCategory.vue'
import UiVideoIframeWithItems from '~/components/UiVideoIframeWithItems.vue'
import UiVideoPopular from '~/components/UiVideoPopular.vue'
import UiVideoSubscriptions from '~/components/UiVideoSubscriptions.vue'
import UiYoutubePolicies from '~/components/UiYoutubePolicies.vue'

const INVERTED_PLAYLIST_MAPPING = _.invert(PLAYLIST_MAPPING)

export default {
  name: 'SectionVideohub',
  components: {
    ContainerGptAd,
    ContainerFullScreenAds,
    UiStickyAd,
    UiVideoCategory,
    UiVideoIframeWithItems,
    UiVideoPopular,
    UiVideoSubscriptions,
    UiYoutubePolicies,
  },
  async fetch() {
    const response = await this.fetchChannelData()
    this.latestData = processResponseItems(response).slice(0, 5)
  },
  data() {
    return {
      latestData: [],
      popularData: [],
      categoriesPlaylistData: {},
    }
  },
  computed: {
    ...mapState({
      section(state) {
        return state.sections.data.items.find(this.isThisSection) ?? {}
      },
    }),
    categories() {
      return this.section.categories ?? []
    },
    playlistIds() {
      return this.categories
        .map(this.mapCategoryToPlatlistId)
        .filter((playlist) => playlist)
    },
  },
  beforeMount() {
    this.fetchPopularData()
    this.fetchCategoriesPlaylistData()
  },
  methods: {
    async fetchCategoriesPlaylistData() {
      const response = await Promise.allSettled(
        this.playlistIds.map(this.fetchYoutubePlaylistItems)
      )
      response.forEach(this.mapDataToCategoriesPlaylist)
    },
    async fetchPopularData() {
      const { items: latest = [] } = await this.fetchChannelData({
        maxResults: 50,
      })
      const ids = latest.map((item) => item.id.videoId).join(',')
      const { items: videos = [] } = await this.$fetchYoutubeVideos({
        part: 'snippet, statistics',
        id: ids,
      })
      const popular = videos
        .sort((a, b) => b.statistics?.viewCount - a.statistics?.viewCount)
        .slice(0, 10)
      this.popularData = processResponseItems({
        items: popular,
      })
    },
    fetchChannelData({ maxResults = 10 } = {}) {
      return this.$fetchYoutubeSearch({
        channelId: 'UCYkldEK001GxR884OZMFnRw',
        maxResults,
        order: 'date',
        part: 'snippet',
        type: 'video',
      })
    },
    fetchYoutubePlaylistItems(playlistId) {
      return this.$fetchYoutubePlaylistItems({
        playlistId,
        part: ['snippet', 'status'],
        maxResults: 15,
      })
    },
    getCategoryItems(category) {
      return this.categoriesPlaylistData[category.name] ?? []
    },
    handleSendGa(param = {}) {
      this.$ga.event(param)
    },
    isThisSection(section) {
      const currentSectionName = this.$route.path.split('/section/')[1]
      return section.name === currentSectionName
    },
    mapCategoryToPlatlistId(category) {
      return PLAYLIST_MAPPING[category.name]
    },
    mapDataToCategoriesPlaylist(data, index) {
      if (data.status === 'fulfilled') {
        const categoryName = INVERTED_PLAYLIST_MAPPING[this.playlistIds[index]]
        this.$set(
          this.categoriesPlaylistData,
          categoryName,
          processResponseItems(data.value)
            ?.filter((item) => item.privacyStatus === 'public')
            .slice(0, 5)
        )
      }
    },
  },
  head() {
    const title = `影音 - ${SITE_TITLE}`
    return {
      title,
      meta: [
        { hid: 'og:title', property: 'og:title', content: title },
        {
          hid: 'og:url',
          property: 'og:url',
          content: `${SITE_URL}${this.$route.path}`,
        },
        { hid: 'twitter:title', name: 'twitter:title', content: title },
        { name: 'section-name', content: 'videohub' },
      ],
    }
  },
}
</script>

<style lang="scss" scoped>
.section {
  overflow: hidden;
  &__latest {
    @include media-breakpoint-up(xl) {
      width: 1024px;
      margin-left: auto;
      margin-right: auto;
      padding: 0 0 53px 0;
    }
  }
  &__bottom-wrapper {
    @include media-breakpoint-up(xl) {
      display: flex;
      flex-wrap: wrap;
    }
  }
  &__popular,
  &__categories-wrapper {
    @include media-breakpoint-up(xl) {
      margin: 30px 0 65px;
    }
  }
  &__popular {
    padding: 30px 0 32px 0;
    @include media-breakpoint-up(xl) {
      order: 1;
      position: relative;
      left: calc((100% - 1024px) / 2);
      width: 239px;
      padding: 0;
    }
  }
  &__subscriptions {
    @include media-breakpoint-up(xl) {
      order: 0;
      width: 100%;
    }
  }
  &__categories-wrapper {
    padding: 0 0 29px;
    @include media-breakpoint-up(xl) {
      order: 2;
      width: calc(1024px - 239px - 39px);
      margin-left: calc((100% - 1024px) / 2 + 39px);
      margin-right: auto;
      padding: 0;
    }
  }
  &__category {
    + .section__category {
      border-top: 1px solid #979797;
      @include media-breakpoint-up(xl) {
        margin-top: 30px;
        border-top: none;
      }
    }
  }

  &__policies {
    margin: 0 auto;
    padding: 5px 20px;
    @include media-breakpoint-up(md) {
      padding: 10px 20px;
    }
    @include media-breakpoint-up(xl) {
      order: 3;
      width: 100%;
    }
  }

  * + .section__ad {
    margin-top: 20px;
  }

  &__ad {
    margin-left: auto;
    margin-right: auto;
    @include media-breakpoint-up(xl) {
      order: 3;
      margin-bottom: 20px;
    }
  }
}
</style>
