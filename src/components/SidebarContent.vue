<template>
  <el-card :body-style="bodyStyle" class="content-card">
    <template #header>
      <div class="header">
        <el-input
          class="search-input"
          placeholder="Search"
          v-model="searchInput"
          @keyup="searchEvent"
          clearable
          @clear="clearSearchClicked"
        ></el-input>
        <el-button
          type="primary"
          class="button"
          @click="searchEvent"
          size="large"
        >
          Search
        </el-button>
      </div>
    </template>
    <SearchFilters
      class="filters"
      ref="filtersRef"
      :entry="filterEntry"
      :envVars="envVars"
      @filterResults="filterUpdate"
      @numberPerPage="numberPerPageUpdate"
      @loading="filtersLoading"
      @cascaderReady="cascaderReady"
    ></SearchFilters>
    <SearchHistory
      ref="searchHistory"
      @search="searchHistorySearch"
    ></SearchHistory>
      pmr hits: {{ pmrNumberOfHits }}
    <div class="content scrollbar" v-loading="loadingCards" ref="content">
      <div class="error-feedback" v-if="results.length === 0 && !loadingCards">
        No results found - Please change your search / filter criteria.
      </div>
      <div v-for="(result, i) in results" :key="result.doi || i" class="step-item">
        <DatasetCard
          v-if="result.dataSource === 'SPARC'"
          class="dataset-card"
          :entry="result"
          :envVars="envVars"
          @mouseenter="hoverChanged(result)"
          @mouseleave="hoverChanged(undefined)"
        ></DatasetCard>
        <PMRDatasetCard
          v-else-if="result.dataSource === 'PMR'"
          class="dataset-card"
          :entry="result"
          :envVars="envVars"
          :hyperlinks="hyperlinks"
          @mouseenter="hoverChanged(result)"
          @mouseleave="hoverChanged(undefined)"
        ></PMRDatasetCard>
      </div>
      <el-pagination
        class="pagination"
        v-model:current-page="page"
        hide-on-single-page
        large
        layout="prev, pager, next"
        :page-size="numberPerPage"
        :total="numberOfHits"
        @current-change="pageChange"
      ></el-pagination>
    </div>
  </el-card>
</template>

<script>
/* eslint-disable no-alert, no-console */
import {
  ElButton as Button,
  ElCard as Card,
  ElDrawer as Drawer,
  ElIcon as Icon,
  ElInput as Input,
  ElPagination as Pagination,
} from 'element-plus'
import SearchFilters from './SearchFilters.vue'
import SearchHistory from './SearchHistory.vue'
import EventBus from './EventBus.js'

import DatasetCard from "./DatasetCard.vue";
import PMRDatasetCard from "./PMRDatasetCard.vue";

import { AlgoliaClient } from '../algolia/algolia.js'
import { getFilters, facetPropPathMapping } from '../algolia/utils.js'
import FlatmapQueries from '../flatmapQueries/flatmapQueries.js'
import mixedPageCalculation from '../mixins/mixedPageCalculation.vue'
import { markRaw } from 'vue'
const RatioOfPMRResults = 0.2; // Ratio of PMR results to Sparc results

// handleErrors: A custom fetch error handler to recieve messages from the server
//    even when an error is found
var handleErrors = async function (response) {
  if (!response.ok) {
    let parse = await response.json()
    if (parse) {
      throw new Error(parse.message)
    } else {
      throw new Error(response)
    }
  }
  return response
}

var initial_state = {
  filters: [],
  searchInput: '',
  lastSearch: '',
  results: [],
  pmrNumberOfHits: 0,
  sparcNumberOfHits: 0,
  filter: [],
  loadingCards: false,
  numberPerPage: 10,
  page: 1,
  pmrResultsOnlyFlag: false,
  noPMRResultsFlag: false,
  hasSearched: false,
  contextCardEnabled: false,
  pmrResults: [],
  RatioOfPMRResults: RatioOfPMRResults,
};



