<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
    <!-- Playback Controls -->
    <div id="controls" v-if="player.playing">
      <button @click="playPause">
        {{ player.playing ? "Pause" : "Play" }}
      </button>
      <button @click="previousTrack">Previous</button>
      <button @click="nextTrack">Next</button>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Play or pause the current track.
     */
    async playPause() {
      const endpoint = this.player.playing
        ? `${this.endpoints.base}/me/player/pause`
        : `${this.endpoints.base}/me/player/play`;

      try {
        await fetch(endpoint, {
          method: 'PUT',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`,
          },
        });
        this.player.playing = !this.player.playing; // Update local state
      } catch (error) {
        console.error("Error toggling play/pause:", error);
      }
    },

    /**
     * Skip to the next track.
     */
    async nextTrack() {
      try {
        await fetch(`${this.endpoints.base}/me/player/next`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`,
          },
        });
      } catch (error) {
        console.error("Error skipping to the next track:", error);
      }
    },

    /**
     * Skip to the previous track.
     */
    async previousTrack() {
      try {
        await fetch(`${this.endpoints.base}/me/player/previous`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`,
          },
        });
      } catch (error) {
        console.error("Error skipping to the previous track:", error);
      }
    },

    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      /**
       * No image (rare).
       */
      if (!this.player.trackAlbum?.image) {
        return
      }

      /**
       * Run node-vibrant to get colours.
       */
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * The newly fetched track is the same as our stored
       * one, we don't want to update the DOM yet.
       */
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item
        })
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          }
        })

      this.swatches = albumColours

      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    },

    /**
     * Spotify Player control: Start playback from a specific track or URI.
     */
    async startPlayback(uri) {
      try {
        const response = await fetch(`${this.endpoints.base}/me/player/play`, {
          method: 'PUT',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`,
          },
          body: JSON.stringify({ uris: [uri] })
        });

        if (!response.ok) {
          throw new Error("Error starting playback");
        }

        this.player.playing = true; // Update state
        this.getNowPlaying(); // Fetch updated track data
      } catch (error) {
        console.error("Error starting playback:", error);
      }
    },

    /**
     * Search Spotify for a track and start playback.
     */
    async searchAndPlay(trackName) {
      try {
        const searchResponse = await fetch(
          `${this.endpoints.base}/v1/search?type=track&q=${encodeURIComponent(trackName)}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`,
            },
          }
        );

        const searchData = await searchResponse.json();

        if (searchData.tracks.items.length > 0) {
          const trackUri = searchData.tracks.items[0].uri;
          await this.startPlayback(trackUri);
        } else {
          console.error("No track found for search:", trackName);
        }
      } catch (error) {
        console.error("Error searching for track:", error);
      }
    }
  }
}
</script>

<style src="@/styles/components/authorise.scss" lang="scss" scoped></style>
