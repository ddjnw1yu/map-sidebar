<template>
  <div ref="container">
    <div v-if="!drawerOpen" @click="toggleDrawer" class="open-tab">
      <el-icon><el-icon-arrow-left /></el-icon>
    </div>
    <el-drawer
      class="side-bar my-drawer"
      v-model="drawerOpen"
      :teleported="false"
      :modal-append-to-body="false"
      size="584"
      :with-header="false"
      :wrapperClosable="false"
      :modal="false"
      modal-class="sidebar-body"
      :z-index="10"
      :lock-scroll="false"
    >
      <div class="box-card">
        <div v-if="drawerOpen" @click="close" class="close-tab">
          <el-icon><el-icon-arrow-right /></el-icon>
        </div>
        <div class="sidebar-container">
          <Tabs
            v-if="activeTabs.length > 1"
            :tabTitles="activeTabs"
            :activeId="activeTabId"
            @titleClicked="tabClicked"
            @tab-close="tabClose"
          />
          <template v-for="tab in tabs" key="tab.id">
            <!-- Connectivity Info -->
            <template v-if="tab.type === 'connectivity' && connectivityInfo">
              <connectivity-info
                :entry="connectivityInfo"
                :availableAnatomyFacets="availableAnatomyFacets"
                v-if="tab.id === activeTabId"
                :envVars="envVars"
                :ref="'connectivityTab_' + tab.id"
                @show-connectivity="showConnectivity"
                @show-reference-connectivities="onShowReferenceConnectivities"
                @connectivity-component-click="onConnectivityComponentClick"
              />
            </template>
            <template v-else-if="tab.type === 'annotation'">
              <annotation-tool
                :ref="'annotationTab_' + tab.id"
                v-show="tab.id === activeTabId"
                :annotationEntry="annotationEntry"
                :createData="createData"
                @annotation="$emit('annotation-submitted', $event)"
                @confirm-create="$emit('confirm-create', $event)"
                @cancel-create="$emit('cancel-create')"
                @confirm-delete="$emit('confirm-delete', $event)"
              />
            </template>
            <template v-else>
              <SidebarContent
                class="sidebar-content-container"
                v-show="tab.id === activeTabId"
                :contextCardEntry="tab.contextCard"
                :envVars="envVars"
                :ref="'searchTab_' + tab.id"
                :hyperlinks="hyperlinks"
                @search-changed="searchChanged(tab.id, $event)"
                @hover-changed="hoverChanged($event)"
              />
            </template>
          </template>
        </div>
      </div>
    </el-drawer>
  </div>
</template>

<script>
import {
  ArrowLeft as ElIconArrowLeft,
  ArrowRight as ElIconArrowRight,
} from '@element-plus/icons-vue'
/* eslint-disable no-alert, no-console */
import { ElDrawer as Drawer, ElIcon as Icon } from 'element-plus'
import SidebarContent from './SidebarContent.vue'
import EventBus from './EventBus.js'
import Tabs from './Tabs.vue'
import AnnotationTool from './AnnotationTool.vue'
import ConnectivityInfo from './ConnectivityInfo.vue'

/**
 * Aims to provide a sidebar for searching capability for SPARC portal.
 */