export default {
  components: {
    SearchFilters,
    DatasetCard,
    PMRDatasetCard,
    SearchHistory,
    Button,
    Card,
    Drawer,
    Icon,
    Input,
    Pagination
  },
  name: 'SideBarContent',
  mixins: [mixedPageCalculation],
  props: {
    visible: {
      type: Boolean,
      default: false,
    },
    isDrawer: {
      type: Boolean,
      default: true,
    },
    entry: {
      type: Object,
      default: () => initial_state,
    },
    initFilters: {
      type: Object,
      default: {
        filter: [],
        searchInput: '',
      }
    },
    envVars: {
      type: Object,
      default: () => {},
    },
    hyperlinks: {
      type: Object,
      default: {},
    }
  },
  data: function () {
    return {
      ...this.entry,
      ...this.initFilters,
      algoliaClient: undefined,
      bodyStyle: {
        flex: '1 1 auto',
        'flex-flow': 'column',
        display: 'flex',
      },
      cascaderIsReady: false,
    }
  },
  computed: {
    // This computed property populates filter data's entry object with $data from this sidebar
    filterEntry: function () {
      return {
        numberOfHits: this.numberOfHits,
        filterFacets: this.filter,
      }
    },
    // npp_SPARC: Number per page for SPARC datasets
    npp_SPARC: function () {
      return Math.round(this.numberPerPage * (1 - RatioOfPMRResults))
    },
    // npp_PMR: Number per page for PMR datasets
    npp_PMR: function () {
      return Math.round(this.numberPerPage * RatioOfPMRResults)
    },
    numberOfHits: function () {
      return this.sparcNumberOfHits + this.pmrNumberOfHits
    },

  },
  methods: {
    hoverChanged: function (data) {
      this.$emit('hover-changed', data)
    },
    // resetSearch: Resets the results, and page, and variable results ratio
    //     Does not: reset filters, search input, or search history
    resetSearch: function () {
      this.pmrNumberOfHits = 0
      this.sparcNumberOfHits = 0
      this.page = 1
      this.calculateVariableRatio()
      this.discoverIds = []
      this._dois = []
      this.results = []
      this.loadingCards = false
    },
    // openPMRSearch: Resets the results, populates dataset cards and filters with PMR data.
    openPMRSearch: function (filter, search = '') {
      this.resetSearch()
      this.loadingCards = true;
      this.flatmapQueries.updateOffset(this.calculatePMROffest())
      this.flatmapQueries.updateLimit(this.PMRLimit(this.pmrResultsOnlyFlag))
      this.flatmapQueries.pmrSearch(filter, search).then((data) => {
        data.forEach((result) => {
          this.results.push(result)
        })
        this.pmrNumberOfHits = this.flatmapQueries.numberOfHits
        this.loadingCards = false;
      })
    },
    openSearch: function (filter, search = '', option = { withSearch: true }) {
      this.searchInput = search
      //Proceed normally if cascader is ready
      if (this.cascaderIsReady) {
        this.filter =
          this.$refs.filtersRef.getHierarchicalValidatedFilters(filter)
        //Facets provided but cannot find at least one valid
        //facet. Tell the users the search is invalid and reset
        //facets check boxes.
        if (
          filter &&
          filter.length > 0 &&
          this.filter &&
          this.filter.length === 0
        ) {
          this.$refs.filtersRef.checkShowAllBoxes()
          this.resetSearch()
        } else if (this.filter) {
          if (option.withSearch) {
            if (this.pmrResultsOnlyFlag) {
              this.openPMRSearch(this.filter, search);
            } else if (this.noPMRResultsFlag) {
              this.searchAlgolia(this.filter, search);
            } else {
              this.searchAlgolia(this.filter, search);
              this.openPMRSearch(this.filter, search);
            }
          }
          this.$refs.filtersRef.setCascader(this.filter)
        }
      } else {
        //cascader is not ready, perform search if no filter is set,
        //otherwise waith for cascader to be ready
        this.filter = filter
        if ((!filter || filter.length == 0) && option.withSearch) {
          if (this.pmrResultsOnlyFlag) {
            this.openPMRSearch(this.filter, search);
          } else if (this.noPMRResultsFlag) {
            this.searchAlgolia(this.filter, search);
          } else {
            this.searchAlgolia(this.filter, search);
            this.openPMRSearch(this.filter, search);
          }
        }
      }
    },
    addFilter: function (filter) {
      if (this.cascaderIsReady) {
        this.resetSearch()
        if (filter) {
          if (this.$refs.filtersRef.addFilter(filter))
            this.$refs.filtersRef.initiateSearch()
        }
      } else {
        if (Array.isArray(this.filter)) {
          this.filter.push(filter)
        } else {
          this.filter = [filter]
        }
      }
    },
    cascaderReady: function () {
      this.cascaderIsReady = true
      this.openSearch(this.filter, this.searchInput)
    },
    clearSearchClicked: function () {
      this.searchInput = ''
      this.resetSearch()
      this.openSearch(this.filter, this.searchInput)
      this.$refs.searchHistory.selectValue = 'Full search history'
    },
    searchEvent: function (event = false) {
      if (event.keyCode === 13 || event instanceof MouseEvent) {
        this.openSearch(this.filter, this.searchInput)
        this.$refs.searchHistory.selectValue = 'Full search history'
        this.$refs.searchHistory.addSearchToHistory(
          this.filter,
          this.searchInput
        )
      }
    },
    updatePMROnlyFlag: function (filters) {
      const dataTypeFilters = filters.filter((item) => item.facetPropPath === 'item.types.name');
      const pmrFilter = dataTypeFilters.filter((item) => item.facet === 'PMR');
      const showAllFilter = dataTypeFilters.filter((item) => item.facet === 'Show all');

      this.pmrResultsOnlyFlag = false;
      this.noPMRResultsFlag = false;

      if (dataTypeFilters.length === 1 && pmrFilter.length === 1) {
        this.pmrResultsOnlyFlag = true;
      }

      if (dataTypeFilters.length > 0 && pmrFilter.length === 0 && showAllFilter.length === 0) {
        this.noPMRResultsFlag = true;
      }
    },
    filterUpdate: function (filters) {
      this.filter = [...filters]
      this.searchAndFilterUpdate();
      // Check if PMR is in the filters
      this.updatePMROnlyFlag(filters)
      // Note that we cannot use the openSearch function as that modifies filters
      this.resetSearch()
      if (this.pmrResultsOnlyFlag) {
        this.openPMRSearch(filters, this.searchInput)
      } else if (this.noPMRResultsFlag) {
        this.searchAlgolia(filters, this.searchInput)
      } else {
        this.searchAlgolia(filters, this.searchInput)
        this.openPMRSearch(filters, this.searchInput)
      }
      this.$emit('search-changed', {
        value: filters,
        type: 'filter-update',
      })
    },
    /**
     * Transform filters for third level items to perform search
     * because cascader keeps adding it back.
     */
    transformFiltersBeforeSearch: function (filters) {
      return filters.map((filter) => {
        if (filter.facet2) {
          filter.facet = filter.facet2;
          delete filter.facet2;
        }
        return filter;
      });
    },
    searchAndFilterUpdate: function () {
      this.resetPageNavigation();
      const transformedFilters = this.transformFiltersBeforeSearch(this.filters);
      this.searchAlgolia(transformedFilters, this.searchInput);
      this.$refs.searchHistory.selectValue = 'Search history';
      // save history only if there has value
      if (this.filters.length || this.searchInput?.trim()) {
        this.$refs.searchHistory.addSearchToHistory(
          this.filters,
          this.searchInput
        );
      }
    },
    searchAlgolia(filters, query = '') {

      // Remove loading if we dont expect any results
      if (this.SPARCLimit() === 0) {
        this.loadingCards = false
        return
      }

      // Algolia search

      this.loadingCards = true
      this.algoliaClient
        .anatomyInSearch(getFilters(filters), query)
        .then((r) => {
          // Send result anatomy to the scaffold and flatmap
          EventBus.emit('anatomy-in-datasets', r.forFlatmap)
          EventBus.emit('number-of-datasets-for-anatomies', r.forScaffold)
        })
      this.algoliaClient
        .search(getFilters(filters), query, this.calculateSPARCOffest(), this.SPARCLimit(this.pmrResultsOnlyFlag) )
        .then((searchData) => {
          this.sparcNumberOfHits = searchData.total
          this.discoverIds = searchData.discoverIds
          this._dois = searchData.dois
          searchData.items.forEach((item) => {
            item.detailsReady = false
            this.results.push(item)
          })
          // add the items to the results
          this.results.concat(searchData.items)
          this.loadingCards = false
          this.scrollToTop()
          this.$emit('search-changed', {
            value: this.searchInput,
            type: 'query-update',
          })
          if (this._abortController) this._abortController.abort()
          this._abortController = new AbortController()
          const signal = this._abortController.signal
          //Search ongoing, let the current flow progress
          this.perItemSearch(signal, { count: 0 })
        })
    },
    filtersLoading: function (val) {
      this.loadingCards = val
    },
    numberPerPageUpdate: function (val) {
      this.numberPerPage = val
      this.pageChange(1)
    },
    pageChange: function (page) {
      this.page = page
      this.results = []
      this.calculateVariableRatio()
      this.openSearch(this.filter, this.searchInput, false)
    },
    handleMissingData: function (doi) {
      let i = this.results.findIndex((res) => res.doi === doi)
      if (this.results[i]) this.results[i].detailsReady = true
    },
    perItemSearch: function (signal, data) {
      //Maximum 10 downloads at once to prevent long waiting time
      //between unfinished search and new search
      const maxDownloads = 10
      if (maxDownloads > data.count) {
        const doi = this._dois.shift()
        if (doi) {
          data.count++
          this.callSciCrunch(this.envVars.API_LOCATION, { dois: [doi] }, signal)
            .then((result) => {
              if (result.numberOfHits === 0) this.handleMissingData(doi)
              else this.resultsProcessing(result)
              this.$refs.content.style['overflow-y'] = 'scroll'
              data.count--
              //Async::Download finished, get the next one
              this.perItemSearch(signal, data)
            })
            .catch((result) => {
              if (result.name !== 'AbortError') {
                this.handleMissingData(doi)
                data.count--
                //Async::Download not aborted, get the next one
                this.perItemSearch(signal, data)
              }
            })
          //Check and make another request until it gets to max downloads
          this.perItemSearch(signal, data)
        }
      }
    },
    scrollToTop: function () {
      if (this.$refs.content) {
        this.$refs.content.scroll({ top: 0, behavior: 'smooth' })
      }
    },
    resetPageNavigation: function () {
      this.page = 1
    },
    resultsProcessing: function (data) {
      this.lastSearch = this.searchInput

      if (data.results.length === 0) {
        return
      }
      data.results.forEach((element) => {
        // match the scicrunch result with algolia result
        let i = this.results.findIndex((res) =>
          element.doi ? element.doi.includes(res.doi) : false
        )
        // Assign scicrunch results to the object
        Object.assign(this.results[i], element)
        // Assign the attributes that need some processing
        Object.assign(this.results[i], {
          numberSamples: element.sampleSize ? parseInt(element.sampleSize) : 0,
          numberSubjects: element.subjectSize
            ? parseInt(element.subjectSize)
            : 0,
          updated:
            (element.updated && element.updated.length) > 0
              ? element.updated[0].timestamp.split('T')[0]
              : '',
          url: element.uri[0],
          datasetId: element.dataset_identifier,
          datasetRevision: element.dataset_revision,
          datasetVersion: element.dataset_version,
          organs:
            element.organs && element.organs.length > 0
              ? [...new Set(element.organs.map((v) => v.name))]
              : undefined,
          species: element.organisms
            ? element.organisms[0].species
              ? [
                  ...new Set(
                    element.organisms.map((v) =>
                      v.species ? v.species.name : null
                    )
                  ),
                ]
              : undefined
            : undefined, // This processing only includes each gender once into 'sexes'
          scaffolds: element['abi-scaffold-metadata-file'],
          thumbnails: element['abi-thumbnail']
            ? element['abi-thumbnail']
            : element['abi-scaffold-thumbnail'],
          scaffoldViews: element['abi-scaffold-view-file'],
          videos: element.video,
          plots: element['abi-plot'],
          images: element['common-images'],
          contextualInformation:
            element['abi-contextual-information'].length > 0
              ? element['abi-contextual-information']
              : undefined,
          segmentation: element['mbf-segmentation'],
          simulation: element['abi-simulation-file'],
          additionalLinks: element.additionalLinks,
          detailsReady: true,
        })
        this.results[i] = this.results[i]
      })
    },
    createfilterParams: function (params) {
      let p = new URLSearchParams()
      //Check if field is array or value
      for (const key in params) {
        if (Array.isArray(params[key])) {
          params[key].forEach((e) => {
            p.append(key, e)
          })
        } else {
          p.append(key, params[key])
        }
      }
      return p.toString()
    },
    callSciCrunch: function (apiLocation, params = {}, signal) {
      return new Promise((resolve, reject) => {
        // Add parameters if we are sent them
        let fullEndpoint =
          this.envVars.API_LOCATION +
          this.searchEndpoint +
          '?' +
          this.createfilterParams(params)
        fetch(fullEndpoint, { signal })
          .then(handleErrors)
          .then((response) => response.json())
          .then((data) => resolve(data))
          .catch((data) => reject(data))
      })
    },
    getAlgoliaFacets: async function () {
      let facets = await this.algoliaClient.getAlgoliaFacets(
        facetPropPathMapping
      )
      return facets
    },
    searchHistorySearch: function (item) {
      this.searchInput = item.search
      this.filters = item.filters
      this.searchAndFilterUpdate();
      // withSearch: false to prevent algoliaSearch in openSearch
      this.openSearch([...item.filters], item.search, { withSearch: false });
    },
  },
  mounted: function () {
    // initialise algolia
    this.algoliaClient = markRaw(new AlgoliaClient(
      this.envVars.ALGOLIA_ID,
      this.envVars.ALGOLIA_KEY,
      this.envVars.PENNSIEVE_API_LOCATION
    ))
    this.algoliaClient.initIndex(this.envVars.ALGOLIA_INDEX)

    // initialise flatmap queries
    this.flatmapQueries = new FlatmapQueries()
    this.flatmapQueries.initialise(this.envVars.FLATMAP_API_LOCATION)

    // open search
    this.openSearch(this.filter, this.searchInput, this.mode )
  },
  created: function () {
    //Create non-reactive local variables
    this.searchEndpoint = 'dataset_info/using_multiple_dois/'
  },
}
</script>

