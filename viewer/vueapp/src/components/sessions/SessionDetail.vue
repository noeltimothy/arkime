<!--
Copyright Yahoo Inc.
SPDX-License-Identifier: Apache-2.0
-->
<template>

  <!-- session detail -->
  <div>

    <!-- detail loading -->
    <div v-if="loading"
      class="mt-1 mb-1 large">
      <span class="fa fa-spinner fa-spin"></span>&nbsp;
      Loading session detail...
    </div> <!-- /detail loading -->

    <!-- detail error -->
    <h5 v-if="error"
      class="text-danger mt-3 mb-3 ml-2">
      <span class="fa fa-exclamation-triangle"></span>&nbsp;
      {{ error }}
    </h5> <!-- /detail error -->

    <!-- detail -->
    <div class="detail-container"
      :ref="`detailContainer-${sessionIndex}`"
      :id="`detailContainer-${sessionIndex}`">
    </div> <!-- /detail -->

    <!-- packet options -->
    <div v-show="!loading && !hidePackets && !user.hidePcap"
      class="packet-options mr-1 ml-1">
      <form class="form-inline mb-2 pt-2 border-top">
        <fieldset :disabled="hidePackets || loading || loadingPackets || renderingPackets">
          <packet-options
            :params="params"
            :decodings="decodings"
            :cyberChefSrcUrl="cyberChefSrcUrl"
            :cyberChefDstUrl="cyberChefDstUrl"
            @updateBase="updateBase"
            @toggleImages="toggleImages"
            @toggleShowSrc="toggleShowSrc"
            @toggleShowDst="toggleShowDst"
            @applyDecodings="applyDecodings"
            @updateDecodings="updateDecodings"
            @toggleTimestamps="toggleTimestamps"
            @toggleShowFrames="toggleShowFrames"
            @updateNumPackets="updateNumPackets"
            @toggleLineNumbers="toggleLineNumbers"
            @toggleCompression="toggleCompression"
          />
        </fieldset>
      </form>
    </div> <!-- /packet options -->

    <!-- packets loading -->
    <div v-if="!loading && loadingPackets && !hidePackets && !user.hidePcap"
      class="mt-4 mb-4 ml-2 mr-2 large">
      <span class="fa fa-spinner fa-spin">
      </span>&nbsp;
      Loading session packets&nbsp;
      <button type="button"
        @click="cancelPacketLoad"
        class="btn btn-warning btn-xs">
        <span class="fa fa-ban">
        </span>&nbsp;
        cancel
      </button>
    </div> <!-- /packets loading -->

    <!-- packets rendering -->
    <div v-if="!loading && renderingPackets && !hidePackets && !user.hidePcap"
      class="mt-4 mb-4 ml-2 mr-2 large">
      <span class="fa fa-spinner fa-spin">
      </span>&nbsp;
      Rendering session packets
    </div> <!-- /packets rendering -->

    <!-- packets error -->
    <div v-if="!error && errorPackets"
      class="mt-4 mb-4 ml-2 mr-2 large">
      <span class="text-danger">
        <span class="fa fa-exclamation-triangle">
        </span>&nbsp;
        {{ errorPackets }}&nbsp;
      </span>
      <button type="button"
        @click="getPackets"
        class="btn btn-success btn-xs">
        <span class="fa fa-refresh">
        </span>&nbsp;
        retry
      </button>
    </div> <!-- /packets error -->

    <!-- packets -->
    <div v-if="!loadingPackets && !errorPackets && !hidePackets && !user.hidePcap"
      class="inner packet-container mr-1 ml-1"
      v-html="packetHtml"
      ref="packetContainer"
      :class="{'show-ts':params.ts,'hide-src':!params.showSrc,'hide-dst':!params.showDst}">
    </div> <!-- packets -->

    <!-- packet options -->
    <div v-show="!loading && !loadingPackets && !errorPackets && !hidePackets && !user.hidePcap"
      class="mr-1 ml-1">
      <form class="form-inline mb-2 pt-2">
        <fieldset :disabled="loading || loadingPackets || renderingPackets">
          <packet-options
            :params="params"
            :decodings="decodings"
            :cyberChefSrcUrl="cyberChefSrcUrl"
            :cyberChefDstUrl="cyberChefDstUrl"
            @updateBase="updateBase"
            @toggleImages="toggleImages"
            @toggleShowSrc="toggleShowSrc"
            @toggleShowDst="toggleShowDst"
            @applyDecodings="applyDecodings"
            @updateDecodings="updateDecodings"
            @toggleTimestamps="toggleTimestamps"
            @toggleShowFrames="toggleShowFrames"
            @updateNumPackets="updateNumPackets"
            @toggleLineNumbers="toggleLineNumbers"
            @toggleCompression="toggleCompression"
          />
        </fieldset>
      </form>
    </div> <!-- /packet options -->

  </div> <!-- /session detail -->

