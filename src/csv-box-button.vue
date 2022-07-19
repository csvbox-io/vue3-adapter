<script>
import { defineComponent } from 'vue';
import { isProxy, toRaw } from 'vue';

export default /*#__PURE__*/defineComponent({
  name: 'CSVBoxButton', // vue component name
  props: {
    licenseKey: {
        type: String,
        required: true
    },
    debugMode: {
        type: Boolean,
        required: false
    },
    useStagingServer: {
        type: Boolean,
        required: false
    },
    onImport: {
        type: Function,
        default: function() {}
    },
    onReady: {
        type: Function,
        default: function() {}
    },
    onClose:{
        type: Function,
        default: function() {}
    },
    user: {
        type: Object,
        default: function () {
            return { user_id: 'default123' };
        }
    },
    dynamicColumns:{
        type: Array,
        default: function () {
            return null;
        }
    },
    options: {
        type: Object,
        default: function () {
            return { user_id: 'default123' };
        }
    }
  },
  computed:{
      iframeSrc() {
          let BASE_URL = `https://${this.useStagingServer ? 'staging' : 'app' }.csvbox.io/embed/${this.licenseKey}`;
          return `${BASE_URL}?library-version=2&source=embedCode&sourceLang=vue`;
      }
  },
  data(){
      return {
          isModalShown: false,
          disableImportButton: true,
          uuid: Math.random().toString(36).substring(2, 15) + Math.random().toString(36).substring(2, 15)
      }
  },
  methods: {
      openModal() {
          if(!this.isModalShown) {
              this.isModalShown = true;
              this.$refs.holder.style.display = 'block';
              this.$refs.iframe.contentWindow.postMessage('openModal', '*');
          }
      },
      onMessageEvent(event) {
          if (event.data === "mainModalHidden") {
              this.$refs.holder.style.display = 'none';
              this.isModalShown = false;
              this.onClose();
          }
          if(event.data === "uploadSuccessful") {
              this.onImport(true);
          }
          if(event.data === "uploadFailed") {
              this.onImport(false);
          }
          if(typeof event.data == "object") {
              if(event?.data?.data?.unique_token == this.uuid) {
                  if(event.data.type && event.data.type == "data-push-status") {
                      if(event.data.data.import_status == "success") {
                          this.onImport(true, event.data.data);
                      } else {
                          this.onImport(false, event.data.data);
                      }
                  }
              }
          }
      },
      randomString() {
          let result = '';
          let characters = 'abcdefghijklmnopqrstuvwxyz0123456789';
          let charactersLength = characters.length;
          for (let i = 0; i < 20; i++) {
              result += characters.charAt(Math.floor(Math.random() * charactersLength)); 
          }
          return result;
      }
  },
  mounted() {

      window.addEventListener("message", this.onMessageEvent, false);

      let iframe = this.$refs.iframe;

      let self = this;

      iframe.onload = function () {

        iframe.contentWindow.postMessage({
            "customer" : self.user ? (isProxy(self.user) ? toRaw(self.user) : self.user) : null,
            "columns" : self.dynamicColumns ? (isProxy(self.dynamicColumns) ? toRaw(self.dynamicColumns) : self.dynamicColumns) : null,
            "options" : self.options ? (isProxy(self.options) ? toRaw(self.options) : self.options) : null,
            "unique_token": self.uuid
        }, "*");

          self.disableImportButton = false;

          self.onReady();

      }
  },
  beforeDestroy() {
      window.removeEventListener("message", this.onMessageEvent);
  }
});
</script>

<template>
    <div>
        <button :disabled="disableImportButton" @click="openModal">
            <slot></slot>
        </button>
        <div ref="holder" class="holder-style">
            <iframe ref="iframe" class="iframe" :src="iframeSrc" frameBorder="0"></iframe>
        </div>
    </div>
</template>

<style scoped>
    .holder-style {
        display: none;
        z-index: 2147483647;
        position: fixed;
        top: 0;
        bottom: 0;
        left: 0;
        right: 0;
    }
    .iframe {
        height: 100%;
        width: 100%;
        position: absolute;
        top: 0;
        left: 0;
    }
</style>
