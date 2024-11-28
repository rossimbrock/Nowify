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

    <!-- Player Controls -->
    <div class="player-controls">
      <button @click="skipToPrevious" class="control-button">⏮️</button>
      <button @click="togglePlayPause" class="control-button">
        {{ isPlaying ? '⏸️' : '▶️' }}
      </button>
      <button @click="skipToNext" class="control-button">⏭️</button>
    </div>
  </div>
</template>

<script>
import * as Vibrant from "node-vibrant";
import props from "@/utils/props.js";

export default {
  name: "NowPlaying",

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player,
  },

  data() {
    return {
      pollPlaying: "",
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: "",
      swatches: [],
      isPlaying: false, // Track play/pause state
    };
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(", ");
    },
  },

  mounted() {
    this.setDataInterval();
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying);
  },

  methods: {
    async togglePlayPause() {
      try {
        const endpoint = this.isPlaying
          ? `${this.endpoints.base}/me/player/pause`
          : `${this.endpoints.base}/me/player/play`;

        const response = await fetch(endpoint, {
          method: "PUT",
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`,
          },
        });

        if (response.ok) {
          this.isPlaying = !this.isPlaying;
        } else {
          console.error("Failed to toggle play/pause:", response.status);
        }
      } catch (error) {
        console.error("Error toggling play/pause:", error);
      }
    },

    async skipToNext() {
      try {
        const response = await fetch(
          `${this.endpoints.base}/me/player/next`,
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`,
            },
          }
        );

        if (!response.ok) {
          console.error("Failed to skip to next track:", response.status);
        }
      } catch (error) {
        console.error("Error skipping to next track:", error);
      }
    },

    async skipToPrevious() {
      try {
        const response = await fetch(
          `${this.endpoints.base}/me/player/previous`,
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`,
            },
          }
        );

        if (!response.ok) {
          console.error("Failed to skip to previous track:", response.status);
        }
      } catch (error) {
        console.error("Error skipping to previous track:", error);
      }
    },

    // Existing methods...
    getNowPlaying() { /* unchanged */ },
    getNowPlayingClass() { /* unchanged */ },
    getAlbumColours() { /* unchanged */ },
    getEmptyPlayer() { /* unchanged */ },
    setDataInterval() { /* unchanged */ },
    setAppColours() { /* unchanged */ },
    handleNowPlaying() { /* unchanged */ },
    handleAlbumPalette(palette) { /* unchanged */ },
    handleExpiredToken() { /* unchanged */ },
  },

  watch: {
    auth(newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying);
      }
    },
    playerResponse() {
      this.handleNowPlaying();
    },
    playerData() {
      this.$emit("spotifyTrackUpdated", this.playerData);
      this.$nextTick(() => {
        this.getAlbumColours();
      });
    },
  },
};
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