</template>

<script>
import Vue from 'vue';
import qs from 'qs';
import sanitizeHtml from 'sanitize-html';
import SessionsService from './SessionsService';
import ArkimeTagSessions from '../sessions/Tags';
import ArkimeRemoveData from '../sessions/Remove';
import ArkimeSendSessions from '../sessions/Send';
import ArkimeExportPcap from '../sessions/ExportPcap';
import ArkimeToast from '../utils/Toast';
import PacketOptions from './PacketOptions';
import FieldActions from './FieldActions';
import UserService from '../users/UserService';

const defaultUserSettings = {
  detailFormat: 'last',
  numPackets: 'last',
  showTimestamps: 'last'
};

// dl resize variables and functions
let selectedDT; // store selected dt to watch drag and calculate new width
let dtOffset; // store offset width to calculate new width
let selectedGrip; // the column resize grip that is currently being dragged
let siblingDD; // the dd element following the dt element in the dl that is being resized

// fired when a column resize grip is clicked
// stores values for calculations when the grip is unclicked
function gripClick (e, div) {
  e.preventDefault();
  e.stopPropagation();
  selectedDT = div.getElementsByTagName('dt')[0];
  siblingDD = selectedDT.nextElementSibling;
  dtOffset = selectedDT.offsetWidth - e.pageX;
  selectedGrip = div.getElementsByClassName('session-detail-grip')[0];
};

// fired when the column resize grip is dragged
// styles the grip to show where it's being dragged
function gripDrag (e) { // move the grip where the user moves their cursor
  if (selectedDT && selectedGrip) {
    const newWidth = dtOffset + e.pageX;
    selectedGrip.style.borderRight = '1px dotted var(--color-gray)';
    selectedGrip.style.left = `${newWidth + 22}px`;
  }
}

// fired when a clicked and dragged grip is dropped
// updates the column and table width and saves the values
function gripUnclick (e, vueThis) {
  if (selectedDT && selectedGrip) {
    const newWidth = Math.max(dtOffset + e.pageX, 100); // min width is 100px
    selectedDT.style.width = `${newWidth}px`;
    siblingDD.style.marginLeft = `${newWidth + 10}px`;
    selectedGrip.style.left = `${newWidth + 22}px`;
    selectedGrip.style.borderRight = 'none';

    // update all the dt and dd styles to reflect the new width
    for (const dt of document.getElementsByTagName('dt')) {
      dt.style.width = `${newWidth}px`;
      dt.nextElementSibling.style.marginLeft = `${newWidth + 10}px`;
    }

    for (const grip of document.getElementsByClassName('session-detail-grip')) {
      grip.style.left = `${newWidth + 22}px`;
    }

    // save it as a user configuration
    vueThis.saveDLWidth(newWidth);
  }

  selectedGrip = undefined;
  selectedDT = undefined;
}

