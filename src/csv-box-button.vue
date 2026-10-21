<script>
import { defineComponent, isProxy, toRaw } from 'vue';
import {
    buildImportUrl,
    generateUuid,
    buildInitPayload,
    createModalLifecycle,
    classifyStructuredMessage
} from '@csvbox/adapter';
// Replaced at build time with the version from package.json.
// See the `replace` config in build/rollup.config.js.
const LIBRARY_VERSION = "__CSVBOX_LIBRARY_VERSION__";

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
        LIBRARY_VERSION
      );
    },
    // Default place to send the file. A file is only ever sent to a named origin, never "*".
    targetOrigin() {
      return new URL(this.iframeSrc).origin;
    },
    // Data-location sheets redirect to another host, so the iframe origin can differ from
    // targetOrigin. baseDomain is used to check the real origin before trusting it.
    baseDomain() {
      return this.customDomain || "app.csvbox.io";
    }
  },
  data() {
    return {
      disableImportButton: true,
      uuid: generateUuid(),
      iframe: null,
      lifecycle: createModalLifecycle(),
      isModalShown: false,
      isIframeLoaded: false,
      pendingFile: null,
      importerOrigin: null,
      probingOrigin: false
    }
  },
  methods: {
    openModal() {
      if (!this.iframe) {
        this.lifecycle.requestOpen();
        this.initImporter();
        return;
      }

      if (this.pendingFile) {
        if (!this.isIframeLoaded) {
          // Deferred: onload's markReady() calls openModal() again once the iframe is up.
          this.lifecycle.requestOpen();
          return;
        }
        // The importer opens its own modal once it has the file, so that it can inject on
        // 'shown.bs.modal'. Sending 'openModal' as well would race that.
        //
        // It can also decline: wrong step, an import file URL configured on the sheet, or
        // an importer too old to know the message at all. So the holder goes up
        // click-through and stays that way until 'csvbox-modal-shown' confirms the modal
        // is really up. It covers the viewport at z-index 2147483647 and the importer
        // renders transparent with its modal closed, so arming it on an unanswered message
        // leaves an invisible sheet over the host page eating every click.
        //
        // display:block regardless, so the iframe lays out and animates normally; only
        // pointer-events is held back. isModalShown likewise waits for the confirmation,
        // or a declined file would latch the importer shut for good.
        this.$refs.holder.style.pointerEvents = 'none';
        this.$refs.holder.style.display = 'block';
        this.postPendingFile();
        return;
      }

      if (this.lifecycle.requestOpen()) {
        this.isModalShown = true;
        // Explicit: a previously declined file leaves the holder click-through.
        this.$refs.holder.style.pointerEvents = 'auto';
        this.$refs.holder.style.display = 'block';
        this.iframe.contentWindow.postMessage('openModal', '*');
      }
    },
    /**
     * Open the importer on a File the host page already has, instead of the file picker.
     *
     * Reachable through a template ref on the component
     * (`this.$refs.importer.openModalWithFile(file)`).
     *
     * The File crosses to the iframe by structured clone, so the importer receives the real
     * object and applies its own extension, worksheet and size rules to it -- this does not
     * bypass any of them. Pass a File; a Blob has no name for the importer to read an
     * extension from.
     */
    openModalWithFile(file) {
      if (!window.File || !(file instanceof window.File)) {
        return;
      }
      this.pendingFile = file;

      // Already open: hand it over now rather than holding it for the next open. The
      // importer takes it only while it is still on the upload step, and ignores it once
      // the user has a dataset and a mapping in progress.
      if (this.isModalShown && this.iframe && this.isIframeLoaded) {
        this.postPendingFile();
        return;
      }

      this.openModal();
    },
    postPendingFile() {
      if (this.importerOrigin) {
        this.sendPendingFile(this.importerOrigin);
        return;
      }

      // Origin not known yet: send a probe without the file, and send the file to the origin
      // that replies. If nobody replies in time, use targetOrigin like before.
      if (this.probingOrigin) {
        return;
      }
      this.probingOrigin = true;
      this.iframe.contentWindow.postMessage({
        type: 'csvbox-set-file',
        unique_token: this.uuid
      }, '*');
      this.probeTimer = setTimeout(() => {
        this.probeTimer = null;
        this.probingOrigin = false;
        this.sendPendingFile(this.targetOrigin);
      }, 1500);
    },
    sendPendingFile(origin) {
      if (!this.pendingFile || !this.iframe) {
        return;
      }
      this.iframe.contentWindow.postMessage({
        type: 'csvbox-set-file',
        // toRaw: data() wraps the File in a reactive Proxy, which structured clone rejects.
        file: toRaw(this.pendingFile),
        unique_token: this.uuid
      }, origin);
      this.pendingFile = null;
    },
    // Remember the importer's origin, only from our own iframe, with our token, and a csvbox host.
    learnImporterOrigin(event) {
      if (this.importerOrigin || !this.iframe || event.source !== this.iframe.contentWindow) {
        return;
      }
      if (!this.isImporterOrigin(event.origin)) {
        return;
      }
      this.importerOrigin = event.origin;
      if (this.probingOrigin) {
        this.probingOrigin = false;
        this.clearProbeTimer();
        this.sendPendingFile(this.importerOrigin);
      }
    },
    isImporterOrigin(origin) {
      let url;
      try {
        url = new URL(origin);
      } catch (e) {
        return false;
      }
      if (url.protocol !== 'https:') {
        return false;
      }
      return url.host === this.baseDomain || url.host.endsWith('-' + this.baseDomain);
    },
    clearProbeTimer() {
      if (this.probeTimer) {
        clearTimeout(this.probeTimer);
        this.probeTimer = null;
      }
    },
    onMessageEvent(event) {
      // Set-file handshake messages. The shared core does not classify these, so they are
      // read here, behind the same unique_token check the core applies.
      const raw = event.data;
      if (raw && typeof raw === "object" && raw.data && raw.data.unique_token == this.uuid) {
        this.learnImporterOrigin(event);

        if (raw.type === "csvbox-set-file-rejected" && raw.data.reason === "not-a-file") {
          // Reply to our probe (it has no file). It only tells us the origin, so do nothing.
          return;
        }
        if (raw.type === "csvbox-modal-shown") {
          // What the file path in openModal() is waiting for. display as well as
          // pointer-events: a second file declined while the first was still opening the
          // modal has put the holder away in the meantime.
          this.isModalShown = true;
          this.lifecycle.requestOpen();
          this.$refs.holder.style.display = 'block';
          this.$refs.holder.style.pointerEvents = 'auto';
          return;
        }
        if (raw.type === "csvbox-set-file-rejected") {
          // The importer has the file and will not take it. Put the holder away only if the
          // modal was never confirmed up: a file declined while it is open leaves that modal
          // where it is, and its own 'csvbox-modal-hidden' will put the holder away.
          if (!this.isModalShown) {
            this.$refs.holder.style.display = 'none';
            this.$refs.holder.style.pointerEvents = 'auto';
          }
          console.warn("[csvbox] importer declined the supplied file: " + raw.data.reason);
          return;
        }
      }

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
        this.$refs.holder.style.pointerEvents = 'auto';
        this.$refs.holder.innerHTML = '';
      }
      this.lifecycle.markClosed();
      // A new iframe is built on the next open, so the origin is learned again.
      this.importerOrigin = null;
      this.probingOrigin = false;
      this.clearProbeTimer();
      this.isModalShown = false;
      this.isIframeLoaded = false;
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
        this.isIframeLoaded = true;
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
    this.clearProbeTimer();
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
