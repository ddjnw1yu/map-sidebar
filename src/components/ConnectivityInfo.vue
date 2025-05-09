<template>
  <div v-if="entry" class="main">
    <div v-if="connectivityEntry.length > 1 && !entryId" class="toggle-button">
      <el-popover
        width="auto"
        trigger="hover"
        :teleported="false"
      >
        <template #reference>
          <el-button
            class="button"
            @click="previous"
            :disabled="this.entryIndex === 0"
          >
            Previous
          </el-button>
        </template>
        <span>{{ previousLabel }}</span>
      </el-popover>
      <el-popover
        width="auto"
        trigger="hover"
        :teleported="false"
      >
        <template #reference>
          <el-button
            class="button"
            @click="next"
            :disabled="this.entryIndex === this.connectivityEntry.length - 1"
          >
            Next
          </el-button>
        </template>
        <span>{{ nextLabel }}</span>
      </el-popover>
    </div>
    <!-- Connectivity Info Title -->
    <div class="connectivity-info-title">
      <div class="title-content">
        <div class="block" v-if="entry.title">
          <div class="title">
            <span @click="onConnectivityClicked({id: entry.featureId[0]})">
              {{ capitalise(entry.title) }}
            </span>
            <template v-if="entry.featuresAlert">
              <el-popover
                width="250"
                trigger="hover"
                :teleported="false"
                popper-class="popover-origin-help"
              >
                <template #reference>
                  <el-icon class="alert"><el-icon-warn-triangle-filled /></el-icon>
                </template>
                <span style="word-break: keep-all">
                  {{ entry.featuresAlert }}
                </span>
              </el-popover>
            </template>
          </div>
          <div v-if="hasProvenanceTaxonomyLabel" class="subtitle">
            {{ provSpeciesDescription }}
          </div>
        </div>
        <div class="block" v-else>
          <div class="title">{{ entry.featureId }}</div>
        </div>
      </div>
      <div class="title-buttons">
        <el-popover
          width="auto"
          trigger="hover"
          :teleported="false"
          popper-class="popover-map-pin"
        >
          <template #reference>
            <el-button class="button-circle secondary" circle @click="showConnectivity">
              <el-icon color="#8300bf">
                <el-icon-location />
              </el-icon>
            </el-button>
          </template>
          <span>
            Show connectivity on map
          </span>
        </el-popover>
        <CopyToClipboard :content="updatedCopyContent" />
        <template v-if="withCloseButton">
          <el-popover
            width="auto"
            trigger="hover"
            :teleported="false"
            popper-class="popover-map-pin"
          >
            <template #reference>
              <el-button class="button-circle" circle @click="closeConnectivity">
                <el-icon color="white">
                  <el-icon-close />
                </el-icon>
              </el-button>
            </template>
            <span>Close</span>
          </el-popover>
        </template>
      </div>
    </div>

    <div
      class="content-container population-display"
      :class="dualConnectionSource ? 'population-display-toolbar' : ''"
    >
      <div class="block attribute-title-container">
        <span class="attribute-title">Population Display</span>
        <el-popover
          v-if="activeView === 'listView'"
          width="250"
          trigger="hover"
          :teleported="false"
          popper-class="popover-origin-help"
        >
          <template #reference>
            <el-icon class="info"><el-icon-warning /></el-icon>
          </template>
          <span style="word-break: keep-all">
            This list is ordered alphabetically,
            switch to graph view for path details.
          </span>
        </el-popover>
      </div>
      <div class="block buttons-row">
        <div v-if="dualConnectionSource">
          <span>Connectivity from:</span>
          <el-radio-group v-model="connectivitySource" @change="onConnectivitySourceChange">
            <el-radio value="map">Map</el-radio>
            <el-radio value="sckan">SCKAN</el-radio>
          </el-radio-group>
        </div>
        <div>
          <el-button
            :class="activeView === 'listView' ? 'button' : 'el-button-secondary'"
            @click="switchConnectivityView('listView')"
          >
            List view
          </el-button>
          <el-button
            :class="activeView === 'graphView' ? 'button' : 'el-button-secondary'"
            @click="switchConnectivityView('graphView')"
          >
            Graph view
          </el-button>
        </div>
      </div>
    </div>

    <div class="content-container content-container-connectivity" v-show="activeView === 'listView'">
      <connectivity-list
        v-loading="connectivityLoading"
        :key="`${connectivityKey}list`"
        :entry="entry"
        :origins="origins"
        :components="components"
        :destinations="destinations"
        :originsWithDatasets="originsWithDatasets"
        :componentsWithDatasets="componentsWithDatasets"
        :destinationsWithDatasets="destinationsWithDatasets"
        :availableAnatomyFacets="availableAnatomyFacets"
        :connectivityError="connectivityError"
        @connectivity-hovered="onConnectivityHovered"
        @connectivity-clicked="onConnectivityClicked"
        @connectivity-action-click="onConnectivityActionClick"
      />
    </div>

    <div class="content-container content-container-connectivity" v-show="activeView === 'graphView'">
      <template v-if="graphViewLoaded">
        <connectivity-graph
          v-loading="connectivityLoading"
          :key="`${connectivityKey}graph`"
          :entry="entry.featureId[0]"
          :mapServer="flatmapApi"
          :sckanVersion="sckanVersion"
          :connectivityFromMap="connectivityFromMap"
          @tap-node="onTapNode"
          ref="connectivityGraphRef"
        />
      </template>
    </div>

    <div class="content-container content-container-references" v-if="resources.length">
      <ExternalResourceCard
        :resources="resources"
        @references-loaded="onReferencesLoaded"
        @show-reference-connectivities="onShowReferenceConnectivities"
      />
    </div>
  </div>
