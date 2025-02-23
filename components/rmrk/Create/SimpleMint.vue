<template>
  <div class="columns mb-6">
    <div class="column is-6 is-offset-3">
      <section>
        <br />
        <Loader v-model="isLoading" :status="status" />
        <div class="box">
          <p class="title is-size-3">
            <!-- {{ $t('mint.context') }} -->
            Create NFT Collectibles
          </p>
          <p class="subtitle is-size-7">{{ $t('using') }} {{ version }}</p>
          <b-field>
            <div>
              {{ $t('computed id') }}: <b>{{ rmrkId }}</b>
            </div>
          </b-field>
          <b-field>
            <Auth />
          </b-field>

          <MetadataUpload
            v-model="file"
            label="Drop your NFT here or click to upload or simply paste image from clipboard. We support various media types (BMP, GIF, JPEG, PNG, SVG, TIFF, WEBP, MP4, OGV, QUICKTIME, WEBM, GLB, FLAC, MP3, JSON)"
            expanded
            preview />

          <BasicInput
            v-model="rmrkMint.name"
            :label="$t('mint.nft.name.label')"
            :message="$t('mint.nft.name.message')"
            :placeholder="$t('mint.nft.name.placeholder')"
            @blur.native.capture="generateSymbol"
            expanded
            spellcheck="true" />

          <BasicInput
            v-model="rmrkMint.symbol"
            :label="$t('mint.collection.symbol.label')"
            :message="$t('mint.collection.symbol.message')"
            :placeholder="$t('mint.collection.symbol.placeholder')"
            @keydown.native.space.prevent
            maxlength="10"
            expanded />

          <BasicInput
            v-model="meta.description"
            maxlength="500"
            type="textarea"
            spellcheck="true"
            class="mb-0 mt-5"
            :label="$t('mint.nft.description.label')"
            :message="$t('mint.nft.description.message')"
            :placeholder="$t('mint.nft.description.placeholder')" />

          <b-field :label="$t('Edition')" class="mt-5">
            <b-numberinput
              v-model="rmrkMint.max"
              placeholder="1 is minumum"
              expanded
              :min="1"></b-numberinput>
          </b-field>

          <MetadataUpload
            v-if="secondaryFileVisible"
            label="Your NFT requires a poster/cover to be seen in gallery. Please upload image (jpg/ png/ gif)"
            v-model="secondFile"
            icon="file-image"
            accept="image/png, image/jpeg, image/gif"
            expanded
            preview />
          <AttributeTagInput
            v-model="rmrkMint.tags"
            placeholder="Get discovered easier through tags" />

          <BalanceInput
            :step="0.1"
            @input="updateMeta"
            label="Price"
            expanded />
          <div class="content mt-3">
            <p>
              Hint: Setting the price now requires making an additional
              transaction.
            </p>
          </div>

          <b-field>
            <PasswordInput v-model="password" :account="accountId" />
          </b-field>
          <b-field>
            <CollapseWrapper
              v-if="rmrkMint.max > 1"
              visible="mint.expert.show"
              hidden="mint.expert.hide">
              <p class="title is-6">
                {{ $t('mint.expert.count', [parseAddresses.length]) }}
              </p>
              <p class="sub-title is-6 has-text-warning" v-show="syncVisible">
                {{ $t('mint.expert.countGlitch', [parseAddresses.length]) }}
              </p>
              <b-field :label="$t('mint.expert.batchSend')">
                <b-input
                  v-model="batchAdresses"
                  type="textarea"
                  :placeholder="'Distribute NFTs to multiple addresses like this:\n- HjshJ....3aJk\n- FswhJ....3aVC\n- HjW3J....9c3V'"
                  spellcheck="true"></b-input>
              </b-field>
              <BasicSlider
                v-model="distribution"
                label="action.distributionCount" />
              <b-field v-show="syncVisible">
                <b-button
                  outlined
                  @click="syncEdition"
                  icon-left="sync"
                  type="is-warning"
                  >{{ $t('mint.expert.sync', [actualDistribution]) }}</b-button
                >
              </b-field>
              <BasicSwitch v-model="random" label="action.random" />
              <BasicSwitch v-model="postfix" label="mint.expert.postfix" />
            </CollapseWrapper>
          </b-field>
          <BasicSwitch v-model="nsfw" label="mint.nfsw" />
          <b-field>
            <b-switch v-model="hasToS" :rounded="false">
              {{ $t('termOfService.accept') }}
            </b-switch>
          </b-field>
          <b-field>
            <b-button
              type="is-primary"
              icon-left="paper-plane"
              @click="sub"
              :disabled="disabled"
              :loading="isLoading"
              outlined>
              {{ $t('mint.submit') }}
            </b-button>
          </b-field>
          <b-field>
            <template>
              <b-icon icon="calculator" />
              <span class="pr-2">{{ $t('mint.estimated') }}</span>
              <Money :value="estimated" inline />
              <span class="pl-2"> ({{ getUsdFromKsm().toFixed(2) }} USD) </span>
            </template>
          </b-field>
        </div>
      </section>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, mixins, Watch } from 'nuxt-property-decorator'