export default {
  name: 'ArkimeSessionDetail',
  props: [
    'session',
    'sessionIndex'
  ],
  components: { PacketOptions },
  data () {
    return {
      component: undefined,
      error: '',
      loading: true,
      hidePackets: true,
      errorPackets: '',
      loadingPackets: false,
      packetPromise: undefined,
      detailHtml: undefined,
      packetHtml: undefined,
      renderingPackets: false,
      decodingForm: false,
      decodings: {},
      params: {
        base: 'natural',
        line: false,
        image: false,
        gzip: false,
        ts: false,
        decode: {},
        packets: 200,
        showFrames: false,
        showSrc: true,
        showDst: true,
        dlWidth: 160
      }
    };
  },
  computed: {
    user: function () {
      return this.$store.state.user;
    },
    cyberChefSrcUrl: function () {
      return `cyberchef.html?nodeId=${this.session.node}&sessionId=${this.session.id}&type=src`;
    },
    cyberChefDstUrl: function () {
      return `cyberchef.html?nodeId=${this.session.node}&sessionId=${this.session.id}&type=dst`;
    }
  },
  created: function () {
    this.setUserParams();
    this.getDetailData();
    UserService.getState('sessionDetailDLWidth').then((response) => {
      this.dlWidth = response.data.width ?? 160;
    });
  },
  methods: {
    /* exposed functions --------------------------------------------------- */
    /* Cancels the packet loading request */
    cancelPacketLoad: function () {
      if (this.packetPromise) {
        this.packetPromise.source.cancel();
        this.packetPromise = null;
        this.loadingPackets = false;
        this.errorPackets = 'Request canceled';
      }
    },
    updateBase (value) {
      this.params.base = value;
      this.getPackets();
    },
    updateNumPackets (value) {
      this.params.packets = value;
      this.getPackets();
    },
    toggleShowFrames: function () {
      this.params.showFrames = !this.params.showFrames;

      if (this.params.showFrames) {
        // show timestamps and info by default for show frames option
        this.params.ts = true;
        // disable other options that don't make sense for raw packets
        this.params.gzip = false;
        this.params.image = false;
        this.params.decode = {};
      } else {
        // reset showFrames/gzip/image/decode options back to what was last used
        this.setBrowserParams();
      }

      this.getPackets();
    },
    toggleShowSrc: function () {
      this.params.showSrc = !this.params.showSrc;
    },
    toggleShowDst: function () {
      this.params.showDst = !this.params.showDst;
    },
    toggleLineNumbers: function () {
      // can only have line numbers in hex mode
      if (this.params.base !== 'hex') { return; }
      this.params.line = !this.params.line;
      this.getPackets();
    },
    toggleCompression: function () {
      this.params.gzip = !this.params.gzip;
      this.getPackets();
    },
    toggleImages: function () {
      this.params.image = !this.params.image;
      this.getPackets();
    },
    toggleTimestamps: function () {
      this.params.ts = !this.params.ts;
      if (localStorage && this.user.settings.showTimestamps === 'last') {
        // update browser saved ts if the user settings is set to last
        localStorage['moloch-ts'] = this.params.ts;
      }
    },
    applyDecodings (decodings) {
      this.params.decode = decodings;
      this.getPackets();
    },
    updateDecodings (decodings) {
      this.$set(this, 'decodings', decodings);
    },
    /* helper functions ---------------------------------------------------- */
    /**
     * Gets the session detail from the server
     * @param {string} message An optional message to display to the user
     */
    getDetailData: function (message, messageType) {
      this.loading = true;

      if (this.component) {
        this.component.$destroy(true);
        this.component.$el.parentNode.removeChild(this.component.$el);
        this.component = undefined;
      }

      SessionsService.getDetail(this.session.id, this.session.node, this.session.cluster).then((response) => {
        this.loading = false;

        this.hidePackets = response.data.includes('hidePackets="true"');
        if (!this.hidePackets) {
          this.getDecodings(); // IMPORTANT: kicks of packet request
        }

        this.component = new Vue({
          // template string here
          template: response.data,
          // makes $parent work
          parent: this,
          // any props the component should receive.
          // reference to data in the template will *not* have access the the current
          // components scope, as you create a new component
          propsData: {
            session: this.session,
            fields: this.$store.state.fieldsMap,
            remoteclusters: this.$store.state.remoteclusters
          },
          props: ['session', 'fields', 'remoteclusters'],
          data: function () {
            return {
              form: undefined,
              cluster: undefined,
              message,
              messageType
            };
          },
          mounted () {
            this.$nextTick(() => { // wait for content to render
              // add grip to each section of the section detail
              const sessionDetailSection = document.getElementsByTagName('dl');
              const dlWidth = (this.$parent.dlWidth || this.dlWidth) + 22;
              for (const div of sessionDetailSection) {
                // set the width of the session detail div based on user setting
                const grip = document.createElement('div');
                grip.classList.add('session-detail-grip');
                grip.style.height = `${div.clientHeight}px`;
                grip.style.left = `${dlWidth}px`;
                div.prepend(grip);
                grip.addEventListener('mousedown', (e) => gripClick(e, div));
              }

              const dts = document.getElementsByTagName('dt');
              for (const dt of dts) {
                // set the width of the dt and the margin of the dd based on user setting
                dt.style.width = `${this.$parent.dlWidth}px`;
                dt.nextElementSibling.style.marginLeft = `${this.$parent.dlWidth + 10}px`;
              }

              // listen for grip drags
              document.addEventListener('mousemove', gripDrag);
              const self = this; // listen for grip unclicks
              document.addEventListener('mouseup', (e) => gripUnclick(e, self));
            });
          },
          computed: {
            expression: {
              get: function () {
                return this.$store.state.expression;
              },
              set: function (newValue) {
                this.$store.commit('setExpression', newValue);
              }
            },
            startTime: {
              get: function () {
                return this.$store.state.time.startTime;
              },
              set: function (newValue) {
                this.$store.commit('setTime', { startTime: newValue });
              }
            }
          },
          methods: {
            /* Saves the dl widths */
            saveDLWidth: function (width) {
              UserService.saveState({ width }, 'sessionDetailDLWidth');
            },
            getField: function (expr) {
              if (!this.fields[expr]) {
                console.log('UNDEFINED', expr);
              }
              return this.fields[expr];
            },
            actionFormDone: function (doneMsg, success, reload) {
              this.form = undefined; // clear the form
              const doneMsgType = success ? 'success' : 'warning';

              if (reload) {
                this.$parent.getDetailData(doneMsg, doneMsgType);
                this.getPackets();
                return;
              }

              if (doneMsg) {
                this.message = doneMsg;
                this.messageType = doneMsgType;
              }
            },
            messageDone: function () {
              this.message = undefined;
              this.messageType = undefined;
            },
            addTags: function () {
              this.form = 'add:tags';
            },
            removeTags: function () {
              this.form = 'remove:tags';
            },
            exportPCAP: function () {
              this.form = 'export:pcap';
            },
            removeData: function () {
              this.form = 'remove:data';
            },
            sendSession: function (cluster) {
              this.form = 'send:session';
              this.cluster = cluster;
            },
            openPermalink: function () {
              const id = this.session.id.split(':');
              let prefixlessId = id.length > 1 ? id[1] : id[0];
              if (prefixlessId[1] === '@') {
                prefixlessId = prefixlessId.substr(2);
              }

              const params = {
                expression: `id == ${prefixlessId}`,
                startTime: Math.floor(this.session.firstPacket / 1000),
                stopTime: Math.ceil(this.session.lastPacket / 1000),
                openAll: 1
              };

              const url = `sessions?${qs.stringify(params)}`;

              window.location = url;
            },
            /**
             * Adds a rootId expression and applies a new start time
             * @param {string} rootId The root id of the session
             * @param {int} startTime The start time of the session
             */
            allSessions: function (rootId, startTime) {
              startTime = Math.floor(startTime / 1000);

              const fullExpression = `rootId == ${rootId}`;

              this.expression = fullExpression;

              if (this.$route.query.startTime) {
                if (this.$route.query.startTime < startTime) {
                  startTime = this.$route.query.startTime;
                }
              }

              this.startTime = startTime;
            },
            /**
             * Opens a new browser tab containing all the unique values for a given field
             * @param {string} fieldID  The field id to display unique values for
             * @param {int} counts      Whether to display the unique values with counts (1 or 0)
             */
            exportUnique: function (fieldID, counts) {
              SessionsService.exportUniqueValues(fieldID, counts, this.$route.query);
            },
            /**
             * Opens the spi graph page in a new browser tab
             * @param {string} fieldID The field id (dbField) to display spi graph data for
             */
            openSpiGraph: function (fieldID) {
              SessionsService.openSpiGraph(fieldID, this.$route.query);
            },
            /**
             * Shows more items in a list of values
             * @param {object} e The click event
             */
            showMoreItems: function (e) {
              e.target.style.display = 'none';
              e.target.previousSibling.style.display = 'inline-block';
            },
            /**
             * Hides more items in a list of values
             * @param {object} e The click event
             */
            showFewerItems: function (e) {
              e.target.parentElement.style.display = 'none';
              e.target.parentElement.nextElementSibling.style.display = 'inline-block';
            },
            /**
             * Toggles a column in the sessions table
             * @param {string} fieldID  The field id to toggle in the sessions table
             */
            toggleColVis: function (fieldID) {
              this.$parent.toggleColVis(fieldID);
            },
            /**
             * Toggles a field's visibility in the info column
             * @param {string} fieldID  The field id to toggle in the info column
             */
            toggleInfoVis: function (fieldID) {
              this.$parent.toggleInfoVis(fieldID);
            },
            /**
             * Adds field == EXISTS! to the search expression
             * @param {string} field  The field name
             * @param {string} op     The relational operator
             */
            fieldExists: function (field, op) {
              const fullExpression = this.$options.filters.buildExpression(field, 'EXISTS!', op);
              this.$store.commit('addToExpression', { expression: fullExpression });
            }
          },
          components: {
            ArkimeTagSessions,
            ArkimeRemoveData,
            ArkimeSendSessions,
            ArkimeExportPcap,
            ArkimeToast,
            FieldActions
          }
        }).$mount();

        $(`#detailContainer-${this.sessionIndex}`).append(this.component.$el);
      }).catch((error) => {
        this.loading = false;
        this.error = error.text || error;
      });
    },
    toggleColVis: function (fieldID) {
      this.$emit('toggleColVis', fieldID);
    },
    toggleInfoVis: function (fieldID) {
      this.$emit('toggleInfoVis', fieldID);
    },
    /* sets some of the session detail query parameters based on user settings */
    setUserParams: function () {
      if (localStorage && this.user.settings) { // display user saved options
        if (this.user.settings.detailFormat === 'last' && localStorage['moloch-base'] && localStorage['moloch-base'] !== 'last') {
          this.params.base = localStorage['moloch-base'];
        } else if (this.user.settings.detailFormat && this.user.settings.detailFormat !== 'last') {
          this.params.base = this.user.settings.detailFormat;
        }

        if (this.user.settings.numPackets === 'last') {
          this.params.packets = localStorage['moloch-packets'] || 200;
        } else {
          this.params.packets = this.user.settings.numPackets || 200;
        }

        if (this.user.settings.showTimestamps === 'last' && localStorage['moloch-ts']) {
          this.params.ts = localStorage['moloch-ts'] === 'true';
        } else if (this.user.settings.showTimestamps) {
          this.params.ts = this.user.settings.showTimestamps === 'on';
        }
      } else if (!this.user.settings) {
        this.user.settings = defaultUserSettings;
      }
    },
    /**
     * retrieves other decodings for packet data
     * IMPORTANT: kicks of packet request
     */
    getDecodings: function () {
      SessionsService.getDecodings().then((response) => {
        this.decodings = response;
        this.setBrowserParams();
        this.getPackets();
      }).catch(() => {
        this.setBrowserParams();
        // still get the packets
        this.getPackets();
      });
    },
    /* sets some of the session detail query parameters based on browser saved options */
    setBrowserParams: function () {
      if (localStorage) { // display browser saved options
        if (localStorage['moloch-line']) {
          this.params.line = JSON.parse(localStorage['moloch-line']);
        }
        if (localStorage['moloch-gzip']) {
          this.params.gzip = JSON.parse(localStorage['moloch-gzip']);
        }
        if (localStorage['moloch-image']) {
          this.params.image = JSON.parse(localStorage['moloch-image']);
        }
        if (localStorage['moloch-decodings']) {
          this.params.decode = JSON.parse(localStorage['moloch-decodings']);
          for (const key in this.decodings) {
            this.decodings[key].active = false;
            if (this.params.decode[key]) {
              this.decodings[key].active = true;
              for (const field in this.params.decode[key]) {
                for (let i = 0, len = this.decodings[key].fields.length; i < len; ++i) {
                  if (this.decodings[key].fields[i].key === field) {
                    this.decodings[key].fields[i].value = this.params.decode[key][field];
                  }
                }
              }
            }
          }
        }
      }
    },
    /* Gets the packets for the session from the server */
    getPackets: function () {
      // if the user is not allowed to view packets, don't request them
      if (this.user.hidePcap) { return; }

      // already loading, don't load again!
      if (this.loadingPackets || this.hidePackets) { return; }

      this.loadingPackets = true;
      this.errorPackets = false;

      if (localStorage) { // update browser saved options
        if (this.user.settings.detailFormat === 'last') {
          localStorage['moloch-base'] = this.params.base;
        }
        if (this.user.settings.numPackets === 'last') {
          localStorage['moloch-packets'] = this.params.packets || 200;
        }
        localStorage['moloch-line'] = this.params.line;
        localStorage['moloch-gzip'] = this.params.gzip;
        localStorage['moloch-image'] = this.params.image;
      }

      this.packetPromise = SessionsService.getPackets(
        this.session.id,
        this.session.node,
        this.session.cluster,
        this.params
      );

      this.packetPromise.promise
        .then((response) => {
          this.loadingPackets = false;
          this.renderingPackets = true;
          this.packetPromise = undefined;

          // remove all un-whitelisted tokens from the html
          this.packetHtml = sanitizeHtml(response, {
            allowedTags: ['h3', 'h4', 'h5', 'h6', 'a', 'b', 'i', 'strong', 'em', 'div', 'pre', 'span', 'br', 'img'],
            allowedClasses: {
              div: ['row', 'col-md-6', 'offset-md-6', 'sessionsrc', 'sessiondst', 'session-detail-ts', 'alert', 'alert-danger'],
              span: ['pull-right', 'small', 'dstcol', 'srccol', 'fa', 'fa-info-circle', 'fa-lg', 'fa-exclamation-triangle', 'sessionln', 'src-col-tip', 'dst-col-tip'],
              em: ['ts-value'],
              h5: ['text-theme-quaternary'],
              a: ['imagetag', 'file']
            },
            allowedAttributes: {
              div: ['value'],
              img: ['src'],
              a: ['target', 'href']
            }
          });

          setTimeout(() => { // wait until session packets are rendered
            // tooltips for src/dst byte images
            if (!this.$refs.packetContainer) { return; }
            const tss = this.$refs.packetContainer.getElementsByClassName('session-detail-ts');
            for (let i = 0; i < tss.length; ++i) {
              let timeEl = tss[i];
              const value = timeEl.getAttribute('value');
              timeEl = timeEl.getElementsByClassName('ts-value');
              if (!isNaN(value)) { // only parse value if it's a number (ms from 1970)
                const time = this.$options.filters.timezoneDateString(
                  parseInt(value),
                  this.user.settings.timezone,
                  this.user.settings.ms
                );
                timeEl[0].innerHTML = time;
              }
            }

            // tooltips for linked images
            const imgs = this.$refs.packetContainer.getElementsByClassName('imagetag');
            for (let i = 0; i < imgs.length; ++i) {
              const img = imgs[i];
              let href = img.href;
              href = href.replace('body', 'bodypng');

              const tooltip = document.createElement('span');
              tooltip.className = 'img-tip';
              tooltip.innerHTML = `File Bytes:
                <br>
                <img src="${href}">
              `;

              img.appendChild(tooltip);
            }

            // add listeners to fetch the src/dst bytes images on mouse enter
            const srcBytes = this.$refs.packetContainer.getElementsByClassName('srccol');
            if (srcBytes && srcBytes.length) {
              srcBytes[0].addEventListener('mouseenter', this.showSrcBytesImg);
            }

            const dstBytes = this.$refs.packetContainer.getElementsByClassName('dstcol');
            if (dstBytes && dstBytes.length) {
              dstBytes[0].addEventListener('mouseenter', this.showDstBytesImg);
            }

            this.renderingPackets = false;
          });
        })
        .catch((error) => {
          this.loadingPackets = false;
          this.errorPackets = error.text || error;
          this.packetPromise = undefined;
        });
    },
    showSrcBytesImg: function () {
      const url = `api/session/raw/${this.session.node}/${this.session.id}.png?type=src`;
      this.$refs.packetContainer.getElementsByClassName('src-col-tip')[0].innerHTML = `Source Bytes:
        <br>
        <img src="${url}">
        <a class="no-decoration download-bytes" href="${url}" download="${this.session.id}-src.png">
          <span class="fa fa-download"></span>&nbsp;
          Download source bytes image
        </button>
      `;
      this.$refs.packetContainer.getElementsByClassName('srccol')[0].removeEventListener('mouseenter', this.showSrcBytesImg);
    },
    showDstBytesImg: function () {
      const url = `api/session/raw/${this.session.node}/${this.session.id}.png?type=dst`;
      this.$refs.packetContainer.getElementsByClassName('dst-col-tip')[0].innerHTML = `Destination Bytes:
        <br>
        <img src="${url}">
        <a class="no-decoration download-bytes" href="${url}" download="${this.session.id}-dst.png">
          <span class="fa fa-download"></span>&nbsp;
          Download destination bytes image
        </button>
      `;
      this.$refs.packetContainer.getElementsByClassName('dstcol')[0].removeEventListener('mouseenter', this.showDstBytesImg);
    }
  },
  beforeDestroy: function () {
    if (this.packetPromise) {
      this.cancelPacketLoad();
    }

    if (this.$refs.packetContainer) {
      const srcBytes = this.$refs.packetContainer.getElementsByClassName('srccol');
      if (srcBytes && srcBytes.length) {
        srcBytes[0].removeEventListener('mouseenter', this.showSrcBytesImg);
      }

      const dstBytes = this.$refs.packetContainer.getElementsByClassName('dstcol');
      if (dstBytes && dstBytes.length) {
        dstBytes[0].removeEventListener('mouseenter', this.showDstBytesImg);
      }
    }
  }
};
</script>

