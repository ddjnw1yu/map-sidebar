<template>
  <div class="dataset-card-container"  ref="container">
    <div class="dataset-card"  ref="card">
      <div class="seperator-path"></div>
      <div v-loading="loading" class="card" >
        <span class="card-left">
          <a class="card-image-container" :href="entry.exposure" target="_blank">
            <img
              class="card-image"
              :src="thumbnail"
              width="210"
              height="210"
              :alt="entry.title"
              loading="lazy"
            />
          </a>
        </span>
        <div class="card-right">
          <el-tag type="primary" class="source-tag">PMR</el-tag>
          <div>
            <h4 class="title">{{ entry.title }}</h4>
            <div class="details">
              {{ entry.authors }}
            </div>
          </div>
          <div class="details" v-if="entry.description">
            {{ entry.description }}
          </div>
          <div class="buttons-group">
            <a
              :href="entry.exposure"
              target="_blank"
              class="el-button el-button--small button"
            >
              Exposure
            </a>
            <a
              v-if="entry.omex"
              :href="entry.omex"
              target="_blank"
              class="el-button el-button--small button"
            >
              COMBINE archive
            </a>
            <el-button v-if="entry.flatmap"
              size="small"
              class="button"
              @click="onFlatmapClick(entry.flatmap)"
            >
              Flatmap
            </el-button>
            <el-button v-if="entry.simulation"
              size="small"
              class="button"
              @click="onSimulationClick(entry.simulation)"
            >
              Simulation
            </el-button>
            <el-button 
              v-if="Object.keys(hyperlinks).length"
              v-for="(value, key) in hyperlinks"
              size="small"
              class="button"
              @click="onHyperlinkClick(key, value)"
            >
              Multiscale {{ key }}
            </el-button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>


<script>
/* eslint-disable no-alert, no-console */
import {
  ElButton,
  ElIcon,
  ElTag
} from 'element-plus'
import EventBus from './EventBus.js'

/**
 * Entry Object - Data types
 * ---------------------------------------
"exposure"
    type: String
    description: The exposure URL

"title"
    type: String
    description: The title

"sha"
    type: String
    description:

"workspace"
    type: String
    description: The workspace URL

"omex"
    type: String
    description:

"image"
    type: String
    description: The image URL

"authors"
    type: String
    description: Comma separated values if more than one

"description"
    type: String
    description: The description

"flatmap"
    type: String
    descriptin: (Optional) to link to flatmap

"simulation"
    type: String
    descriptin: (Optional) simulation resource
 * ---------------------------------------
 */

const placeholderThumbnail = 'assets/missing-image.svg';

export default {
  name: "PMRDatasetCard",
  components: {
    ElButton,
    ElIcon,
    ElTag
  },
  props: {
    /**
     * Object containing information for
     * the required viewing.
     */
    entry: {
      type: Object,
      default: () => {}
    },
    envVars: {
      type: Object,
      default: () => {}
    },
    hyperlinks: {
      type: Object,
      default: {},
    }
  },
  data: function () {
    return {
      thumbnail: this.entry.image || placeholderThumbnail,
      loading: false,
    };
  },
  methods: {
    onHyperlinkClick: function (key, value) {
      this.emitPMRActionClick({
        type: key[0].toUpperCase() + key.slice(1),
        resource: value,
        multiscale: true
      });
    },
    onFlatmapClick: function (data) {
      this.emitPMRActionClick({
        type: 'Flatmap',
        resource: data
      });
    },
    onSimulationClick: function (data) {
      this.emitPMRActionClick({
        type: 'Simulation',
        resource: data,
      });
    },
    emitPMRActionClick: function (data) {
      const payload = {
        ...data,
        name: this.entry.title,
        description: this.entry.description,
        apiLocation: this.envVars.API_LOCATION,
      };
      EventBus.emit('pmr-action-click', payload);
    },
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">

.dataset-card-container {
  padding: 0 1rem;
}

.dataset-card {
  position: relative;
  min-height:17rem;
}

.title {
  padding-bottom: 0.75rem;
  font-family: Asap;
  font-size: 14px;
  font-weight: bold;
  font-stretch: normal;
  font-style: normal;
  line-height: 1.5;
  letter-spacing: 1.05px;
  color: #484848;
}
.card {
  font-family: Asap;
  padding-top: 18px;
  position: relative;
  display: flex;

  h4,
  p {
    margin: 0;
  }
}

.card-left{
  flex: 1;
}

.card-right {
  flex: 1.3;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  min-height: 17rem;
}

.details {
  font-family: Asap;
  font-size: 14px;
  font-weight: normal;
  font-stretch: normal;
  font-style: normal;
  line-height: 1.5;
  letter-spacing: 1.05px;
  color: #484848;
}

.el-button {
  font-family: Asap;
  font-size: 14px;
  font-weight: normal;
  font-stretch: normal;
  font-style: normal;
  line-height: normal;
  letter-spacing: normal;
  text-decoration: none;
}

.button {
  background-color: $app-primary-color;
  border: $app-primary-color;
  color: white;
  transition: all 0.3s ease;

  &:hover,
  &:focus {
    color: white;
    background-color: $app-primary-color;
    outline: none;
  }
}

.card-image-container {
  display: block;
  width: 244px; // gallery image full-size width
}

.card-image {
  max-width: 100%;
  height: auto;
  object-fit: cover;
}

.buttons-group {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  gap: 0.75rem;

  .el-button {
    border-radius: 4px!important;
    font-size: 0.75rem!important;
    padding: 0.2rem 0.2rem!important;
    background: #f9f2fc!important;
    border: 1px solid $app-primary-color!important;
    color: $app-primary-color!important;

    &.active {
      background: $app-primary-color!important;
      border: 1px solid $app-primary-color!important;
      color: #fff!important;
    }
  }

  .el-button + .el-button {
    margin: 0;
  }
}

.source-tag {
  margin-bottom: 0.75rem;
  margin-right: 2rem;
  position: absolute;
  bottom: 0;
  right: 0;
}

.loading-icon {
  z-index: 20;
  width: 40px;
  height: 40px;
  left: 80px;
}

.loading-icon ::v-deep(.el-loading-mask) {
  background-color: rgba(117, 190, 218, 0.0) !important;
}

.loading-icon ::v-deep(.el-loading-spinner .path) {
  stroke: $app-primary-color;
}
</style>