</template>

<script>
  /* eslint-disable no-alert, no-console */
import {
  Warning as ElIconWarning,
  Location as ElIconLocation,
  Search as ElIconSearch,
} from '@element-plus/icons-vue'
import {
  ElButton as Button,
  ElContainer as Container,
  ElIcon as Icon,
} from 'element-plus'

import EventBus from './EventBus.js'
import {
  CopyToClipboard,
  ConnectivityGraph,
  ConnectivityList,
  ExternalResourceCard,
} from '@abi-software/map-utilities';
import '@abi-software/map-utilities/dist/style.css';

const titleCase = (str) => {
  return str.replace(/\w\S*/g, (t) => {
    return t.charAt(0).toUpperCase() + t.substr(1).toLowerCase()
  })
}

const capitalise = function (str) {
  if (str) return str.charAt(0).toUpperCase() + str.slice(1)
  return ''
}

const ERROR_TIMEOUT = 3000; // 3 seconds

export default {
  name: 'ConnectivityInfo',
  components: {
    Button,
    Container,
    Icon,
    ElIconWarning,
    ElIconLocation,
    ElIconSearch,
    ExternalResourceCard,
    CopyToClipboard,
    ConnectivityGraph,
    ConnectivityList,
  },
  props: {
    connectivityEntry: {
      type: Array,
      default: [],
    },
    entryId: {
      type: String,
      default: "",
    },
    envVars: {
      type: Object,
      default: () => {},
    },
    availableAnatomyFacets: {
      type: Array,
      default: () => [],
    },
    withCloseButton: {
      type: Boolean,
      default: false,
    },
  },
  data: function () {
    return {
      entryIndex: 0,
      updatedCopyContent: '',
      activeView: 'listView',
      timeoutID: undefined,
      connectivityLoading: false,
      dualConnectionSource: false,
      connectivitySource: 'sckan',
      connectivityError: null,
      graphViewLoaded: false,
      connectivityFromMap: null,
    };
  },
  computed: {
    entry: function () {
      if (this.entryId) {
        return this.connectivityEntry.filter((entry) => {
          return entry.featureId[0] === this.entryId;
        })[this.entryIndex];
      }
      return this.connectivityEntry[this.entryIndex];
    },
    previousLabel: function () {
      if (this.entryIndex === 0) {
        return "This is the first item. Click 'Next' to see more information.";
      }
      return this.connectivityEntry[this.entryIndex - 1].title;
    },
    nextLabel: function () {
      if (this.entryIndex === this.connectivityEntry.length - 1) {
        return "This is the last item. Click 'Previous' to see more information.";
      }
      return this.connectivityEntry[this.entryIndex + 1].title;
    },
    hasProvenanceTaxonomyLabel: function () {
      return (
        this.entry.provenanceTaxonomyLabel &&
        this.entry.provenanceTaxonomyLabel.length > 0
      );
    },
    provSpeciesDescription: function () {
      let text = "Studied in";
      this.entry.provenanceTaxonomyLabel.forEach((label) => {
        text += ` ${label},`;
      });
      text = text.slice(0, -1); // remove last comma
      text += " species";
      return text;
    },
    connectivityKey: function () {
      return this.entry.featureId[0] + this.entry.connectivitySource;
    },
    origins: function () {
      return this.entry.origins;
    },
    components: function () {
      return this.entry.components;
    },
    destinations: function () {
      return this.entry.destinations;
    },
    originsWithDatasets: function () {
      return this.entry.originsWithDatasets;
    },
    componentsWithDatasets: function () {
      return this.entry.componentsWithDatasets;
    },
    destinationsWithDatasets: function () {
      return this.entry.destinationsWithDatasets;
    },
    resources: function () {
      return this.entry.hyperlinks;
    },
    sckanVersion: function () {
      return this.entry.knowledgeSource;
    },
    flatmapApi: function () {
      return this.envVars.FLATMAPAPI_LOCATION;
    },
  },
  watch: {
    entry: {
      deep: true,
      immediate: true,
      handler: function (newVal, oldVal) {
        if (newVal !== oldVal) {
          this.connectivityLoading = true;
          this.activeView = localStorage.getItem('connectivity-active-view') || this.activeView;
          if (this.activeView === 'graphView') {
            this.graphViewLoaded = true;
          }

          this.checkAndUpdateDualConnection();
          this.connectivitySource = this.entry.connectivitySource;
          this.updateGraphConnectivity();
          this.connectivityLoading = false;
          this.$emit('loaded');
        }
      },
    },
  },
  methods: {
    previous: function () {
      if (this.entryIndex !== 0) {
        this.entryIndex = this.entryIndex - 1;
      }
    },
    next: function () {
      if (this.entryIndex !== this.connectivityEntry.length - 1) {
        this.entryIndex = this.entryIndex + 1;
      }
    },
    titleCase: function (title) {
      return titleCase(title)
    },
    capitalise: function (text) {
      return capitalise(text)
    },
    showConnectivity: function () {
      // move the map center to highlighted area
      const featureIds = this.entry.featureId || [];
      // connected to flatmapvuer > moveMap(featureIds) function
      this.$emit('show-connectivity', featureIds);
    },
    switchConnectivityView: function (val) {
      this.activeView = val;
      localStorage.setItem('connectivity-active-view', this.activeView);

      if (val === 'graphView' && !this.graphViewLoaded) {
        // to load the connectivity graph only after the container is in view
        this.$nextTick(() => {
          this.graphViewLoaded = true;
        });
      }
    },
    onTapNode: function (data) {
      // save selected state for list view
      const name = data.map(t => t.label).join(', ');
      this.onConnectivityHovered(name);
    },
    onShowReferenceConnectivities: function (refSource) {
      this.$emit('show-reference-connectivities', refSource);
    },
    onReferencesLoaded: function (references) {
      this.updatedCopyContent = this.getUpdateCopyContent(references);
    },
    getUpdateCopyContent: function (references) {
      if (!this.entry) {
        return '';
      }

      const contentArray = [];

      // Use <div> instead of <h1>..<h6> or <p>
      // to avoid default formatting on font size and margin

      // Title
      let title = this.entry.title;
      let featureId = this.entry.featureId;
      const titleContent = [];

      if (title) {
        titleContent.push(`<strong>${capitalise(this.entry.title)}</strong>`);
      }

      if (featureId?.length) {
        if (typeof featureId === 'object') {
          titleContent.push(`(${featureId[0]})`);
        } else {
          titleContent.push(`(${featureId})`);
        }
      }

      contentArray.push(`<div>${titleContent.join(' ')}</div>`);

      // Description
      if (this.entry.provenanceTaxonomyLabel?.length) {
        contentArray.push(`<div>${this.provSpeciesDescription}</div>`);
      }

      // entry.paths
      if (this.entry.paths) {
        contentArray.push(`<div>${this.entry.paths}</div>`);
      }

      function transformData(title, items, itemsWithDatasets = []) {
        let contentString = `<div><strong>${title}</strong></div>`;
        const transformedItems = [];
        items.forEach((item) => {
          let itemNames = [];
          item.split(',').forEach((name) => {
            const match = itemsWithDatasets.find((a) => a.name === name.trim());
            if (match) {
              itemNames.push(`${capitalise(name)} (${match.id})`);
            } else {
              itemNames.push(`${capitalise(name)}`);
            }
          });
          transformedItems.push(itemNames.join(','));
        });
        const contentList = transformedItems
          .map((item) => `<li>${item}</li>`)
          .join('\n');
        contentString += '\n';
        contentString += `<ul>${contentList}</ul>`;
        return contentString;
      }

      // Origins
      if (this.origins?.length) {
        const title = 'Origin';
        const origins = this.origins;
        const originsWithDatasets = this.originsWithDatasets;
        const transformedOrigins = transformData(title, origins, originsWithDatasets);
        contentArray.push(transformedOrigins);
      }

      // Components
      if (this.components?.length) {
        const title = 'Components';
        const components = this.components;
        const componentsWithDatasets = this.componentsWithDatasets;
        const transformedComponents = transformData(title, components, componentsWithDatasets);
        contentArray.push(transformedComponents);
      }

      // Destination
      if (this.destinations?.length) {
        const title = 'Destination';
        const destinations = this.destinations;
        const destinationsWithDatasets = this.destinationsWithDatasets;
        const transformedDestinations = transformData(title, destinations, destinationsWithDatasets);
        contentArray.push(transformedDestinations);
      }

      // References
      if (references) {
        let contentString = `<div><strong>References</strong></div>`;
        contentString += '\n';
        const contentList = references.list
          .map((item) => `<li>${item}</li>`)
          .join('\n');
        contentString += `<ul>${contentList}</ul>`;
        contentArray.push(contentString);
      }

      return contentArray.join('\n\n<br>');
    },
    getConnectivityDatasets: function (label) {
      const allWithDatasets = [
        ...this.componentsWithDatasets,
        ...this.destinationsWithDatasets,
        ...this.originsWithDatasets,
      ];
      const names = label.split(','); // some features have more than one value
      let data = [];
      names.forEach((n) => {
        const foundData = allWithDatasets.find((a) =>
          a.name.toLowerCase().trim() === n.toLowerCase().trim()
        );
        if (foundData) {
          data.push({
            id: foundData.id,
            label: foundData.name
          });
        }
      });
      return data
    },
    onConnectivityHovered: function (label) {
      const payload = {
        connectivityInfo: this.entry,
        data: label ? this.getConnectivityDatasets(label) : [],
      };
      // type: to show error only for click event
      this.$emit('connectivity-hovered', payload);
    },
    onConnectivityClicked: function (data) {
      let payload = {
        query: data.id,
        filter: [],
        data: data.label ? this.getConnectivityDatasets(data.label) : [],
      };
      this.$emit('connectivity-clicked', payload);
    },
    getErrorConnectivities: function (errorData) {
      const errorDataToEmit = [...new Set(errorData)];
      let errorConnectivities = '';

      errorDataToEmit.forEach((connectivity, i) => {
        const { label } = connectivity;
        errorConnectivities += (i === 0) ? capitalise(label) : label;

        if (errorDataToEmit.length > 1) {
          if ((i + 2) === errorDataToEmit.length) {
            errorConnectivities += ' and ';
          } else if ((i + 1) < errorDataToEmit.length) {
            errorConnectivities += ', ';
          }
        }
      });

      return errorConnectivities;
    },
    /**
     * Function to show error message.
     * `errorInfo` includes `errorData` array (optional) for error connectivities
     * and `errorMessage` for error message.
     * @arg `errorInfo`
     */
    getConnectivityError: function (errorInfo) {
      const { errorData, errorMessage } = errorInfo;
      const errorConnectivities = this.getErrorConnectivities(errorData);

      return {
        errorConnectivities,
        errorMessage,
      };
    },
    pushConnectivityError: function (errorInfo) {
      const connectivityError = this.getConnectivityError(errorInfo);
      const connectivityGraphRef = this.$refs.connectivityGraphRef;

      // error for graph view
      if (connectivityGraphRef) {
        connectivityGraphRef.showErrorMessage(connectivityError);
      }

      // error for list view
      this.connectivityError = {...connectivityError};

      if (this.timeoutID) {
        clearTimeout(this.timeoutID);
      }

      this.timeoutID = setTimeout(() => {
        this.connectivityError = null;
      }, ERROR_TIMEOUT);
    },
    onConnectivitySourceChange: function (connectivitySource) {
      const { featureId } = this.entry;

      this.connectivityLoading = true;

      if (this.activeView !== 'graphView') {
        this.graphViewLoaded = false;
      }

      this.checkAndUpdateDualConnection();
      this.updateGraphConnectivity();

      EventBus.emit('connectivity-source-change', {
        featureId: featureId,
        connectivitySource: connectivitySource,
      });
    },
    updateGraphConnectivity: function () {
      if (this.connectivitySource === "map") {
        this.getConnectionsFromMap().then((response) => {
          this.connectivityFromMap = response;
          this.connectivityLoading = false;
        });
      } else {
        this.connectivityFromMap = null;
        this.connectivityLoading = false;
      }
    },
    getConnectionsFromMap: async function () {
      const url =
        this.flatmapApi +
        `flatmap/${this.entry.mapuuid}/connectivity/${this.entry.featureId[0]}`;

      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error(`Response status: ${response.status}`);
        }

        return await response.json();
      } catch (error) {
        throw new Error(error);
      }
    },
    onConnectivityActionClick: function (data) {
      EventBus.emit('onConnectivityActionClick', data);
    },
    closeConnectivity: function () {
      this.$emit('close-connectivity');
    },
    checkAndUpdateDualConnection: async function () {
      const response = await this.getConnectionsFromMap()

      if (response?.connectivity?.length) {
        this.dualConnectionSource = true;
      } else {
        this.dualConnectionSource = false;
        this.connectivitySource = 'sckan';
      }
    },
  },
  mounted: function () {
    this.updatedCopyContent = this.getUpdateCopyContent();

    EventBus.on('connectivity-graph-error', (errorInfo) => {
      this.pushConnectivityError(errorInfo);
    });
  },
}
</script>

