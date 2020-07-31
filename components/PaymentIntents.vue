<template>
  <div></div>
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
    } = {
      paymentIntents: {},
    }
    return data
  },
  computed: {
    ...mapState(['paymentIntentsVisible']),
    freshPaymentIntents(): Array<any> {
      const { paymentIntents } = this
      // console.log('paymentIntents :>> ', paymentIntents)

      const freshIntents = []
      for (const intent in paymentIntents) {
        // console.log('intent :>> ', intent)
        freshIntents.push(paymentIntents[intent])
      }

      freshIntents.sort(function (a, b) {
        return b.timestamp - a.timestamp
      })
      return freshIntents
    },
  },
  created() {
    this.pollPaymentIntents() // TODO enable
  },
  async mounted() {},
  methods: {
    ...mapActions(['queryDocuments']),
    ...mapMutations(['dismissPaymentIntent']),
    dismissIntent(docId: string) {
      console.log('this.paymentIntentsVisible :>> ', this.paymentIntentsVisible)
      this.dismissPaymentIntent(docId)
      console.log('this.paymentIntentsVisible :>> ', this.paymentIntentsVisible)
    },
    requestFromUserId(requesteeUserId: string) {
      const intent = this.paymentIntents[requesteeUserId]
      console.log('intent :>> ', intent)
      const refId = intent.$id
      const requesteeUserName = intent.requesteeUserName
      const fiatAmount = intent.encFiatAmount
      const POSOpts = {
        refId,
        requesteeUserId,
        requesteeUserName,
        fiatAmount,
        mode: 'Intent',
      }
      this.$store.commit('setPOSOptions', POSOpts)
      this.dismissIntent(intent.$id)
      this.$router.push('/charge')
    },
    async pollPaymentIntents() {
      if (this.$router.currentRoute.path !== '/') return

      const queryOpts = {
        limit: 10,
        startAt: 1,
        orderBy: [['timestamp', 'desc']],
        where: [
          ['requesterUserId', '==', this.$store.state.name.docId],
          ['timestamp', '>', timestamp() - 120],
        ],
      }
      const documents = await this.queryDocuments({
        contract: 'PaymentRequest',
        typeLocator: 'PaymentIntent',
        queryOpts,
      })
      this.paymentIntents = documents
        .reverse()
        .reduce(function (map: any, obj: any) {
          map[obj.data.requesteeUserId] = obj.toJSON()
          return map
        }, {})
      await sleep(1000)
      this.pollPaymentIntents()
    },
  },
})
</script>

<style scoped></style>
