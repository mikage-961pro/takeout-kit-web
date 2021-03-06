<template>
  <div>
    <v-card
      class="mx-auto"
    >
      <v-row justify="center">
        <v-dialog v-model="isDialog" max-width="500px">
          <v-card>
            <v-card-title>ソート＆フィルタ</v-card-title>
            <v-divider></v-divider>
            <v-card-text>
              <v-container fluid>
                <v-checkbox 
                  v-model="sortSetting.takeout" 
                  label="テイクアウトありのみ表示"
                  @change="searchCheck()"
                ></v-checkbox>
                <v-checkbox 
                  v-model="sortSetting.delivery" 
                  label="宅配サービスありのみ表示"
                  @change="searchCheck()"
                ></v-checkbox>
                <v-checkbox 
                  v-model="sortSetting.thirdDelivery" 
                  label="外部宅配サービスありのみ表示"
                  @change="searchCheck()"
                ></v-checkbox>
              </v-container>
            </v-card-text>
          </v-card>
        </v-dialog>
      </v-row>
      
      <v-toolbar>

        <v-toolbar-title :class="$style.text_color">一覧</v-toolbar-title>

        <v-spacer></v-spacer>

        <v-text-field
          aria-label="タグ検索"
          hide-details
          prepend-icon="fas fa-search"
          single-line
          style="max-width:60%"
          v-model='sortSetting.keyword'
          @keydown.enter="searchText"
        ></v-text-field>

        <v-btn 
          icon
          @click="onDialog"
          aria-label="検索を絞り込む"
        >
          <v-icon>fas fa-filter</v-icon>
        </v-btn>

      </v-toolbar>
        
      <v-card-text>
        <v-container
          id="scroll-target"
          style="height: 75vh"
          class="overflow-y-auto ma-1"
        >
          <div v-for="(shop, i) in shops" :key="i" class="pb-4">
            <nuxt-link :to="`/shops/${shop.id}`" tag="div">
              <v-layout>
                <v-flex xs5>
                  <v-img
                    v-if="!!shop.imageUrl"
                    :src="shop.imageUrl"
                    :lazy-src="require('~/assets/images/loading.gif')"
                    height="125px"
                    contain
                  ></v-img>
                  <v-layout
                    v-else
                    column
                    justify-center
                    align-center
                  >
                    <h3 
                      :class="[$style.text_color, 'pt-10', 'display-1']"
                      style="user-select: none;"
                    > 
                      {{ shop.name }}
                    </h3>
                  </v-layout>
                </v-flex>
                <v-flex xs7>
                  <v-card-title :class="$style.text_color"> {{ shop.name }} </v-card-title>
                  <div class="ml-2">
                    <v-chip
                        v-for="(tag, i) in shop.tag"
                        :key="i"
                        :class="[$style.text_color, 'ma-1']"
                        label
                        @click.native.stop="tagClick"
                    >
                      {{ tag }}
                    </v-chip>
                  </div>
                </v-flex>
              </v-layout>
            </nuxt-link>  
            <v-divider class="mt-5 mb-2"></v-divider>
          </div>
          <client-only>
            <infinite-loading
              ref="infiniteLoading" 
              spinner="spiral"
              @infinite="infiniteLoad"
            >
              <div slot="no-results"/>
              <span :class="$style.text_color" slot="no-more">おしまい🥺</span>
            </infinite-loading>
          </client-only>
        </v-container>
      </v-card-text>
    </v-card>
  </div>
</template>

<script>

export default {
  head() {
    return {
      titleTemplate: 'めしてく'
    }
  },
  data() {
    return {
      location: null,
      sortSetting: {
        sort: 2,
        keyword: '',
        delivery: false,
        thirdDelivery: false,
        takeout: false
      },
      isDialog: false,
      shops: [],
      currentPage: 0,
      hitCount: 0
    }
  },
  methods: {
    async getPosition(options) {
      return new Promise(function(resolve, reject) {
        navigator.geolocation.getCurrentPosition(resolve, reject, options)
      })
    },
    async setLocation() {
      try {
        let position = await this.getPosition()
        position = position.coords
        this.location = {
          latitude: position.latitude,
          longitude: position.longitude
        }
      } catch(e) {
        this.location = null
      }
    },
    async infiniteLoad() {
      if (!!this.$route.query.keyword) {
        this.sortSetting.keyword = this.$route.query.keyword
      }
      this.currentPage ++
      let requestUrl = this.getShopsRequestUrl({
        page: this.currentPage,
        sort: this.sortSetting.sort,
        keyword: this.sortSetting.keyword.replace(' ',','),
        delivery: this.sortSetting.delivery ,
        thirdDelivery: this.sortSetting.thirdDelivery ,
        takeout: this.sortSetting.takeout 
      })
      try {
        let axRes = await this.$axios.get(requestUrl)
        if (!!axRes.data) {
          if (axRes.data.shops.length > 0) {
            this.shops = this.shops.concat(axRes.data.shops)
            this.$refs.infiniteLoading.stateChanger.loaded()
            return 
          } 
        }
      } catch(e) {
        this.currentPage --
      }
      try {
        this.$refs.infiniteLoading.stateChanger.complete()
      } catch {}
    },
    getShopsRequestUrl({
      page = 1,
      sort = 2,
      keyword = '',
      delivery = false,
      thirdDelivery = false,
      takeout = false
    }) {
      return `/shops?page=${page}&sort=${sort}&delivery=${delivery}&thirdDelivery=${thirdDelivery}&takeout=${takeout}&keyword=${keyword}`
    },
    async searchText(event) {
      this.shops = []
      this.currentPage = 0
      this.hitCount = 0
      try {
        const url = this.getShopsRequestUrl({
          sort: this.sortSetting.sort,
          keyword: this.sortSetting.keyword.replace(' ',','),
          delivery: this.sortSetting.delivery ,
          thirdDelivery: this.sortSetting.thirdDelivery ,
          takeout: this.sortSetting.takeout 
        })
        const shops = (await this.$axios.get(url)).data
        this.shops = shops.shops
        this.currentPage = shops.currentPage
        this.hitCount = shops.hitCount
      } catch(e) {
        this.shops = []
        this.currentPage = 0
        this.hitCount = 0
      }
      event.target.blur()
    },
    onDialog() {
      this.isDialog = !this.isDialog
    },
    async searchCheck() {
      this.shops = []
      this.currentPage = 0
      this.hitCount = 0
      try {
        const url = this.getShopsRequestUrl({
          sort: this.sortSetting.sort,
          keyword: this.sortSetting.keyword.replace(' ',','),
          delivery: this.sortSetting.delivery ,
          thirdDelivery: this.sortSetting.thirdDelivery ,
          takeout: this.sortSetting.takeout 
        })
        const shops = (await this.$axios.get(url)).data
        this.shops = shops.shops
        this.currentPage = shops.currentPage
        this.hitCount = shops.hitCount
      } catch(e) {
        this.shops = []
        this.currentPage = 0
        this.hitCount = 0
      }
    },
    async tagClick(event) {
      this.sortSetting.keyword = event.toElement.innerText
      await this.searchCheck()
      return false
    }
  }
}
</script>

<style lang="scss" module>
#link_button_color {
  color: $tomago-black;
  background: $tamago-yellow;
}
.text_color {
  color: $tomago-black;
}
</style>