<style>
.session-detail {
  display: block;
  margin-left: var(--px-md);
  margin-right: var(--px-md);
}

.session-detail h4.sessionDetailMeta {
  padding-top: 10px;
  border-top: 1px solid var(--color-gray);
}

.packet-options {
  border-top: var(--color-gray) 1px solid;
}

.packet-container .tooltip .tooltip-inner {
  max-width: 400px;
}

.packet-container .file {
  display: inline-block;
  margin-bottom: 20px;
}

/* image tooltips */
.packet-container .srccol,
.packet-container .dstcol {
  padding-bottom: 10px;
}

.packet-container .srccol:hover,
.packet-container .dstcol:hover,
.packet-container .imagetag:hover {
  position: relative;
}
.packet-container .srccol .src-col-tip,
.packet-container .dstcol .dst-col-tip,
.packet-container .imagetag .img-tip {
  display: none;
}
.packet-container .srccol:hover .src-col-tip,
.packet-container .dstcol:hover .dst-col-tip,
.packet-container .imagetag:hover .img-tip {
  font-size: 12px;
  display: block;
  position: absolute;
  margin: 10px;
  left: 0;
  top: 20px;
  padding: 4px 6px;
  color: white;
  text-align: center;
  background-color: black;
  border-radius: 5px;
  line-height: 1.2;
  z-index: 2;
}
.packet-container .srccol .src-col-tip:before,
.packet-container .dstcol .dst-col-tip:before,
.packet-container .imagetag .img-tip:before {
  content: '';
  display: block;
  width: 0;
  height: 0;
  position: absolute;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-bottom: 8px solid black;
  top: -7px;
  left: 8px;
}

