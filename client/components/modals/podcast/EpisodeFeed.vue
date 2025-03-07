<template>
  <modals-modal v-model="show" name="podcast-episodes-modal" :width="1200" :height="'unset'" :processing="processing">
    <template #outer>
      <div class="absolute top-0 left-0 p-5 w-2/3 overflow-hidden">
        <p class="text-3xl text-white truncate">{{ title }}</p>
      </div>
    </template>
    <div ref="wrapper" id="podcast-wrapper" class="p-4 w-full text-sm py-2 rounded-lg bg-bg shadow-lg border border-black-300 relative overflow-hidden">
      <div v-if="episodesCleaned.length" class="w-full py-3 mx-auto flex">
        <form @submit.prevent="submit" class="flex flex-grow">
          <ui-text-input v-model="search" @input="inputUpdate" type="search" :placeholder="$strings.PlaceholderSearchEpisode" class="flex-grow mr-2 text-sm md:text-base" />
        </form>
      </div>
      <div ref="episodeContainer" id="episodes-scroll" class="w-full overflow-x-hidden overflow-y-auto">
        <div
          v-for="(episode, index) in episodesList"
          :key="index"
          class="relative"
          :class="itemEpisodeMap[episode.cleanUrl] ? 'bg-primary bg-opacity-40' : selectedEpisodes[episode.cleanUrl] ? 'cursor-pointer bg-success bg-opacity-10' : index % 2 == 0 ? 'cursor-pointer bg-primary bg-opacity-25 hover:bg-opacity-40' : 'cursor-pointer bg-primary bg-opacity-5 hover:bg-opacity-25'"
          @click="toggleSelectEpisode(episode)"
        >
          <div class="absolute top-0 left-0 h-full flex items-center p-2">
            <span v-if="itemEpisodeMap[episode.cleanUrl]" class="material-icons text-success text-xl">download_done</span>
            <ui-checkbox v-else v-model="selectedEpisodes[episode.cleanUrl]" small checkbox-bg="primary" border-color="gray-600" />
          </div>
          <div class="px-8 py-2">
            <div class="flex items-center font-semibold text-gray-200">
              <div v-if="episode.season || episode.episode">#</div>
              <div v-if="episode.season">{{ episode.season }}x</div>
              <div v-if="episode.episode">{{ episode.episode }}</div>
            </div>
            <div class="flex items-center mb-1">
              <div class="break-words">{{ episode.title }}</div>
              <widgets-podcast-type-indicator :type="episode.episodeType" />
            </div>
            <p v-if="episode.subtitle" class="break-words mb-1 text-sm text-gray-300 episode-subtitle">{{ episode.subtitle }}</p>
            <p class="text-xs text-gray-300">Published {{ episode.publishedAt ? $dateDistanceFromNow(episode.publishedAt) : 'Unknown' }}</p>
          </div>
        </div>
      </div>
      <div class="flex justify-end pt-4">
        <ui-checkbox v-if="!allDownloaded" v-model="selectAll" @input="toggleSelectAll" label="Select all episodes" small checkbox-bg="primary" border-color="gray-600" class="mx-8" />
        <ui-btn v-if="!allDownloaded" :disabled="!episodesSelected.length" @click="submit">{{ buttonText }}</ui-btn>
        <p v-else class="text-success text-base px-2 py-4">All episodes are downloaded</p>
      </div>
    </div>
  </modals-modal>
</template>