<style lang="scss" scoped>
.dataset-card {
  position: relative;

  &::before {
    content: "";
    display: block;
    width: calc(100% - 15px);
    height: 100%;
    position: absolute;
    top: 7px;
    left: 7px;
    border-style: solid;
    border-radius: 5px;
    border-color: transparent;
  }

  &:hover {
    &::before {
      border-color: var(--el-color-primary);
    }
  }
}

.content-card {
  height: 100%;
  flex-flow: column;
  display: flex;
  border: 0;
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
}

.data-type-select {
  width: 90px;
  margin-left: 10px;
}

.button {
  background-color: $app-primary-color;
  border: $app-primary-color;
  color: white;
}

.step-item {
  font-size: 14px;
  margin-bottom: 18px;
  text-align: left;
}

.search-input {
  width: 298px !important;
  height: 40px;
  padding-right: 14px;

  :deep(.el-input__inner) {
    font-family: inherit;
  }
}

.header {
  .el-button {
    font-family: inherit;

    &:hover,
    &:focus {
      background: $app-primary-color;
      box-shadow: -3px 2px 4px #00000040;
      color: #fff;
    }
  }
}

.pagination {
  padding-bottom: 16px;
  background-color: white;
  padding-left: 95px;
  font-weight: bold;
}

.pagination :deep(button) {
  background-color: white !important;
}
.pagination :deep(li) {
  background-color: white !important;
}
.pagination :deep(li.is-active) {
  color: $app-primary-color;
}

