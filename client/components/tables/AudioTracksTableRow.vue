<template>
  <tr>
    <td class="text-center">
      <p>{{ track.index }}</p>
    </td>
    <td class="font-sans">{{ showFullPath ? track.metadata.path : track.metadata.filename }}</td>
    <td v-if="!showFullPath" class="hidden lg:table-cell">
      {{ track.audioFile.codec || '' }}
    </td>
    <td v-if="!showFullPath" class="hidden xl:table-cell">
      {{ $bytesPretty(track.audioFile.bitRate || 0, 0) }}
    </td>
    <td class="hidden md:table-cell">
      {{ $bytesPretty(track.metadata.size) }}
    </td>
    <td class="hidden sm:table-cell">
      {{ $secondsToTimestamp(track.duration) }}
    </td>
    <td v-if="contextMenuItems.length" class="text-center">
      <ui-context-menu-dropdown :items="contextMenuItems" menu-width="110px" @action="contextMenuAction" />
    </td>
  </tr>
</template>

<script>
export default {
  props: {
    libraryItemId: String,
    showFullPath: Boolean,
    track: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {}
  },
  computed: {
    userToken() {
      return this.$store.getters['user/getToken']
    },
    userCanDownload() {
      return this.$store.getters['user/getUserCanDownload']
    },
    userCanDelete() {
      return this.$store.getters['user/getUserCanDelete']
    },
    userIsAdmin() {
      return this.$store.getters['user/getIsAdminOrUp']
    },
    contextMenuItems() {
      const items = []
      if (this.userCanDownload) {
        items.push({
          text: this.$strings.LabelDownload,
          action: 'download'
        })
      }

      if (this.userCanDelete) {
        items.push({
          text: this.$strings.ButtonDelete,
          action: 'delete'
        })
      }

      if (this.userIsAdmin) {
        items.push({
          text: this.$strings.LabelMoreInfo,
          action: 'more'
        })
      }
      return items
    },
    downloadUrl() {
      return `${process.env.serverUrl}/s/item/${this.libraryItemId}/${this.$encodeUriPath(this.track.metadata.relPath).replace(/^\//, '')}?token=${this.userToken}`
    }
  },
  methods: {
    contextMenuAction(action) {
      if (action === 'delete') {
        this.deleteLibraryFile()
      } else if (action === 'download') {
        this.downloadLibraryFile()
      } else if (action === 'more') {
        this.$emit('showMore', this.track.audioFile)
      }
    },
    deleteLibraryFile() {
      const payload = {
        message: 'This will delete the file from your file system. Are you sure?',
        callback: (confirmed) => {
          if (confirmed) {
            this.$axios
              .$delete(`/api/items/${this.libraryItemId}/file/${this.track.audioFile.ino}`)
              .then(() => {
                this.$toast.success('File deleted')
              })
              .catch((error) => {
                console.error('Failed to delete file', error)
                this.$toast.error('Failed to delete file')
              })
          }
        },
        type: 'yesNo'
      }
      this.$store.commit('globals/setConfirmPrompt', payload)
    },
    downloadLibraryFile() {
      const a = document.createElement('a')
      a.style.display = 'none'
      a.href = this.downloadUrl
      a.download = this.track.metadata.filename
      document.body.appendChild(a)
      a.click()
      setTimeout(() => {
        a.remove()
      })
    }
  },
  mounted() {}
}
</script>