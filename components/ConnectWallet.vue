<template>
  <div>
    <v-card class="mx-auto mb-6" max-width="550px">
      <v-card-title class="text-center justify-center py-6">
        <h1 class="font-weight-bold display-1">Checkout</h1>
      </v-card-title>
      <v-card-text class="text-center justify-center py-6">
        Scan the QR Code with
        <a target="_blank" href="http://evowallet.io">EvoWallet</a> to checkout.
      </v-card-text>

      <v-tabs v-model="tab" centered>
        <v-tab>
          Guest
        </v-tab>
        <v-tab>
          Signup
        </v-tab>
        <v-tab>
          By Name
        </v-tab>
      </v-tabs>
      <v-tabs-items v-model="tab">
        <v-tab-item>
          <v-card-text class="text-center justify-center py-6">
            Choose for a single time purchase.<br />
            {{ $store.getters.qrCheckout }}
          </v-card-text>
          <v-card-text class="text-center justify-center py-6">
            <qrcode
              :value="$store.getters.qrCheckout"
              tag="img"
              style="margin-to: -5px;"
            ></qrcode>
          </v-card-text>
        </v-tab-item>
        <v-tab-item>
          <v-card-text class="text-center justify-center py-6">
            <v-card-text class="text-center justify-center py-6">
              Choose to enable "pay by name" in the future.
            </v-card-text>
            <qrcode
              :value="$store.state.name.label"
              tag="img"
              style="margin-to: -5px;"
            ></qrcode>
          </v-card-text>
        </v-tab-item>
        <v-tab-item>
          <v-card-text class="text-center justify-center py-6">
            <v-card-text class="text-center justify-center py-6">
              Signed up users can enter their name to pay.
            </v-card-text>
            <!-- <NameAutocomplete v-model="customer" /> -->
          </v-card-text>
        </v-tab-item>
      </v-tabs-items>
    </v-card>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { mapActions, mapMutations, mapState } from 'vuex'

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms))
const timestamp = () => Math.floor(Date.now() / 1000)

export default Vue.extend({
  data: () => {
    const data: {
      paymentIntents: any
      tab: any
    } = {
      paymentIntents: {},
      tab: null,
    }
    return data
  },
  created() {
    this.pollPaymentIntents() // TODO better to use a watcher for checkoutStep === 2
  },
  mounted() {
    console.log('this.tab :>> ', this.tab)
  },
  methods: {
    ...mapActions(['queryDocuments', 'requestFiat']),
    async pollPaymentIntents() {
      console.log(
        'this.$store.getters.checkoutStep :>> ',
        this.$store.getters.checkoutStep
      )
      if (this.$store.getters.checkoutStep !== 2) {
        await sleep(2000)
        this.pollPaymentIntents()
        return
      }

      const queryOpts = {
        limit: 1,
        startAt: 1,
        orderBy: [['timestamp', 'desc']],
        where: [
          ['requesterUserId', '==', this.$store.state.name.docId],
          ['invoiceId', '==', this.$store.state.selectedItem.invoiceId],
          ['timestamp', '>', timestamp() - 120],
        ],
      }
      const documents = await this.queryDocuments({
        contract: 'PaymentRequest',
        typeLocator: 'PaymentIntent',
        queryOpts,
      })
      if (documents.length) {
        const intent = documents[0].toJSON()
        this.$store.commit('setIntentDoc', intent)
        this.$store.commit('showSnackbar', {
          text: `Received Intent from ${intent.requesteeUserName}`,
          color: 'blue',
        })

        const requestDoc = await this.requestFiat({
          requesteeUserId: intent.requesteeUserId,
          requesteeUserName: intent.requesteeUserName,
          refId: intent.$id,
          fiatAmount: this.$store.state.selectedItem.fiatAmount,
          fiatSymbol: this.$store.state.selectedItem.fiatSymbol,
          memo: this.$store.state.selectedItem.name,
          invoiceId: this.$store.state.selectedItem.invoiceId,
          //   address = undefined,
        })
        console.log('requestDoc :>> ', requestDoc)
        this.$store.commit('setRequestDoc', requestDoc)

        this.$store.commit('showSnackbar', {
          text: `Sent Request to ${documents[0].data.requesteeUserName}`,
          color: 'blue',
        })
      }
      await sleep(1000)
      this.pollPaymentIntents()
    },
  },
})
</script>

<style scoped></style>