<style lang="scss" scoped>
.connectivity-info-title {
  padding: 0;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  gap: 1rem;

  .title-content {
    flex: 1 0 0%;
    max-width: 85%;
  }
}

.toggle-button {
  display: flex;
  justify-content: space-between;

  .is-disabled {
    color: #fff !important;
    background-color: #ac76c5 !important;
    border: 1px solid #ac76c5 !important;
  }
}

.title {
  text-align: left;
  // width: 16em;
  line-height: 1.3em !important;
  font-size: 18px;
  // font-family: Helvetica;
  font-weight: bold;
  padding-bottom: 8px;
  color: $app-primary-color;

  span:hover {
    cursor: pointer;
  }
}

.block + .block {
  margin-top: 0.5em;
}

.button-circle {
  margin: 0;
  width: 24px !important;
  height: 24px !important;

  &,
  &:hover,
  &:focus,
  &:active {
    background-color: $app-primary-color;
    border-color: $app-primary-color;
  }

  &.secondary {
    background-color: white;
  }
}

.alert {
  color: #8300bf;
  margin-left: 5px;
  vertical-align: text-bottom;

  &,
  > svg {
    width: 1.25rem;
    height: 1.25rem;
  }
}

.hide {
  color: $app-primary-color;
  cursor: pointer;
  margin-right: 6px;
  margin-top: 3px;
}

