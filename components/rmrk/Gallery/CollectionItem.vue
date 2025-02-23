<template>
  <section>
    <div class="columns is-centered">
      <div class="column is-half has-text-centered">
        <div class="container image is-128x128 mb-2">
          <BasicImage
            :src="image"
            :alt="name"
            rounded
            customClass="collection__image" />
        </div>
        <h1 class="title is-2">
          <template>
            {{ name }}
          </template>
        </h1>
      </div>
    </div>

    <div class="columns is-align-items-center">
      <div class="column">
        <div>
          <div class="label">
            {{ $t('creator') }}
          </div>
          <div v-if="issuer" class="subtitle is-size-6">
            <ProfileLink :address="issuer" inline showTwitter showDiscord />
            <p v-if="showMintTime" class="is-flex is-align-items-center py-3">
              <b-icon icon="clock" size="is-medium" />
              <span class="ml-2">Started minting {{ formattedTimeToNow }}</span>
            </p>
          </div>
        </div>
      </div>

      <div class="column is-6-tablet is-7-desktop is-8-widescreen">
        <CollectionActivity :id="id" />
      </div>

      <div class="column has-text-right">
        <Sharing
          v-if="sharingVisible"
          class="mb-2"
          :label="name"
          :iframe="iframeSettings">
          <DonationButton :address="issuer" />
        </Sharing>
      </div>
    </div>

    <div class="columns is-centered">
      <div class="column is-8 has-text-centered">
        <DescriptionWrapper
          v-if="!isLoading"
          :rowHeight="50"
          :text="description.replaceAll('\n', '  \n')" />
      </div>
    </div>

    <b-tabs
      position="is-centered"
      v-model="activeTab"
      ref="tabsContainer"
      :style="{ minHeight: '800px' }"
      class="tabs-container-mobile">
      <b-tab-item label="Collection" value="collection">
        <Search v-bind.sync="searchQuery" :disableToggle="!totalListed">
          <Layout class="mr-5" />
          <b-field>
            <Pagination
              hasMagicBtn
              simple
              replace
              preserveScroll
              :total="total"
              v-model="currentValue"
              v-if="activeTab === 'collection'"
              :per-page="first" />
          </b-field>
        </Search>

        <GalleryCardList
          :items="collection.nfts"
          :listed="!!(searchQuery && searchQuery.listed)"
          horizontalLayout />

        <Pagination
          class="py-5"
          replace
          v-if="activeTab === 'collection'"
          preserveScroll
          :total="total"
          v-model="currentValue"
          :per-page="first" />
      </b-tab-item>
      <b-tab-item label="Activity" value="activity">
        <CollectionPriceChart :priceData="priceData" />
      </b-tab-item>
      <b-tab-item label="History" value="history">
        <History
          v-if="!isLoading && activeTab === 'history'"
          :events="eventsOfNftCollection"
          :openOnDefault="isHistoryOpen"
          hideCollapse
          @setPriceChartData="setPriceChartData" />
      </b-tab-item>
    </b-tabs>
  </section>
</template>

<script lang="ts">
import { emptyObject } from '@/utils/empty'
import { Component, mixins, Watch, Ref } from 'nuxt-property-decorator'
import { CollectionWithMeta, Interaction } from '../service/scheme'
import {
  sanitizeIpfsUrl,
  fetchCollectionMetadata,
  onlyPriceEvents,
} from '../utils'
import isShareMode from '@/utils/isShareMode'
import shouldUpdate from '@/utils/shouldUpdate'
import collectionById from '@/queries/collectionById.graphql'
import collectionChartById from '@/queries/rmrk/subsquid/collectionChartById.graphql'
import { CollectionMetadata } from '../types'
import { NFT } from '@/components/rmrk/service/scheme'
import { exist } from '@/components/rmrk/Gallery/Search/exist'
import { SearchQuery } from './Search/types'
import ChainMixin from '@/utils/mixins/chainMixin'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import { getCloudflareImageLinks } from '~/utils/cachingStrategy'
import { mapOnlyMetadata } from '~/utils/mappers'
import CreatedAtMixin from '@/utils/mixins/createdAtMixin'
import { CollectionChartData as ChartData } from '@/utils/chart'
import { mapDecimals } from '@/utils/mappers'
import { notificationTypes, showNotification } from '@/utils/notification'
import allCollectionSaleEvents from '@/queries/rmrk/subsquid/allCollectionSaleEvents.graphql'
import { sortedEventByDate } from '~/utils/sorting'