.packet-container .srccol .src-col-tip .download-bytes,
.packet-container .dstcol .dst-col-tip .download-bytes,
.packet-container .imagetag .img-tip .download-bytes {
  display: block;
  margin-top: 4px;
  font-size: 0.9rem;
}

/* timestamps */
.packet-container .session-detail-ts {
  display: none;
  color: var(--color-foreground-accent);
  font-weight: bold;
  padding: 0 4px;
  border-bottom: 1px solid var(--color-gray);
}
.packet-container.show-ts .session-detail-ts {
  display: block !important;
}

/* src/dst packet visibility */
.packet-container .sessionsrc {
  visibility: visible;
}
.packet-container .sessiondst {
  visibility: visible;
}
.packet-container.hide-src .sessionsrc {
  height: 0px;
  visibility: hidden;
}
.packet-container.hide-dst .sessiondst {
  height: 0px;
  visibility: hidden;
}

/* packets */
.packet-container pre {
  display: block;
  padding: 3px;
  font-size: 12px;
  word-break: break-all;
  word-wrap: break-word;
  border-radius: 0 0 4px 4px;
  margin-top: -1px;
  color: inherit;
  overflow-y: auto;
  white-space: pre-wrap;
}
.packet-container pre .sessionln {
  color: var(--color-foreground-accent, green);
}
/* src/dst packet text colors */
.packet-container .sessiondst {
  color: var(--color-dst, #0000FF) !important;
  max-width: 50vw !important;
  word-wrap: break-word;
}
.packet-container .sessionsrc {
  color: var(--color-src, #CA0404) !important;
  max-width: 50vw !important;
  word-wrap: break-word;
}

/* list values */
.session-detail dt {
  float: left;
  clear: left;
  width: 160px;
  text-align: right;
  margin-right: 6px;
  line-height: 1.7;
  font-weight: 600;
}

.session-detail dd {
  margin-left: 165px;
}

/* more items link */
.session-detail .show-more-items {
  margin-left: 6px;
  margin-right: 6px;
  font-size: 12px;
  text-decoration: none;
}

/* top navigation links */
.session-detail .nav-pills {
  border-bottom: 1px solid var(--color-gray);
  padding-bottom: 4px;
  padding-top: 4px;
}

.session-detail .nav-pills li.nav-item a,
.session-detail .nav-pills div.nav-item button {
  color: var(--color-foreground);
  background-color: transparent;
  border: none;
}

.session-detail .nav-pills li.nav-item a:hover,
.session-detail .nav-pills div.nav-item button:hover {
  color: var(--color-button, #FFF);
}

.session-detail .nav-pills > li.nav-item > a:hover,
.session-detail .nav-pills > li.nav-item.open > a,
.session-detail .nav-pills > li.nav-item.open > a:hover,
.session-detail .nav-pills > div.nav-item > button:hover,
.session-detail .nav-pills > div.nav-item.open > button,
.session-detail .nav-pills > div.nav-item.open > button:hover  {
  background-color: var(--color-quaternary-lighter);
}

/* clickable labels */
.session-detail .clickable-label {
  margin-top: -2px;
}

.session-detail .clickable-label button.btn {
  height: 23px;
  background-color: transparent;
  font-size: 12px;
  font-weight: 600;
  line-height: 21px;
  padding: 0 5px 1px 5px;
  max-width: 160px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.session-detail .clickable-label button.btn:hover {
  color: #333;
  background-color: var(--color-gray);
  border-color: var(--color-gray);
}

.session-detail .clickable-label .dropdown-menu {
  max-height: 280px;
  overflow-y: auto;
}

.session-detail .clickable-label .dropdown-menu .dropdown-item {
  font-size: 12px;
}

/* dl resizing */
.session-detail-grip {
  width: 5px;
  z-index: 4;
  cursor: col-resize;
  position: absolute;
  display: inline-block;
}
dl:hover > .session-detail-grip {
  border-right: 1px dotted var(--color-gray) !important;
}
</style>