.slide-fade-enter-active {
  transition: opacity 0.5s, transform 0.5s;
}
.slide-fade-leave-active {
  transition: opacity 0.2s, transform 0.2s;
}
.slide-fade-enter, .slide-fade-leave-to /* .slide-fade-leave-active in <2.1.8 */ {
  opacity: 0;
  transform: translateY(-8px);
}

.main {
  font-size: 14px;
  text-align: left;
  line-height: 1.5em;
  font-family: Asap, sans-serif, Helvetica;
  font-weight: 400;
  /* outline: thin red solid; */
  overflow-y: auto;
  scrollbar-width: thin;
  min-width: 16rem;
  background-color: #f7faff;
  height: 100%;
  border-left: 1px solid var(--el-border-color);
  border-top: 1px solid var(--el-border-color);
  display: flex;
  flex-direction: column;
  gap: 1.75rem;
  padding: 1rem;
}

.info {
  color: #8300bf;
  transform: rotate(180deg);
  margin-left: 8px;
}

.attribute-title-container {
  margin-bottom: 0.5em;
}

.attribute-title {
  font-size: 16px;
  font-weight: 600;
  /* font-weight: bold; */
  text-transform: uppercase;
}

.main {
  .el-button.is-round {
    border-radius: 4px;
    padding: 9px 20px 10px 20px;
    display: flex;
    height: 36px;
  }
}

