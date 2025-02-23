<template>
  <div>
    <Search v-bind.sync="searchQuery" v-if="showSearchBar">
      <Layout class="mr-5" />
      <b-field>
        <Pagination
          hasMagicBtn
          simple
          replace
          :total="total"
          v-model="currentValue" />
      </b-field>
    </Search>
    <GalleryCardList
      :items="items"
      horizontalLayout
      :route="route"
      :link="link"
      :listed="searchQuery.listed" />
    <Pagination
      class="pt-5 pb-5"
      replace
      :total="total"
      v-model="currentValue" />
  </div>
</template>

<script lang="ts">
import { Component, mixins, Prop } from 'nuxt-property-decorator'
import { DocumentNode } from 'graphql'
import { NFTWithMeta } from '../service/scheme'
import { SearchQuery } from '@/components/rmrk/Gallery/Search/types'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import { getCloudflareImageLinks } from '~/utils/cachingStrategy'
import { mapOnlyMetadata } from '~/utils/mappers'

const components = {
  GalleryCardList: () => import('./GalleryCardList.vue'),
  Pagination: () => import('@/components/rmrk/Gallery/Pagination.vue'),
  Search: () =>
    import('@/components/rmrk/Gallery/Search/SearchBarCollection.vue'),
  Layout: () => import('@/components/rmrk/Gallery/Layout.vue'),
}

@Component({ components })
export default class PaginatedCardList extends mixins(PrefixMixin) {
  @Prop({ default: '/rmrk/detail' }) public route!: string
  @Prop({ default: 'rmrk/detail' }) public link!: string
  @Prop() public query!: DocumentNode
  @Prop(String) public account!: string
  @Prop(Boolean) public showSearchBar!: boolean

  private searchQuery: SearchQuery = {
    search: '',
    type: '',
    sortBy: 'BLOCK_NUMBER_DESC',
    listed: false,
  }

  private currentValue = 1
  private first = 20
  private total = 0
  protected items: NFTWithMeta[] = []

  private buildSearchParam(): Record<string, unknown>[] {
    const params: any[] = []

    if (this.searchQuery.search) {
      params.push({
        name: { likeInsensitive: `%${this.searchQuery.search}%` },
      })
    }

    if (this.searchQuery.listed) {
      params.push({
        price: { greaterThan: '0' },
      })
    }

    return params
  }

  get offset(): number {
    return this.currentValue * this.first - this.first
  }

  created() {
    this.$apollo.addSmartQuery('items', {
      query: this.query,
      manual: true,
      client: this.urlPrefix,
      update: ({ nFTEntities }) => nFTEntities.nodes,
      loadingKey: 'isLoading',
      result: this.handleResult,
      variables: () => {
        return {
          account: this.account,
          orderBy: this.searchQuery.sortBy,
          search: this.buildSearchParam(),
          first: this.first,
          offset: this.offset,
        }
      },
      fetchPolicy: 'cache-and-network',
    })
  }

  protected async handleResult({ data }: any) {
    if (data) {
      const { nodes, totalCount } = data.nFTEntities
      await getCloudflareImageLinks(nodes.map(mapOnlyMetadata)).catch(
        console.warn
      )
      this.total = totalCount
      this.items = nodes
      this.$emit('change', this.total)
    }
  }
}
</script>