export default {
  components: {
    SidebarContent,
    Tabs,
    ElIconArrowLeft,
    ElIconArrowRight,
    Drawer,
    Icon,
    ConnectivityInfo,
    AnnotationTool,
  },
  name: 'SideBar',
  props: {
    /**
     * The option to show side bar.
     */
    visible: {
      type: Boolean,
      default: false,
    },
    /**
     * The environment variables object with
     * `API_LOCATION`, `ALGOLIA_KEY`, `ALGOLIA_ID`,
     * `ALGOLIA_INDEX`, `PENNSIEVE_API_LOCATION`, `BL_SERVER_URL`,
     * `NL_LINK_PREFIX`, `ROOT_URL`
     */
    envVars: {
      type: Object,
      default: () => {},
    },
    /**
     * The array of objects to show multiple sidebar contents.
     */
    tabs: {
      type: Array,
      default: () => [
        { id: 1, title: 'Search', type: 'search' },
        { id: 2, title: 'Connectivity', type: 'connectivity' },
        { id: 3, title: 'Annotation', type: 'annotation' }
      ],
    },
    /**
     * The active tab id for default tab.
     */
    activeTabId: {
      type: Number,
      default: 1,
    },
    /**
     * The option to show or hide sidebar on page load.
     */
    openAtStart: {
      type: Boolean,
      default: false,
    },
    /**
     * The connectivity info data to show in sidebar.
     */
    connectivityInfo: {
      type: Object,
      default: null,
    },
    /**
     * The annotation data to show in sidebar.
     */
    annotationEntry: {
      type: Object,
      default: null,
    },
    createData: {
      type: Object,
      default: {
        toBeConfirmed: false,
        points: [],
        shape: "",
        x: 0,
        y: 0,
      },
    },
    hyperlinks: {
      type: Object,
      default: {},
    }
  },
  data: function () {
    return {
      drawerOpen: false,
      initFilters: { filter: [], searchInput: '' },
      availableAnatomyFacets: []
    }
  },
  methods: {
    /**
     * This event is emitted when the mouse hover are changed.
     * @arg data
     */
    hoverChanged: function (data) {
      this.$emit('hover-changed', data)
    },
    /**
     * This event is emitted when the show connectivity button is clicked.
     * @arg featureIds
     */
    showConnectivity: function (featureIds) {
      this.$emit('show-connectivity', featureIds);
    },
    /**
     * This event is emitted when the show related connectivities button in reference is clicked.
     * @param refSource
     */
    onShowReferenceConnectivities: function (refSource) {
      this.$emit('show-reference-connectivities', refSource);
    },
    /**
     * This function is triggered after a connectivity component is clicked.
     * @arg data
     */
    onConnectivityComponentClick: function (data) {
      this.$emit('connectivity-component-click', data);
    },
    /**
     * This event is emitted when the search filters are changed.
     * @arg `obj` {data, id}
     */
    searchChanged: function (id, data) {
      this.$emit('search-changed', { ...data, id: id })
    },
    /**
     * The function to close sidebar.
     * @public
     */
    close: function () {
      this.drawerOpen = false
    },
    /**
     * The function to toggle (open and close) sidebar.
     * @public
     */
    toggleDrawer: function () {
      this.drawerOpen = !this.drawerOpen
    },
    openSearch: function (facets, query) {
      this.initFilters.filter = facets;
      this.initFilters.searchInput = query;
      this.drawerOpen = true;
      // Because refs are in v-for, nextTick is needed here
      this.$nextTick(() => {
        const searchTabRef = this.getSearchTabRefById(1);
        searchTabRef.openSearch(facets, query);
      })
    },
    /**
     * Get the tab object by tab id and type.
     * If not found, return the first available tab.
     */
    getTabByIdAndType: function (id, type) {
      const tabId = id || this.activeTabId;
      const tabType = type || 'search'; // default to search tab
      const tabObj = this.activeTabs.find((tab) => tab.id === tabId && tab.type === tabType);
      const firstAvailableTab = this.activeTabs[0];
      return tabObj || firstAvailableTab;
    },
    /**
     * Get the ref id of the tab by id and type.
     */
    getTabRefId: function (id, type) {
      let refIdPrefix = 'searchTab_'; // default to search tab
      if (type === 'connectivity') {
        refIdPrefix = 'connectivityTab_';
      } else if (type === 'annotation') {
        refIdPrefix = 'annotationTab_';
      }
      const tabObj = this.getTabByIdAndType(id, type);
      const tabRefId = refIdPrefix + tabObj.id;
      return tabRefId;
    },
    getSearchTabRefById: function (id) {
      const searchTabId = id || 1; // to use id when there are multiple search tabs
      const searchTabRefId = this.getTabRefId(searchTabId, 'search');
      return this.$refs[searchTabRefId][0];
    },
    /**
     * The function to add filters to sidebar search.
     *
     * @param {Object} filter
     * @public
     */
    addFilter: function (filter) {
      this.drawerOpen = true
      filter.AND = true // When we add a filter external, it is currently only with an AND boolean

      // Because refs are in v-for, nextTick is needed here
      this.$nextTick(() => {
        const searchTabRef = this.getSearchTabRefById(1);
        searchTabRef.addFilter(filter)
      })
    },
    openNeuronSearch: function (neuron) {
      this.drawerOpen = true
      // Because refs are in v-for, nextTick is needed here
      this.$nextTick(() => {
        const searchTabRef = this.getSearchTabRefById(1);
        searchTabRef.openSearch(
          '',
          undefined,
          'scicrunch-query-string/',
          { field: '*organ.curie', curie: neuron }
        )
      })
    },
    getAlgoliaFacets: async function () {
      const searchTabRef = this.getSearchTabRefById(1);
      return await searchTabRef.getAlgoliaFacets()
    },
    setDrawerOpen: function (value = true) {
      this.drawerOpen = value
    },
    /**
     * The function to emit 'tabClicked' event with tab's `id` and tab's `type`
     * when user clicks the sidebar tab.
     * @param {Object} {id, type}
     * @public
     */
    tabClicked: function ({id, type}) {
      /**
       * This event is emitted when user click sidebar's tab.
       * @arg {Object} {id, type}
       */
      this.$emit('tabClicked', {id, type});
    },
    tabClose: function (id) {
      this.$emit('tab-close', id);
    },
    /**
     * To receive error message for connectivity graph
     * @param {String} errorMessage
     */
    updateConnectivityGraphError: function (errorInfo) {
      EventBus.emit('connectivity-graph-error', errorInfo);
    },
  },
  computed: {
    // This should respect the information provided by the property
    activeTabs: function() {
      const tabs = []
      this.tabs.forEach((tab) => {
        if (tab.type === "search") {
          tabs.push(tab)
        } else if (tab.type === "connectivity") {
          if (this.connectivityInfo) {
            tabs.push(tab);
          }
        } else if (tab.type === "annotation") {
          if (this.annotationEntry && Object.keys(this.annotationEntry).length > 0) {
            tabs.push(tab);
          }
        }
      })
      return tabs;
    },
  },
  created: function () {
    this.drawerOpen = this.openAtStart
  },
  mounted: function () {
    EventBus.on('PopoverActionClick', (payLoad) => {
      /**
       * This event is emitted when the image is clicked on or the button below the image is clicked on.
       * @arg payLoad
       */
      this.$emit('actionClick', payLoad)
    })
    EventBus.on('number-of-datasets-for-anatomies', (payLoad) => {
      /**
       * This emits a object with keys as anatomy and values as number of datasets for that anatomy.
       * @arg payload
       */
      this.$emit('number-of-datasets-for-anatomies', payLoad)
    })
    EventBus.on('anatomy-in-datasets', (payLoad) => {
       /**
       * This emits a lis of datasets, with the anatomy for each one. Used by flatmap for markers
       * @arg payload
       */
      this.$emit('anatomy-in-datasets', payLoad)
    })

    EventBus.on('contextUpdate', (payLoad) => {
      /**
       * This event is emitted when the context card is updated.
       * Example, context card update on first load.
       * @arg payload
       */
      this.$emit('contextUpdate', payLoad)
    })
    EventBus.on('datalink-clicked', (payLoad) => {
      /**
       * This event is emitted
       * when the dataset button or dataset image thumbnail
       * from the gallery component is clicked.
       * @arg payload
       */
      this.$emit('datalink-clicked', payLoad);
    })
    EventBus.on('onConnectivityActionClick', (payLoad) => {
      // switch to search tab with tab id: 1
      this.tabClicked({id: 1, type: 'search'});
      this.$emit('actionClick', payLoad);
    })

    // Get available anatomy facets for the connectivity info
    EventBus.on('available-facets', (payLoad) => {
        this.availableAnatomyFacets = payLoad.find((facet) => facet.label === 'Anatomical Structure').children
    })

    // Get available anatomy facets for the connectivity info
    EventBus.on('pmr-action-click', (payLoad) => {
      this.$emit('actionClick', payLoad);
    })

  },
}
</script>

