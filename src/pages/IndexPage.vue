<template>
  <q-page class="flex flex-center">
    <div class="q-pa-md">
      <div class="row  justify-center items-center flex-center">
        <div class="col-3">
          <q-img width="256px" src="~/assets/logo.png"></q-img>
        </div>

      </div>
      <div class="row justify-center">
        <q-card class="uploadcard">
          <q-card-section>
            <q-card-title>
              Seal your PDF
            </q-card-title>
            <q-card-section>
              <q-form ref="mainform">
              <q-file v-model="fileChoosed" :rules="[val => !! val || 'PDF is required']" outlined label="Drop or choose PDF here (max 1mb)" accept=".pdf, application/pdf"
                max-file-size="1048576" @rejected="onRejected" />
                <q-input label="Immudb Vault Write Key" :rules="[val => !! val || 'Write API Key is required']"  type="password" v-model="writeapikey" outlined></q-input>
                <q-input label="Immudb Vault Read Key (optional)" v-model="readapikey"  type="password" outlined></q-input>
              </q-form>
            </q-card-section>
            <q-card-actions align="right">
              <q-btn color="primary" label="Seal it 🦭!" @click="stampFile" />
            </q-card-actions>
            <q-card-actions>
              (All data are processed in browser, and saved directly into Immudb Vault. No external server is involved)
            </q-card-actions>
          </q-card-section>

        </q-card>
      </div>
    </div>
  </q-page>
</template>

<style>
.uploadcard {
  width: 1024px;
}
</style>

<script>
import { defineComponent, ref } from 'vue'
import { degrees, PDFDocument, rgb, StandardFonts } from 'pdf-lib';
import QRCode from 'qrcode'
import { api } from "../boot/axios"
import { useQuasar } from 'quasar'



export default defineComponent({
  name: 'IndexPage',
  methods: {
    onRejected(files) {
      this.$q.notify({
          message: "Files could not be processed. Must be PDF smaller than 1MB",
          color: "negative",
          position: "top-right"
        })
    },
    dataURItoBlob(dataURI) {
      var byteString = atob(dataURI.split(',')[1]);
      var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0]
      var ab = new ArrayBuffer(byteString.length);
      var ia = new Uint8Array(ab);
      for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
      }
      var blob = new Blob([ab], { type: mimeString });
      return blob;

    },
    async addPDFDocument(doc) {
      let base64d = await this.toBase64(doc)
      try {
        
      let result = await api.put("/ledger/default/collection/default/document", {
        doc: base64d
      }, {
        headers: {
          "X-API-KEY": this.writeapikey,
          "Content-Type": "application/json"
        }
      })
      if (result.status == 200) {
        return result.data.documentId
      }
      return null
    } catch(err ) {

      this.$q.notify({
          message: "Something goes wrong with uploading an document. Check your credentials",
          color: "negative",
          position: "top-right"
        })
      return null
    }

    },
    toBase64(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result);
        reader.onerror = reject;
      })

    },
    async stampFile() {
      const validated = await this.$refs.mainform.validate()
      if(!validated) {
        this.$q.notify({
          message: "You must provide required fields",
          color: "negative",
          position: "top-right"
        })
        return

      }
      let that = this
      var reader = new FileReader();
      let documentID = await this.addPDFDocument(this.fileChoosed)
      if(documentID == null ){
        return
      }
      reader.onload = async function () {

        var arrayBuffer = this.result
        var array = new Uint8Array(arrayBuffer)
        const pdfDoc = await PDFDocument.load(array)
        const helveticaFont = await pdfDoc.embedFont(StandardFonts.Helvetica)

        const pages = pdfDoc.getPages()

        const page = pdfDoc.addPage()
        const { width, height } = page.getSize()
        let sealItVerify = window.location.origin + "/#/verify/" + documentID
        if(that.readapikey != "") {
          sealItVerify = sealItVerify + "?readkey=" + encodeURIComponent(that.readapikey)
        }
        let url2 = await QRCode.toDataURL(sealItVerify)
        let pngImage = await pdfDoc.embedPng(url2)
        const pngDims = pngImage.scale(1)
        page.drawImage(pngImage, {
          x: 100,
          y: page.getHeight() / 2 - pngDims.height + 250,
          width: pngDims.width,
          height: pngDims.height,
        })

        page.drawText('Document was sealed in Immudb Vault. Scan QR code to see original document.', {
          x: 100,
          y: height / 2 + 300,
          size: 10,
          font: helveticaFont,
          color: rgb(0, 0, 0)
        })

        const pdfBytes = await pdfDoc.save()
        let blob = new Blob([pdfBytes], { type: "application/pdf" });
        let url = URL.createObjectURL(blob);
        let link = document.createElement("a");
        link.download = "sealed-" + documentID + ".pdf";
        link.href = url;
        link.click();
        URL.revokeObjectURL(url);
      }
      reader.readAsArrayBuffer(this.fileChoosed);

    }
  },
  setup() {
    const $q = useQuasar()
    return {
      $q: $q,
      writeapikey: ref(""),
      readapikey: ref(""),
      fileChoosed: ref(null)
    }
  }
})
</script>