.error-feedback {
  font-family: Asap;
  font-size: 14px;
  font-style: italic;
  padding-top: 15px;
}

.content-card :deep(.el-card__header) {
  background-color: #292b66;
  padding: 1rem;
}

.content-card :deep(.el-card__body) {
  background-color: #f7faff;
  overflow-y: hidden;
  padding: 1rem;
}

.content {
  // width: 515px;
  flex: 1 1 auto;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.06);
  border: solid 1px #e4e7ed;
  background-color: #ffffff;
  overflow-y: scroll;
  scrollbar-width: thin;
  border-radius: var(--el-border-radius-base);
}

.content :deep(.el-loading-spinner .path) {
  stroke: $app-primary-color;
}

.content :deep(.step-item:first-child .seperator-path) {
  display: none;
}

.content :deep(.step-item:not(:first-child) .seperator-path) {
  width: 455px;
  height: 0px;
  border: solid 1px #e4e7ed;
  background-color: #e4e7ed;
}

.scrollbar::-webkit-scrollbar-track {
  border-radius: 10px;
  background-color: #f5f5f5;
}

.scrollbar::-webkit-scrollbar {
  width: 12px;
  right: -12px;
  background-color: #f5f5f5;
}

.scrollbar::-webkit-scrollbar-thumb {
  border-radius: 4px;
  box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.06);
  background-color: #979797;
}

:deep(.el-input__suffix) {
  padding-right: 0px;
}

:deep(.my-drawer) {
  background: rgba(0, 0, 0, 0);
  box-shadow: none;
}
</style>
