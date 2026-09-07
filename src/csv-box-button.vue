<script>
import { defineComponent, isProxy, toRaw } from 'vue';
import {
    buildImportUrl,
    generateUuid,
    buildInitPayload,
    createModalLifecycle,
    classifyStructuredMessage
} from '@csvbox/adapter';
import packageJson from '../package.json';

export default /*#__PURE__*/defineComponent({
  name: 'CSVBoxButton',
  props: {
    licenseKey: {
      type: String,
      required: true
    },
    onImport: {
      type: Function,
      default: function () {}
    },
    onReady: {
      type: Function,
      default: function () {}
    },
    onSubmit: {
      type: Function,
      default: function () {}
    },
    onClose: {
      type: Function,
      default: function () {}
    },
    user: {
      type: Object,
      default: function () {
        return { user_id: 'default123' };
      }
    },
    dynamicColumns: {
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
    },
    dataLocation: {
      type: String,
      required: false
    },
    customDomain: {
      type: String,
      required: false
    },
    language: {
      type: String,
      required: false
    },
    lazy: {
      type: Boolean,
      required: false,
      default: false
    },
    loadStarted: {
      type: Function,
      default: function () {}
    },
    environment: {
      type: Object,
      default: function () {
        return null;
      }
    },
    theme: {
      type: String,
      required: false
    }
  },
  computed: {
    iframeSrc() {
      return buildImportUrl(
        {
          licenseKey: this.licenseKey,
          customDomain: this.customDomain,
          dataLocation: this.dataLocation,
          language: this.language,
          theme: this.theme,
          environment: this.environment
        },
        "vue3",
        packageJson.version
      );
    }
  },
  data() {
    return {
      disableImportButton: true,
      uuid: generateUuid(),
      iframe: null,
      lifecycle: createModalLifecycle()
    }
  },
  methods: {
    openModal() {
      if (!this.iframe) {
        this.lifecycle.requestOpen();
        this.initImporter();
        return;
      }
      if (this.lifecycle.requestOpen()) {
        this.$refs.holder.style.display = 'block';
        this.iframe.contentWindow.postMessage('openModal', '*');
      }
    },
    onMessageEvent(event) {
      let message = classifyStructuredMessage(event.data, this.uuid);
      if (!message) {
        return;
      }

      if (message.type === "data-on-submit") {
        this.onSubmit?.(message.metadata);
      } else if (message.type === "data-push-status") {
        this.onImport(message.success, message.metadata);
      } else if (message.type === "csvbox-modal-hidden") {
        this.handleModalClosed();
      } else if (message.type === "csvbox-upload-successful") {
        this.onImport(true);
      } else if (message.type === "csvbox-upload-failed") {
        this.onImport(false);
      }
    },
    handleModalClosed() {
      if (this.$refs.holder) {
        this.$refs.holder.style.display = 'none';
        this.$refs.holder.innerHTML = '';
      }
      this.lifecycle.markClosed();
      this.iframe = null;
      this.onClose();
    },
    initImporter() {
      this.uuid = generateUuid();
      this.loadStarted();

      const iframe = document.createElement("iframe");
      this.iframe = iframe;
      iframe.setAttribute("src", this.iframeSrc);
      iframe.setAttribute("allow", "clipboard-read; clipboard-write *");
      iframe.frameBorder = 0;
      iframe.classList.add('csvbox-iframe');

      window.addEventListener("message", this.onMessageEvent, false);

      iframe.onload = () => {
        let user = this.user;
        if (isProxy(user)) { user = toRaw(user); }
        let dynamicColumns = this.dynamicColumns;
        if (isProxy(dynamicColumns)) { dynamicColumns = toRaw(dynamicColumns); }
        let options = this.options;
        if (isProxy(options)) { options = toRaw(options); }

        iframe.contentWindow.postMessage(buildInitPayload(user, dynamicColumns, options, this.uuid), "*");
        this.disableImportButton = false;
        this.onReady();
        if (this.lifecycle.markReady()) {
            this.openModal();
        }
      };

      this.$refs.holder.appendChild(iframe);
    }
  },
  mounted() {
    if (this.lazy) {
        this.disableImportButton = false;
    } else {
        this.initImporter();
    }
  },
  beforeUnmount() {
    window.removeEventListener("message", this.onMessageEvent);
  }
});
</script>

<template>
    <div>
        <button :disabled="disableImportButton" @click.prevent="openModal">
            <slot></slot>
        </button>
        <div ref="holder" class="holder-style"></div>
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
</style>
<style>
.csvbox-iframe {
    height: 100%;
    width: 100%;
    position: absolute;
    top: 0;
    left: 0;
}
</style>
