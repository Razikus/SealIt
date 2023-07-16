<template>
  <q-page padding>
    <q-dialog persistent v-model="apikeyDialog">

      <q-card>
        <q-card-section title>
          <h5>Please provide read key</h5>
        </q-card-section>
        <q-separator></q-separator>
        <q-card-section>
          <q-input v-model="providedAPIKey" label="Read Key" type="password"> </q-input>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn color="accent" @click="unsealWithProvidedAPIKey" label="Unseal 🦭"></q-btn>

        </q-card-actions>

      </q-card>


    </q-dialog>
    <div class="q-pa-md">
      <div class="row  justify-center items-center flex-center">
        <q-img width="256px" src="~/assets/logo.png"></q-img>
      </div>
      <div class="row  justify-center items-center flex-center" v-if="!done">
        Your PDF should be downloaded in few seconds...
      </div>
      <div class="row  justify-center items-center flex-center" v-else>
        Done!
      </div>

    </div>
  </q-page>
</template>

<script>
import { ref } from 'vue';
import { api } from "../boot/axios"
import { useQuasar } from 'quasar'

export default {
  methods: {
    async unsealWithProvidedAPIKey() {
      let idOf = this.$route.params.what
      await this.tryLoadPDF(idOf, this.providedAPIKey)
      this.apikeyDialog = false
    },
    async tryLoadPDF(documentID, readapikey) {
      try {


        let result = await api.post("/ledger/default/collection/default/document/" + documentID + "/audit", {
          page: 1,
          perPage: 1,
          desc: true
        }, {
          headers: {
            "X-API-KEY": readapikey,
            "Content-Type": "application/json"
          }
        })
        if (result.status == 200) {
          let doc = result.data.revisions[0].document.doc

          let link = document.createElement("a");
          link.download = "unsealed-" + documentID + ".pdf";
          link.href = doc;
          link.click();
          URL.revokeObjectURL(doc);
          this.done = true
        }
      } catch (err) {
          this.$q.notify({
              message: "Something goes wrong with downloading an document. Check your credentials and refresh",
              color: "negative",
              position: "top-right"
            })

      }

    }
  },
  mounted() {
    let idOf = this.$route.params.what
    let readApiKey = this.$route.query.readkey
    if (!readApiKey) {
      this.apikeyDialog = true
    } else {
      this.tryLoadPDF(idOf, readApiKey)
    }
  },
  setup() {
    const $q = useQuasar()
    return {
      $q: $q,
      apikeyDialog: ref(false),
      done: ref(false),
      providedAPIKey: ref(""),
    }
  }
  // name: 'PageName',
}
</script>
