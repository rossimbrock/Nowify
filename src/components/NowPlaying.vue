<template>
  <div id="app">
    <div class="now-playing" :class="getNowPlayingClass()" :style="nowPlayingStyle">
      <!-- Time Display -->
      <div class="now-playing__time" :style="{ color: timeTextColor }" style="font-size: 1rem; font-weight: 400; margin-top: 0.5rem; opacity: 0.6;">{{ currentTime }}</div>

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
import * as Vibrant from 'node-vibrant'

import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player,
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      currentTime: '',
      previousAlbumImage: '',  // Track the previous album image for comparison
      timeTextColor: 'white',  // Default time text color
    };
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(', ');
    },
    nowPlayingStyle() {
      // Dynamically set the background style
      return {
        background: this.colourPalette.length ? `linear-gradient(135deg, ${this.colourPalette[0]} 0%, ${this.colourPalette[1]} 100%)` : ''
      };
    }
  },

  mounted() {
    this.setDataInterval();
    this.updateTime();
    setInterval(this.updateTime, 60000); // Update time every minute
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    updateTime() {
      const now = new Date();
      const hours = now.getHours() % 12 || 12;
      const minutes = now.getMinutes().toString().padStart(2, '0');
      const period = now.getHours() >= 12 ? 'PM' : 'AM';
      this.currentTime = `${hours}:${minutes} ${period}`;
    },
    
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
        if (!response.ok) {
          throw new Error(`An error has occurred: ${response.status}`)
        }
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

    getNowPlayingClass() {
      const playerClass = this.player.trackTitle ? (this.player.playing ? 'active' : 'paused') : 'idle'
      return `now-playing--${playerClass}`
    },

    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return
      }

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 1000)
    },

    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette[0] === '#FFFFFF' ? 'black' : 'white'
      )
      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette[0]
      )
    },

    handleNowPlaying() {
      if (this.playerResponse.error?.status === 401 || this.playerResponse.error?.status === 400) {
        this.handleExpiredToken()
        return
      }
      if (this.playerResponse.is_playing === false) {
        return
      }
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(artist => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images.find(image => image.height === 640 && image.width === 640)?.url || this.playerResponse.item.album.images[0].url
        }
      }
    },

    handleAlbumPalette(palette) {
      const albumColours = Object.keys(palette)
        .filter(item => item !== null)
        .map(colour => ({
          text: palette[colour].getTitleTextColor(),
          background: palette[colour].getHex(),
          population: palette[colour].getPopulation()
        }));

      // Filter out tan or neutral colors
      const filteredColours = albumColours.filter(colour => !this.isTanColor(colour.background));

      // Sort by population to get the most prominent colors
      filteredColours.sort((a, b) => b.population - a.population);

      let primaryColor = filteredColours[0]?.background || '#FFFFFF';
      let secondaryColor = filteredColours[1]?.background || '#FFFFFF';

      if (primaryColor && secondaryColor) {
        this.colourPalette = [primaryColor, secondaryColor];
      } else {
        this.colourPalette = [primaryColor, filteredColours[2]?.background || '#FFFFFF'];
      }

      this.updateTextColor(this.colourPalette[0]);
      this.updateTimeColor(this.colourPalette[0]);

      this.previousAlbumImage = this.player.trackAlbum.image;

      this.$nextTick(() => {
        this.setAppColours();
      });
    },

    // Function to check if a color is a tan or neutral-like color
    isTanColor(hex) {
      const rgb = this.hexToRgb(hex);
      const r = rgb.r;
      const g = rgb.g;
      const b = rgb.b;

      // Check if the color is close to tan or neutral (common tan RGB values: 194, 178, 128)
      // We'll allow a tolerance range to catch similar colors
      return (
        (r >= 180 && r <= 220) &&
        (g >= 160 && g <= 200) &&
        (b >= 100 && b <= 160)
      );
    },

    // Convert hex color to RGB format
    hexToRgb(hex) {
      const r = parseInt(hex.substring(1, 3), 16);
      const g = parseInt(hex.substring(3, 5), 16);
      const b = parseInt(hex.substring(5, 7), 16);
      return { r, g, b };
    },

    updateTextColor(backgroundColor) {
      const luminance = this.getLuminance(backgroundColor);
      this.timeTextColor = luminance < 0.5 ? 'white' : 'black';
    },

    getLuminance(hex) {
      const rgb = this.hexToRgb(hex);
      const a = [rgb.r / 255, rgb.g / 255, rgb.b / 255].map(function (v) {
        return v <= 0.03928 ? v / 12.92 : Math.pow((v + 0.055) / 1.055, 2.4);
      });
      const luminance = 0.2126 * a[0] + 0.7152 * a[1] + 0.0722 * a[2];
      return luminance;
    },

    updateTimeColor(backgroundColor) {
      const luminance = this.getLuminance(backgroundColor);
      this.timeTextColor = luminance < 0.5 ? 'white' : 'black';
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    },

    async togglePlayPause() {
      try {
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
          const playResponse = await fetch(`${this.endpoints.base}/me/player/play`, {
            method: 'PUT',
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
          const pauseResponse = await fetch(`${this.endpoints.base}/me/player/pause`, {
            method: 'PUT',
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
        })
        if (!response.ok) {
          throw new Error('Error skipping to the next track')
        }
      } catch (error) {
        console.error(error)
      }
    },

    async previousTrack() {
      try {
        const response = await fetch(`${this.endpoints.base}/${this.endpoints.previous}`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`
          }
        })
        if (!response.ok) {
          throw new Error('Error skipping to the previous track')
        }
      } catch (error) {
        console.error(error)
      }
    },
  },
};
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>