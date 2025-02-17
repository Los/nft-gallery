<template>
  <CollapseCardWrapper :label="$t('nft.offer.label', [total])">
    <Loader v-model="isLoading" :status="status" />
    <p v-if="total" class="title is-size-4 has-text-success">
      {{ $t('nft.offer.count', [total]) }}
    </p>
    <OfferTable
      :offers="offers"
      :account-id="accountId"
      :is-owner="isOwner"
      @select="onOfferSelected" />
  </CollapseCardWrapper>
</template>

<script lang="ts">
import { Component, Emit, Prop, mixins } from 'nuxt-property-decorator'
import { isSameAccount } from '~/utils/account'
import { Offer, OfferResponse } from './types'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import { createTokenId } from '~/components/unique/utils'
import offerListByNftId from '@/queries/subsquid/bsx/offerListByNftId.graphql'
import SubscribeMixin from '~/utils/mixins/subscribeMixin'
import UseApiMixin from '~/utils/mixins/useApiMixin'
import OfferMixin from '~/utils/mixins/offerMixin'

const components = {
  Loader: () => import('@/components/shared/Loader.vue'),
  CollapseCardWrapper: () =>
    import('@/components/shared/collapse/CollapseCardWrapper.vue'),
  OfferTable: () => import('@/components/bsx/Offer/OfferTable.vue'),
}

@Component({ components })
export default class OfferList extends mixins(
  PrefixMixin,
  SubscribeMixin,
  UseApiMixin,
  OfferMixin
) {
  protected offers: Offer[] = []
  protected total = 0
  @Prop(String) public currentOwnerId!: string
  @Prop(String) public collectionId!: string
  @Prop(String) public nftId!: string

  get isOwner(): boolean {
    return Boolean(
      this.currentOwnerId &&
        this.accountId &&
        isSameAccount(this.currentOwnerId, this.accountId)
    )
  }

  // fetch() {
  //   this.fetchOffers()
  // }

  get tokenId(): [string, string] {
    return [this.collectionId, this.nftId]
  }

  public mounted() {
    this.$apollo.addSmartQuery<OfferResponse>('offers', {
      client: this.urlPrefix,
      query: offerListByNftId,
      variables: () => ({
        id: createTokenId(this.collectionId, this.nftId),
        account: this.currentOwnerId,
      }),
      result: ({ data }) => this.setResponse(data),
      manual: true,
      pollInterval: 15000,
    })
  }

  @Emit('offersUpdate')
  protected setResponse(response: OfferResponse) {
    this.offers = response.offers
    this.total = response.stats.total
  }

  protected fetchOffers() {
    try {
      this.$apollo.addSmartQuery<OfferResponse>('offersManualFetch', {
        client: this.urlPrefix,
        query: offerListByNftId,
        variables: () => ({
          id: createTokenId(this.collectionId, this.nftId),
          account: this.currentOwnerId,
        }),
        manual: true,
        result: ({ data }) => {
          this.setResponse(data)
        },
      })
    } catch (e) {
      this.$consola.error(e)
    }
  }

  public async onOfferSelected(maker: string) {
    const { collectionId, nftId } = this
    await this.submit(maker, nftId, collectionId, this.fetchOffers)
  }
}
</script>