const components = {
  GalleryCardList: () =>
    import('@/components/rmrk/Gallery/GalleryCardList.vue'),
  CollectionActivity: () =>
    import('@/components/rmrk/Gallery/CollectionActivity.vue'),
  Sharing: () => import('@/components/rmrk/Gallery/Item/Sharing.vue'),
  ProfileLink: () => import('@/components/rmrk/Profile/ProfileLink.vue'),
  Search: () => import('./Search/SearchBarCollection.vue'),
  Pagination: () => import('@/components/rmrk/Gallery/Pagination.vue'),
  DonationButton: () => import('@/components/transfer/DonationButton.vue'),
  Layout: () => import('@/components/rmrk/Gallery/Layout.vue'),
  CollectionPriceChart: () =>
    import('@/components/rmrk/Gallery/CollectionPriceChart.vue'),
  BasicImage: () => import('@/components/shared/view/BasicImage.vue'),
  DescriptionWrapper: () =>
    import('@/components/shared/collapse/DescriptionWrapper.vue'),
  History: () => import('@/components/rmrk/Gallery/History.vue'),
}
@Component<CollectionItem>({
  components,
})
export default class CollectionItem extends mixins(
  ChainMixin,
  PrefixMixin,
  CreatedAtMixin
) {
  @Ref('tabsContainer') readonly tabsContainer
  private id = ''
  private collection: CollectionWithMeta = emptyObject<CollectionWithMeta>()
  public meta: CollectionMetadata = emptyObject<CollectionMetadata>()
  private searchQuery: SearchQuery = {
    search: '',
    type: '',
    sortBy: 'BLOCK_NUMBER_DESC',
    listed: false,
  }
  public activeTab = 'collection'
  private currentValue = parseInt(this.$route.query?.page as string) || 1
  private first = 16
  protected total = 0
  protected totalListed = 0
  protected stats: NFT[] = []
  protected priceData: [ChartData[], ChartData[]] | [] = []
  private statsLoaded = false
  private queryLoading = 0
  public eventsOfNftCollection: Interaction[] | [] = []
  public selectedEvent = 'all'
  public priceChartData: [Date, number][][] = []
  private openHistory = true

  get hasChartData(): boolean {
    return this.priceData.length > 0
  }

  get isLoading(): boolean {
    return Boolean(this.queryLoading)
  }

  get offset(): number {
    return this.currentValue * this.first - this.first
  }

  get image(): string | undefined {
    return this.meta.image
  }

  get description(): string {
    return this.meta.description || ''
  }

  get name(): string {
    return this.collection.name || this.id
  }

  get isHistoryOpen(): boolean {
    return this.openHistory
  }

  get nfts(): NFT[] {
    return this.collection.nfts || []
  }

  get issuer(): string {
    return this.collection.issuer || ''
  }

  get sharingVisible(): boolean {
    return !isShareMode
  }

  get compactCollection(): boolean {
    return this.$store.getters['preferences/getCompactCollection']
  }

  get showMintTime(): boolean {
    return this.$store.state.preferences.showMintTimeCollection
  }

  private buildSearchParam(checkForEmpty?): Record<string, unknown>[] {
    const params: any[] = []

    if (this.searchQuery.search) {
      params.push({
        name: { likeInsensitive: `%${this.searchQuery.search}%` },
      })
    }

    if (this.searchQuery.listed || checkForEmpty) {
      params.push({
        price: { greaterThan: '0' },
      })
    }

    return params
  }

  public created(): void {
    this.checkId()
    this.checkActiveTab()
    this.checkIfEmptyListed()
    this.$apollo.addSmartQuery('collection', {
      query: collectionById,
      client: this.urlPrefix,
      loadingKey: 'queryLoading',
      manual: true,
      variables: () => {
        return {
          id: this.id,
          orderBy: this.searchQuery.sortBy,
          search: this.buildSearchParam(),
          first: this.first,
          offset: this.offset,
        }
      },
      result: this.handleResult,
    })
  }

  public async checkIfEmptyListed(): Promise<void> {
    // if the collection is empty, we need to check if there are any listed NFTs
    this.$apollo.addSmartQuery('totalListed', {
      query: collectionById,
      client: this.urlPrefix,
      update: (data) => {
        return data.collectionEntity.nfts.totalCount
      },
      variables: () => {
        return {
          id: this.id,
          orderBy: this.searchQuery.sortBy,
          search: this.buildSearchParam(true),
          first: this.first,
          offset: this.offset,
        }
      },
    })
  }
  public setPriceChartData(data: [Date, number][][]) {
    this.priceChartData = data
  }

  protected async loadStats(): Promise<void> {
    const { data } = await this.$apollo.query<{
      buys: ChartData[]
      listings: ChartData[]
    }>({
      query: collectionChartById,
      client: 'subsquid',
      variables: {
        id: this.id,
      },
    })

    if (!data) {
      return
    }
    // console.log(data.collection.nfts.nodes)

    // const events: Interaction[][] =
    //   data.collection.nfts.nodes.map((nft) => nft.events) || []

    this.loadPriceData(data)
    this.checkTabLocate()
  }

  // Get collection query with NFT Events on it
  protected async fetchHistoryEvents() {
    try {
      const { data } = await this.$apollo.query<{ events: Interaction[] }>({
        query: allCollectionSaleEvents,
        client: 'subsquid',
        variables: {
          id: this.id,
          and: {
            // interaction_eq: 'BUY',
          },
        },
      })
      if (data && data.events && data.events.length) {
        let events: Interaction[] = data.events
        // TODO : default value of HISTORY for BUY
        // Check if lot of BUY Events, default selectedEvent of History.vue to "BUY"
        this.eventsOfNftCollection = [...sortedEventByDate(events, 'DESC')]
        this.checkTabLocate()
      }
    } catch (e) {
      showNotification(`${e}`, notificationTypes.warn)
    }
  }

  public loadPriceData({
    buys,
    listings,
  }: {
    buys: ChartData[]
    listings: ChartData[]
  }): void {
    this.priceData = []

    const mapToDecimals = mapDecimals(this.decimals, false)
    const soldPriceData = buys.map((buy) => ({
      ...buy,
      value: mapToDecimals(buy.value),
      average: mapToDecimals(buy.average || 0),
    }))

    const listedPriceData = listings.map((listing) => ({
      ...listing,
      value: mapToDecimals(listing.value),
    }))

    this.priceData = [listedPriceData, soldPriceData]
  }

  public async handleResult({ data }: any): Promise<void> {
    const { collectionEntity } = data
    if (!collectionEntity) {
      this.$router.push({ name: 'errorcollection' })
      return
    }
    this.firstMintDate = collectionEntity.createdAt
    await getCloudflareImageLinks(
      collectionEntity.nfts.nodes.map(mapOnlyMetadata)
    ).catch(console.warn)
    this.collection = {
      ...collectionEntity,
      nfts: collectionEntity.nfts.nodes,
    }
    this.total = collectionEntity.nfts.totalCount

    await this.fetchMetadata()
  }

  public async fetchMetadata(): Promise<void> {
    if (this.collection['metadata'] && !this.meta['image']) {
      const meta = await fetchCollectionMetadata(this.collection)
      this.meta = {
        ...meta,
        image: sanitizeIpfsUrl(meta.image || ''),
      }
      this.$store.dispatch('history/setCurrentlyViewedCollection', {
        name: this.name,
        image: this.image,
        description: this.description,
        numberOfItems: this.total || 0,
      })
    }
  }

  public checkId(): void {
    if (this.$route.params.id) {
      this.id = this.$route.params.id
    }
  }

  public checkActiveTab(): void {
    exist(this.$route.query.tab, (val) => {
      this.activeTab = val
    })
  }

  checkTabLocate() {
    exist(this.$route.query.locate, (val) => {
      if (val !== 'true') {
        return
      }
      this.tabsContainer.$el.scrollIntoView()
      this.$router.replace({
        query: { ...this.$route.query, ['locate']: 'false' },
      })
    })
  }

  @Watch('activeTab')
  protected onTabChange(val: string, oldVal: string): void {
    let queryTab = this.$route.query.tab

    if (shouldUpdate(val, oldVal) && queryTab !== val) {
      this.$router.replace({
        path: String(this.$route.path),
        query: { tab: val },
      })
    }

    // Load chart data once when clicked on activity tab for the first time.
    if (val === 'activity') {
      this.loadStats()
    } else if (val === 'history') {
      this.fetchHistoryEvents()
    }
  }

  get iframeSettings(): Record<string, unknown> {
    return { width: '100%', height: '100vh' }
  }

  protected priceEvents(nftEvents: Interaction[]): Interaction[] {
    return nftEvents.filter(onlyPriceEvents)
  }
}
</script>

<style lang="scss">
.collection__image img {
  border-radius: 0px;
  color: transparent;
}
</style>
