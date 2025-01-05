<template>
  <div id="app">
    <!-- Add the time display in the top-left corner -->
    <div class="now-playing__time" v-text="currentTime"></div>
    
    <div class="now-playing" :class="getNowPlayingClass()">
      <!-- Main Container for Image and Right Section -->
      <div class="now-playing__content">
        
        <!-- Album Cover on Left -->
        <div v-if="player.trackTitle" class="now-playing__cover">
          <img
            :src="player.trackAlbum.image"
            :alt="player.trackTitle"
            class="now-playing__image"
          />
        </div>
        
        <!-- Song Details and Controls on Right -->
        <div class="now-playing__right">
          <div class="now-playing__details">
            <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
            <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
            <h3 class="now-playing__album" v-text="player.trackAlbum.title"></h3>
          </div>

          <!-- Control Buttons (Play/Pause, Previous, Next) -->
          <div class="controls">
            <button @click="previousTrack" class="control-button prev">
              <i class="fas fa-backward"></i>
            </button>
            <button @click="togglePlayPause" class="control-button play-pause">
              <i :class="player.playing ? 'fas fa-pause' : 'fas fa-play'"></i>
            </button>
            <button @click="nextTrack" class="control-button next">
              <i class="fas fa-forward"></i>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant';

import props from '@/utils/props.js';

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
      swatches: [],
      currentTime: ''
    };
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(', ');
    }
  },

  mounted() {
    this.setDataInterval();
    this.updateTime();
    setInterval(this.updateTime, 60000); // Update time every minute
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying);
  },

  methods: {
    updateTime() {
      const now = new Date();
      const hours = now.getHours() % 12 || 12;
      const minutes = now.getMinutes().toString().padStart(2, '0');
      const ampm = now.getHours() >= 12 ? 'PM' : 'AM';
      this.currentTime = `${hours}:${minutes} ${ampm}`;
    },

    async getNowPlaying() {
      let data = {};
      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        );
        if (!response.ok) {
          throw new Error(`An error has occurred: ${response.status}`);
        }
        if (response.status === 204) {
          data = this.getEmptyPlayer();
          this.playerData = data;
          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data);
          });
          return;
        }
        data = await response.json();
        this.playerResponse = data;
      } catch (error) {
        this.handleExpiredToken();
        data = this.getEmptyPlayer();
        this.playerData = data;
        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data);
        });
      }
    },

    getNowPlayingClass() {
      const playerClass = this.player.trackTitle ? (this.player.playing ? 'active' : 'paused') : 'idle';
      return `now-playing--${playerClass}`;
    },

    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return;
      }

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette);
        });
    },

    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      };
    },

    setDataInterval() {
      clearInterval(this.pollPlaying);
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying();
      }, 1000);
    },

    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      );

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      );
    },

    handleNowPlaying() {
      if (this.playerResponse.error?.status === 401 || this.playerResponse.error?.status === 400) {
        this.handleExpiredToken();
        return;
      }
      if (this.playerResponse.is_playing === false) {
        // Do not clear the track data, just ensure it remains as is
        return;
      }
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        // Track hasn't changed, so do nothing
        return;
      }
      // If the track changes, update the data
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(artist => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images.find(image => image.height === 640 && image.width === 640)?.url || this.playerResponse.item.album.images[0].url
        }
      };
    },

    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => item === null ? null : item)
        .map(colour => ({
          text: palette[colour].getTitleTextColor(),
          background: palette[colour].getHex()
        }));

      this.swatches = albumColours;
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)];

      this.$nextTick(() => {
        this.setAppColours();
      });
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying);
      this.$emit('requestRefreshToken');
    },

    async togglePlayPause() {
      try {
        // Get the current playback state (whether it's playing or paused)
        const playbackStateResponse = await fetch(`${this.endpoints.base}/me/player`, {
          method: 'GET',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`,
            'Content-Type': 'application/json'
          }
        });

        if (!playbackStateResponse.ok) {
          throw new Error('Error fetching playback state');
        }

        const playbackState = await playbackStateResponse.json();

        if (!playbackState || !playbackState.is_playing) {
          // POST to start playback if nothing is playing
          const playResponse = await fetch(`${this.endpoints.base}/me/player/play`, {
            method: 'PUT', // Use PUT for play (should be PUT, not POST)
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`,
              'Content-Type': 'application/json'
            }
          });

          if (!playResponse.ok) {
            throw new Error('Error playing the track');
          }

          this.player.playing = true;
        } else {
          // PUT to pause the track if it is currently playing
          const pauseResponse = await fetch(`${this.endpoints.base}/me/player/pause`, {
            method: 'PUT', // Use PUT to pause
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`,
              'Content-Type': 'application/json'
            }
          });

          if (!pauseResponse.ok) {
            throw new Error('Error pausing the track');
          }

          this.player.playing = false;
        }
      } catch (error) {
        console.error(error);
      }
    },

    async nextTrack() {
      try {
        const response = await fetch(`${this.endpoints.base}/${this.endpoints.next}`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`
          }
        });
        if (!response.ok) {
          throw new Error('Error skipping to the next track');
        }
      } catch (error) {
        console.error(error);
      }
    },

    async previousTrack() {
      try {
        const response = await fetch(`${this.endpoints.base}/${this.endpoints.previous}`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`
          }
        });
        if (!response.ok) {
          throw new Error('Error going to the previous track');
        }
      } catch (error) {
        console.error(error);
      }
    }
  },

  watch: {
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying);
      }
    },

    playerResponse: function() {
      this.handleNowPlaying();
    },

    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData);
      this.$nextTick(() => {
        this.getAlbumColours();
      });
    }
  }
};
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>