import { MediaType } from '../types'
import { emptyObject } from '@/utils/empty'
import Support from '@/components/shared/Support.vue'
import Connector from '@kodadot1/sub-api'
import exec, {
  execResultValue,
  txCb,
  estimate,
} from '@/utils/transactionExecutor'
import { notificationTypes, showNotification } from '@/utils/notification'
import SubscribeMixin from '@/utils/mixins/subscribeMixin'
import RmrkVersionMixin from '@/utils/mixins/rmrkVersionMixin'
import {
  Attribute,
  SimpleNFT,
  NFTMetadata,
  NFT,
  getNftId,
  Collection,
} from '../service/scheme'
import { unSanitizeIpfsUrl } from '@/utils/ipfs'
import { formatBalance } from '@polkadot/util'
import { generateId } from '@/components/rmrk/service/Consolidator'
import { canSupport, feeTx } from '@/utils/support'
import { resolveMedia } from '../utils'
import NFTUtils, { MintType } from '../service/NftUtils'
import { DispatchError } from '@polkadot/types/interfaces'
import TransactionMixin from '@/utils/mixins/txMixin'
import { encodeAddress, isAddress } from '@polkadot/util-crypto'
import ChainMixin from '@/utils/mixins/chainMixin'
import correctFormat from '@/utils/ss58Format'
import {
  isFileWithoutType,
  isSecondFileVisible,
  nsfwAttribute,
  offsetAttribute,
  secondaryFileVisible,
} from '@/components/rmrk/Create/mintUtils'
import {
  sendFunction,
  shuffleFunction,
  toDistribute,
} from '@/components/accounts/utils'
import { PinningKey, pinFileToIPFS, pinJson } from '@/utils/pinning'
import { uploadDirect } from '@/utils/directUpload'
import { IPFS_KODADOT_IMAGE_PLACEHOLDER } from '~/utils/constants'
import { createMetadata, findUniqueSymbol } from '@kodadot1/minimark'
import Vue from 'vue'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import collectionList from '@/queries/collectionListByAccount.graphql'

const components = {
  Auth: () => import('@/components/shared/Auth.vue'),
  MetadataUpload: () => import('./DropUpload.vue'),
  PasswordInput: () => import('@/components/shared/PasswordInput.vue'),
  Tooltip: () => import('@/components/shared/Tooltip.vue'),
  Support,
  AttributeTagInput: () => import('./AttributeTagInput.vue'),
  BalanceInput: () => import('@/components/shared/BalanceInput.vue'),
  Money: () => import('@/components/shared/format/Money.vue'),
  Loader: () => import('@/components/shared/Loader.vue'),
  CollapseWrapper: () =>
    import('@/components/shared/collapse/CollapseWrapper.vue'),
  BasicSwitch: () => import('@/components/shared/form/BasicSwitch.vue'),
  SendHandler: () => import('@/components/rmrk/Create/Admin/SendHandler.vue'),
  BasicSlider: () => import('@/components/shared/form/BasicSlider.vue'),
  BasicInput: () => import('@/components/shared/form/BasicInput.vue'),
}