<script>
export default {
  props: {
    value: Boolean,
    libraryItem: {
      type: Object,
      default: () => {}
    },
    episodes: {
      type: Array,
      default: () => []
    }
  },
  data() {
    return {
      processing: false,
      episodesCleaned: [],
      selectedEpisodes: {},
      selectAll: false,
      search: null,
      searchTimeout: null,
      searchText: null
    }
  },
  watch: {
    show: {
      immediate: true,
      handler(newVal) {
        if (newVal) this.init()
      }
    }
  },
  computed: {
    show: {
      get() {
        return this.value
      },
      set(val) {
        this.$emit('input', val)
      }
    },
    title() {
      if (!this.libraryItem) return ''
      return this.libraryItem.media.metadata.title || 'Unknown'
    },
    allDownloaded() {
      return !this.episodesCleaned.some((episode) => !this.itemEpisodeMap[episode.cleanUrl])
    },
    episodesSelected() {
      return Object.keys(this.selectedEpisodes).filter((key) => !!this.selectedEpisodes[key])
    },
    buttonText() {
      if (!this.episodesSelected.length) return 'No Episodes Selected'
      return `Download ${this.episodesSelected.length} Episode${this.episodesSelected.length > 1 ? 's' : ''}`
    },
    itemEpisodes() {
      if (!this.libraryItem) return []
      return this.libraryItem.media.episodes || []
    },
    itemEpisodeMap() {
      var map = {}
      this.itemEpisodes.forEach((item) => {
        if (item.enclosure) map[item.enclosure.url.split('?')[0]] = true
      })
      return map
    },
    episodesList() {
      return this.episodesCleaned.filter((episode) => {
        if (!this.searchText) return true
        return (episode.title && episode.title.toLowerCase().includes(this.searchText)) || (episode.subtitle && episode.subtitle.toLowerCase().includes(this.searchText))
      })
    }
  },
  methods: {
    inputUpdate() {
      clearTimeout(this.searchTimeout)
      this.searchTimeout = setTimeout(() => {
        if (!this.search || !this.search.trim()) {
          this.searchText = ''
          return
        }
        this.searchText = this.search.toLowerCase().trim()
      }, 500)
    },
    toggleSelectAll(val) {
      for (const episode of this.episodesCleaned) {
        if (this.itemEpisodeMap[episode.cleanUrl]) this.selectedEpisodes[episode.cleanUrl] = false
        else this.$set(this.selectedEpisodes, episode.cleanUrl, val)
      }
    },
    checkSetIsSelectedAll() {
      for (const episode of this.episodesCleaned) {
        if (!this.itemEpisodeMap[episode.cleanUrl] && !this.selectedEpisodes[episode.cleanUrl]) {
          this.selectAll = false
          return
        }
      }
      this.selectAll = true
    },
    toggleSelectEpisode(episode) {
      if (this.itemEpisodeMap[episode.enclosure.url?.split('?')[0]]) return
      this.$set(this.selectedEpisodes, episode.cleanUrl, !this.selectedEpisodes[episode.cleanUrl])
      this.checkSetIsSelectedAll()
    },
    submit() {
      var episodesToDownload = []
      if (this.episodesSelected.length) {
        episodesToDownload = this.episodesSelected.map((cleanUrl) => this.episodesCleaned.find((ep) => ep.cleanUrl == cleanUrl))
      }

      var payloadSize = JSON.stringify(episodesToDownload).length
      var sizeInMb = payloadSize / 1024 / 1024
      var sizeInMbPretty = sizeInMb.toFixed(2) + 'MB'
      console.log('Request size', sizeInMb)
      if (sizeInMb > 4.99) {
        return this.$toast.error(`Request is too large (${sizeInMbPretty}) should be < 5Mb`)
      }

      this.processing = true
      this.$axios
        .$post(`/api/podcasts/${this.libraryItem.id}/download-episodes`, episodesToDownload)
        .then(() => {
          this.processing = false
          this.$toast.success('Started downloading episodes')
          this.show = false
        })
        .catch((error) => {
          var errorMsg = error.response && error.response.data ? error.response.data : 'Failed to download episodes'
          console.error('Failed to download episodes', error)
          this.processing = false
          this.$toast.error(errorMsg)

          this.selectedEpisodes = {}
          this.selectAll = false
        })
    },
    init() {
      this.episodesCleaned = this.episodes
        .filter((ep) => ep.enclosure?.url)
        .map((_ep) => {
          return {
            ..._ep,
            cleanUrl: _ep.enclosure.url.split('?')[0]
          }
        })
      this.episodesCleaned.sort((a, b) => (a.publishedAt < b.publishedAt ? 1 : -1))
      this.selectAll = false
      this.selectedEpisodes = {}
    }
  },
  mounted() {}
}
</script>

<style scoped>
#podcast-wrapper {
  min-height: 400px;
  max-height: 80vh;
}
#episodes-scroll {
  max-height: calc(80vh - 200px);
}
</style>