<style lang="scss" scoped>
.box-card {
  flex: 3;
  height: 100%;
  overflow: hidden;
  pointer-events: auto;
}

.side-bar {
  position: relative;
  height: 100%;
  pointer-events: none;
  width:600px;
  float: right;
}

:deep(.sidebar-body) {
  position: absolute !important;
}

.side-bar :deep(.el-drawer:focus) {
  outline: none;
}

.sidebar-container {
  height: 100%;
  flex-flow: column;
  display: flex;
  background-color: white;
  box-shadow: var(--el-box-shadow-light);
  border-radius: var(--el-card-border-radius);
}

.open-tab {
  width: 20px;
  height: 40px;
  z-index: 8;
  position: absolute;
  top: calc(50vh - 80px);
  right: 0px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.06);
  border: solid 1px $app-primary-color;
  background-color: #f9f2fc;
  text-align: center;
  vertical-align: middle;
  cursor: pointer;
  pointer-events: auto;
}

.el-icon svg {
  font-weight: 600;
  margin-top: 24px;
  color: $app-primary-color;
  cursor: pointer;
  pointer-events: auto;
  transform: scaleY(2);
  text-align: center;
  vertical-align: middle;
}

.close-tab {
  float: left;
  flex: 1;
  width: 20px;
  height: 40px;
  z-index: 8;
  margin-top: calc(50vh - 80px);
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.06);
  border: solid 1px $app-primary-color;
  background-color: #f9f2fc;
  text-align: center;
  vertical-align: middle;
  cursor: pointer;
  pointer-events: auto;
}

:deep(.my-drawer) {
  background: rgba(0, 0, 0, 0);
  box-shadow: none;
}

:deep(.my-drawer .el-drawer__body) {
  height: 100%;
  padding: 0;
}

.sidebar-content-container {
  flex: 1 1 auto;

  .tab-container ~ & {
    border-radius: 0;
    border: 0 none;
    position: relative;
  }
}
</style>

<style lang="scss">
.side-bar {
  --el-color-primary: #8300BF;
  --el-color-primary-light-7: #DAB3EC;
  --el-color-primary-light-8: #e6ccf2;
  --el-color-primary-light-9: #f3e6f9;
  --el-color-primary-light-3: #f3e6f9;
  --el-color-primary-dark-2: #7600AC;
}
.el-button--primary {
  --el-button-hover-text-color: var(--el-color-primary);
  --el-button-hover-link-text-color: var(--el-color-primary-light-5);
  --el-button-hover-bg-color: var(--el-color-primary-light-3);
  --el-button-hover-border-color: var(--el-color-primary-light-3);
}
</style>