.button {
  margin-left: 0px !important;
  margin-top: 0px !important;
  font-size: 14px !important;
  background-color: $app-primary-color;
  color: #fff;

  &:hover {
    color: #fff !important;
    background-color: #ac76c5 !important;
    border: 1px solid #ac76c5 !important;
  }

  & + .button {
    margin-top: 10px !important;
  }
}

.el-button-secondary {
  border-color: transparent !important;
  background-color: transparent !important;
  color: inherit !important;

  &:hover {
    color: $app-primary-color !important;
    border-color: $app-primary-color !important;
    background-color: #f9f2fc !important;
  }
}

.buttons-row {
  text-align: right;

  .button {
    cursor: default;
    border-color: transparent;

    &:hover {
      background-color: $app-primary-color !important;
      border-color: transparent !important;
    }
  }

  .el-button + .el-button {
    margin-top: 0 !important;
    margin-left: 10px !important;
  }

  > div:first-child {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
}

.neuron-connection-button {
  display: flex;
  flex: 1 1 auto !important;
  flex-direction: row !important;
  align-items: center;
  justify-content: space-between;

  .el-button + .el-button {
    margin-top: 0 !important;
    margin-left: 10px !important;
  }
}

.population-display {
  display: flex;
  flex: 1 1 auto !important;
  flex-direction: row !important;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid $app-primary-color;
  padding-bottom: 0.5rem !important;

  &.population-display-toolbar {
    flex-direction: column !important;
    align-items: start;

    .buttons-row {
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: space-between;
      width: 100%;
    }
  }

  .el-radio {
    margin-right: 1rem;
  }
}

.tooltip-container {
  &::after,
  &::before {
    content: '';
    display: block;
    position: absolute;
    width: 0;
    height: 0;
    border-style: solid;
    flex-shrink: 0;
  }
  .tooltip {
    &::after {
      display: none;
    }
    &::before {
      display: none;
    }
  }
}

.maplibregl-popup-anchor-bottom {
  .tooltip-container {
    &::after,
    &::before {
      top: 100%;
      border-width: 12px;
    }
    &::after {
      margin-top: -1px;
      border-color: rgb(255, 255, 255) transparent transparent transparent;
    }
    &::before {
      margin: 0 auto;
      border-color: $app-primary-color transparent transparent transparent;
    }
  }
}

.maplibregl-popup-anchor-top {
  .tooltip-container {
    &::after,
    &::before {
      top: -24px;
      border-width: 12px;
    }
    &::after {
      margin-top: 1px;
      border-color: transparent transparent rgb(255, 255, 255) transparent;
    }
    &::before {
      margin: 0 auto;
      border-color: transparent transparent $app-primary-color transparent;
    }
  }
}

.content-container {
  flex: 1 1 100%;
  padding: 0;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  gap: 1rem;

  > div,
  > .block + .block {
    margin: 0;
  }
}

/* Fix for chrome bug where under triangle pops up above one on top of it  */
.selector:not(*:root),
.tooltip-container::after {
  top: 99.4%;
}

.title-buttons {
  display: flex;
  flex: 1 0 0%;
  max-width: 15%;
  flex-direction: row;
  justify-content: end;
  gap: 0.5rem;

  :deep(.copy-clipboard-button) {
    &,
    &:hover,
    &:focus {
      border-color: $app-primary-color !important;
      border-radius: 50%;
    }
  }

  .el-button + .el-button {
    margin-left: 0 !important;
  }
}

:deep(.el-popper.popover-map-pin) {
  padding: 4px 10px;
  min-width: max-content;
  font-family: Asap;
  font-size: 12px;
  line-height: inherit;
  color: inherit;
  background: #f3ecf6 !important;
  border: 1px solid $app-primary-color;

  & .el-popper__arrow::before {
    border: 1px solid;
    border-color: $app-primary-color;
    background: #f3ecf6;
  }
}

.content-container-connectivity {
  position: relative;

  &:not([style*="display: none"]) ~ .content-container-references {
    margin-top: -1.25rem;
  }
}
</style>