@Component<SimpleMint>({
  components,
})
export default class SimpleMint extends mixins(
  SubscribeMixin,
  RmrkVersionMixin,
  TransactionMixin,
  ChainMixin,
  PrefixMixin
) {
  private rmrkMint: SimpleNFT = {
    ...emptyObject<SimpleNFT>(),
    max: 1,
  }
  private meta: NFTMetadata = emptyObject<NFTMetadata>()
  // private accountId: string = '';
  private uploadMode = true
  private file: File | null = null
  private secondFile: File | null = null
  private password = ''
  private hasToS = false
  private nsfw = false
  private price = 0
  private estimated = ''
  protected batchAdresses = ''
  protected postfix = true
  protected random = false
  protected distribution = 100
  protected first = 100
  protected collections: Collection[] = []

  // query for nfts information by accountId
  public async created() {
    this.$apollo.addSmartQuery('collections', {
      query: collectionList,
      client: this.urlPrefix,
      manual: true,
      loadingKey: 'isLoading',
      result: this.handleResult,
      variables: () => {
        return {
          account: this.accountId,
          first: this.first,
        }
      },
      fetchPolicy: 'cache-and-network',
    })
  }

  // handle query results
  public handleResult(data: any) {
    const collectionEntities = data?.data?.collectionEntities
    if (collectionEntities) {
      this.collections = collectionEntities.nodes
    }
  }

  // set symbol name
  private generateSymbol(): void {
    if (!this.rmrkMint?.symbol && this.rmrkMint.name?.length) {
      const symbol = this.generateSymbolCore(
        this.rmrkMint.name,
        this.collections
      )
      Vue.set(this.rmrkMint, 'symbol', symbol)
    }
  }

  // core: to generate symbol
  private generateSymbolCore(name: string, collections: Collection[]): string {
    let symbol = name.replaceAll(' ', '_')
    const usedSymbols = collections.map((collection) => {
      // collection.id is like xxxx-symbolName
      const splitArray = collection.id.split('-')
      return splitArray.length > 1 ? splitArray[1] : ''
    })
    symbol = findUniqueSymbol(symbol, usedSymbols)
    return symbol.slice(0, 10) // symbol's length have to smaller than 10
  }

  protected updateMeta(value: number): void {
    this.price = value

    if (this.canCalculateTransactionFees) {
      this.estimateTx()
    }
  }

  get fileType(): MediaType {
    return resolveMedia(this.file?.type)
  }

  get secondaryFileVisible(): boolean {
    const fileType = this.fileType
    return (
      isFileWithoutType(this.file, fileType) || isSecondFileVisible(fileType)
    )
  }

  get accountId(): string {
    return this.$store.getters.getAuthAddress
  }

  get rmrkId(): string {
    return generateId(this.accountId, this.rmrkMint?.symbol || '')
  }

  get canCalculateTransactionFees(): boolean {
    const { name, symbol, max } = this.rmrkMint
    return !!(this.price && name && symbol && max)
  }

  get disabled(): boolean {
    const { name, symbol, max } = this.rmrkMint
    return !(
      name &&
      symbol &&
      max &&
      this.hasToS &&
      this.accountId &&
      this.file &&
      !this.syncVisible
    )
  }

  get hasSupport(): boolean {
    return this.$store.state.preferences.hasSupport
  }

  get hasCarbonOffset(): boolean {
    return this.$store.state.preferences.hasCarbonOffset
  }

  get arweaveUpload(): boolean {
    return this.$store.state.preferences.arweaveUpload
  }

  protected async estimateTx() {
    const { accountId, version } = this
    const { api } = Connector.getInstance()

    const result = NFTUtils.generateRemarks(this.rmrkMint, accountId, version)
    const cb = api.tx.utility.batchAll
    const remarks: string[] = Array.isArray(result)
      ? result
      : [
          NFTUtils.toString(result.collection, version),
          ...result.nfts.map((nft) => NFTUtils.toString(nft, version)),
        ]

    const args = !this.hasSupport
      ? remarks.map(this.toRemark)
      : [
          ...remarks.map(this.toRemark),
          ...(await canSupport(this.hasSupport, 3)),
        ]

    this.estimated = await estimate(this.accountId, cb, [args])
  }

  get enoughTokens(): boolean {
    return this.parseAddresses.length <= this.rmrkMint.max
  }

  get ss58Format(): number {
    return this.chainProperties?.ss58Format
  }

  get parseAddresses(): string[] {
    const { batchAdresses } = this
    const addresses = batchAdresses
      .split('\n')
      .map((x) => x.split('-'))
      .filter((x) => x.length)
      .map((x) => x[1])
      .filter((x) => x)
      .map((a) => a.trim())
    const onlyValid = addresses
      .filter((a) => isAddress(a))
      .map((a) => encodeAddress(a, correctFormat(this.ss58Format)))

    return onlyValid
  }

  get syncVisible(): boolean {
    return this.rmrkMint.max < this.actualDistribution
  }

  get actualDistribution(): number {
    return toDistribute(this.parseAddresses.length, this.distribution)
  }

  protected syncEdition(): void {
    this.rmrkMint.max = this.actualDistribution
  }

  protected async sub(): Promise<void> {
    this.isLoading = true
    this.status = 'loader.ipfs'
    const { accountId, version } = this
    const { api } = Connector.getInstance()

    try {
      const meta = await this.constructMeta()
      this.rmrkMint.metadata = meta || ''

      const result = NFTUtils.generateRemarks(
        this.rmrkMint,
        accountId,
        version,
        false,
        this.postfix && this.rmrkMint.max > 1
          ? (name: string, index: number) => `${name} #${index + 1}`
          : undefined
      ) as MintType

      const cb = api.tx.utility.batchAll
      const remarks: string[] = Array.isArray(result)
        ? result
        : [
            NFTUtils.toString(result.collection, version),
            ...result.nfts.map((nft) => NFTUtils.toString(nft, version)),
          ]

      const args = !this.hasSupport
        ? remarks.map(this.toRemark)
        : [
            ...remarks.map(this.toRemark),
            ...(await canSupport(this.hasSupport, 3)),
          ]

      const tx = await exec(
        this.accountId,
        '',
        cb,
        [args],
        txCb(
          async (blockHash) => {
            execResultValue(tx)
            const header = await api.rpc.chain.getHeader(blockHash)
            const blockNumber = header.number.toString()

            if (this.batchAdresses) {
              this.sendBatch(result.nfts, blockNumber)
            } else if (this.price) {
              this.listForSale(result.nfts, blockNumber)
            } else {
              this.navigateToDetail(result.nfts[0], blockNumber)
            }

            showNotification(
              `[NFT] Saved ${this.rmrkMint.max} entries in block ${blockNumber}`,
              notificationTypes.success
            )

            if (!this.batchAdresses || !this.price) {
              this.isLoading = false
            }
          },
          (dispatchError) => {
            execResultValue(tx)
            this.onTxError(dispatchError)
            this.isLoading = false
          },
          (res) => this.resolveStatus(res.status)
        )
      )
    } catch (e) {
      if (e instanceof Error) {
        showNotification(e.toString(), notificationTypes.danger)
        this.isLoading = false
      }
    }
  }

  public async fetchRandomSeed(): Promise<number[]> {
    const { api } = Connector.getInstance()
    const random = await api.query.babe.randomness()
    return Array.from(random)
  }

  protected async sendBatch(
    remarks: NFT[],
    originalBlockNumber: string
  ): Promise<void> {
    try {
      const { version, price } = this
      const addresses = this.parseAddresses
      showNotification(`[APP] Sending NFTs to ${addresses.length} adresses`)

      const onlyNfts = remarks
        .filter(NFTUtils.isNFT)
        .map((nft) => ({ ...nft, id: getNftId(nft, originalBlockNumber) }))
      // .map(nft =>
      //   NFTUtils.createInteraction('SEND', version, nft.id, String(price))
      // )

      if (!onlyNfts.length) {
        showNotification('Can not send empty NFTs', notificationTypes.danger)
        return
      }

      const { api } = Connector.getInstance()
      const outOfTheNamesForTheRemarks = sendFunction(
        addresses,
        this.distribution,
        this.random ? shuffleFunction(await this.fetchRandomSeed()) : undefined
      )(
        onlyNfts.map((nft) => nft.id),
        this.version
      )
      const restOfTheRemarks =
        onlyNfts.length > addresses.length && this.price
          ? onlyNfts
              .slice(outOfTheNamesForTheRemarks.length)
              .map((nft) =>
                NFTUtils.createInteraction(
                  'LIST',
                  version,
                  nft.id,
                  String(price)
                )
              )
          : []

      this.isLoading = true

      const cb = api.tx.utility.batchAll
      const args = [...outOfTheNamesForTheRemarks, ...restOfTheRemarks].map(
        this.toRemark
      )

      const estimatedFee = await estimate(this.accountId, cb, [args])
      const support = feeTx(estimatedFee)
      args.push(support)

      const tx = await exec(
        this.accountId,
        '',
        cb,
        [args],
        txCb(
          async (blockHash) => {
            execResultValue(tx)
            const header = await api.rpc.chain.getHeader(blockHash)
            const blockNumber = header.number.toString()

            showNotification(
              `[SEND] Saved prices for ${
                this.rmrkMint.max
              } NFTs with tag ${formatBalance(price, {
                decimals: this.decimals,
                withUnit: this.unit,
              })} in block ${blockNumber}`,
              notificationTypes.success
            )

            this.isLoading = false
            const firstNft = remarks.find(NFTUtils.isNFT)

            if (firstNft) {
              this.navigateToDetail(firstNft, originalBlockNumber)
            }
          },
          (dispatchError) => {
            execResultValue(tx)
            this.onTxError(dispatchError)
            this.isLoading = false
          }
        )
      )
    } catch (e) {
      if (e instanceof Error) {
        showNotification(e.message, notificationTypes.danger)
        this.isLoading = false
      }
    }
  }

  protected onTxError(dispatchError: DispatchError): void {
    const { api } = Connector.getInstance()
    if (dispatchError.isModule) {
      const decoded = api.registry.findMetaError(dispatchError.asModule)
      const { docs, name, section } = decoded
      showNotification(
        `[ERR] ${section}.${name}: ${docs.join(' ')}`,
        notificationTypes.danger
      )
    } else {
      showNotification(
        `[ERR] ${dispatchError.toString()}`,
        notificationTypes.danger
      )
    }

    this.isLoading = false
  }

  get chainProperties() {
    return this.$store.getters['chain/getChainProperties']
  }

  get decimals(): number {
    return this.chainProperties.tokenDecimals
  }

  get unit(): string {
    return this.chainProperties.tokenSymbol
  }

  public async listForSale(remarks: NFT[], originalBlockNumber: string) {
    try {
      const { price, version } = this
      showNotification(
        `[APP] Listing NFT to sale for ${formatBalance(price, {
          decimals: this.decimals,
          withUnit: this.unit,
        })}`
      )

      const onlyNfts = remarks
        .filter(NFTUtils.isNFT)
        .map((nft) => ({ ...nft, id: getNftId(nft, originalBlockNumber) }))
        .map((nft) =>
          NFTUtils.createInteraction('LIST', version, nft.id, String(price))
        )

      if (!onlyNfts.length) {
        showNotification('Can not list empty NFTs', notificationTypes.danger)
        return
      }

      this.isLoading = true
      const { api } = Connector.getInstance()

      const cb = api.tx.utility.batchAll
      const args = onlyNfts.map(this.toRemark)

      const tx = await exec(
        this.accountId,
        '',
        cb,
        [args],
        txCb(
          async (blockHash) => {
            execResultValue(tx)
            const header = await api.rpc.chain.getHeader(blockHash)
            const blockNumber = header.number.toString()

            showNotification(
              `[LIST] Saved prices for ${
                this.rmrkMint.max
              } NFTs with tag ${formatBalance(price, {
                decimals: this.decimals,
                withUnit: this.unit,
              })} in block ${blockNumber}`,
              notificationTypes.success
            )

            this.isLoading = false
            const firstNft = remarks.find(NFTUtils.isNFT)

            if (firstNft) {
              this.navigateToDetail(firstNft, originalBlockNumber)
            }
          },
          (dispatchError) => {
            execResultValue(tx)
            this.onTxError(dispatchError)
            this.isLoading = false
          }
        )
      )
    } catch (e) {
      if (e instanceof Error) {
        showNotification(e.message, notificationTypes.danger)
        this.isLoading = false
      }
    }
  }

  public nsfwAttribute(): Attribute[] {
    if (!this.nsfw) {
      return []
    }

    return [{ trait_type: 'NSFW', value: Number(this.nsfw) }]
  }

  public offsetAttribute(): Attribute[] {
    if (!this.hasCarbonOffset) {
      return []
    }

    return [{ trait_type: 'carbonless', value: Number(this.hasCarbonOffset) }]
  }

  public async constructMeta(): Promise<string | undefined> {
    const { file, secondFile, rmrkMint, meta: m } = this
    if (!file) {
      throw new ReferenceError('No file found!')
    }

    const { token }: PinningKey = await this.$store.dispatch(
      'pinning/fetchPinningKey',
      this.accountId
    )

    const fileHash = await pinFileToIPFS(file, token)
    const secondFileHash = secondFile
      ? await pinFileToIPFS(secondFile, token)
      : undefined

    let imageHash: string | undefined = fileHash
    let animationUrl: string | undefined = undefined

    // if secondaryFileVisible(file) then assign secondaryFileHash to image and set animationUrl to fileHash
    if (secondaryFileVisible(file)) {
      animationUrl = fileHash
      imageHash = secondFileHash || IPFS_KODADOT_IMAGE_PLACEHOLDER
    }

    const attributes = [
      ...nsfwAttribute(this.nsfw),
      ...offsetAttribute(this.hasCarbonOffset),
    ]

    const meta = createMetadata(
      rmrkMint.name,
      m.description || '',
      imageHash,
      animationUrl,
      attributes,
      'https://kodadot.xyz',
      file.type
    )

    const metaHash = await pinJson(meta, imageHash)

    if (file) {
      uploadDirect(file, metaHash).catch(console.warn)
    }
    return unSanitizeIpfsUrl(metaHash)
  }

  private toRemark(remark: string) {
    const { api } = Connector.getInstance()
    return api.tx.system.remark(remark)
  }

  protected navigateToDetail(nft: NFT, blockNumber: string) {
    showNotification('You will go to the detail in 2 seconds')
    const go = () =>
      this.$router.push({
        path: `/rmrk/detail/${getNftId(nft, blockNumber)}`,
        query: { message: 'congrats' },
      })
    setTimeout(go, 2000)
  }

  protected getUsdFromKsm() {
    let KSMVal = formatBalance(this.estimated, {
      decimals: this.decimals,
      withUnit: false,
      forceUnit: '-',
    })

    return this.$store.getters['fiat/getCurrentKSMValue'] * Number(KSMVal)
  }

  @Watch('rmrkMint', { deep: true })
  rmrkMintObjectChanged(): void {
    if (this.canCalculateTransactionFees) {
      this.estimateTx()
    }
  }
}
</script>
