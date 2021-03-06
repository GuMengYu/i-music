<template>
  <v-sheet
    class="frame"
    dark
  >
    <div
      class="frame-header mt-6"
      @click="close"
    />
    <v-row class="frame-content d-flex align-center">
      <v-col
        class="frame-content-left d-flex justify-center align-center"
      >
        <div class="left-container">
          <v-card class="rounded-lg album-cover">
            <v-img
              class="cover-img"
              :src="albumPicUrl"
            />
          </v-card>
          <div class="control_bar d-flex flex-column mt-12 justify-space-between">
            <div class="d-flex justify-space-between mt-4">
              <div class="left d-flex flex-column text-body-1 font-weight-bold">
                <span>{{ track.name }}</span>
                <span>{{ $ochain(track, 'ar', '0', 'name') }}</span>
              </div>
              <div class="right d-flex align-center">
                <v-btn icon>
                  <v-icon>
                    {{ icon.mdiDotsHorizontal }}
                  </v-icon>
                </v-btn>
              </div>
            </div>
            <div class="control_process mt-2">
              <vue-slider
                ref="vueLyricSlider"
                v-model="currentTime"
                class="playing-progress"
                :max="~~(track.dt / 1000) || 0"
                :min="0"
                :interval="1"
                :duration="0"
                :drag-on-click="true"
                :dot-size="10"
                :height="2"
                tooltip="none"
                @drag-end="handleSlideChange"
              />
              <div class="time-info d-flex justify-space-between text-caption">
                <span>{{ currentTime * 1000 | formatDuring }}</span>
                <span>{{ track.dt | formatDuring }}</span>
              </div>
            </div>
            <control />
            <div class="control_volume mt-4">
              <v-slider
                v-model="volume"
                class="playing-volume"
                dense
                hide-details
                :max="1"
                min="0"
                step="0.01"
                :append-icon="icon.mdiVolumeHigh"
                :prepend-icon="icon.mdiVolumeLow"
              />
            </div>
          </div>
        </div>
      </v-col>
      <v-col
        v-if="lyric.length"
        class="frame-content-right"
      >
        <div
          ref="lyricContainer"
          class="frame-lyrics"
        >
          <div class="first"></div>
          <div
            v-for="(item, index) in lyric"
            :key="index"
            :aria-time="item.time"
            :aria-index="index"
            :class="{active: index === activeIdx}"
            v-html="item.sentence"
          >
            {{ item.sentence }}
          </div>
          <div class="last mb-10"></div>
        </div>
      </v-col>
    </v-row>
    <div class="frame-bg">
      <v-img :src="albumPicUrl" class="bg-color album-artwork"></v-img>
      <v-img :src="albumPicUrl" class="bg-black album-artwork"></v-img>

<!--      <img class="bg-color album-artwork" :src="albumPicUrl" />-->
<!--      <img class="bg-black album-artwork" :src="albumPicUrl" />-->
    </div>
  </v-sheet>
</template>

<script>
import {mdiSkipPrevious, mdiDotsHorizontal, mdiPauseCircle, mdiSkipNext, mdiVolumeLow, mdiVolumeHigh, mdiShuffle, mdiRepeat} from '@mdi/js';
import { get, sync } from 'vuex-pathify'
import {formatLyric} from '@/util/fn';
import VueSlider from 'vue-slider-component';
import {findIndex} from 'lodash';
import Control from '@components/app/Control'
export default {
  name: 'DefaultTrackDetail',
  components: {
    Control,
    VueSlider},
  data: () => ({
    icon: {
      mdiDotsHorizontal,
      mdiVolumeLow,
      mdiVolumeHigh,
      mdiShuffle,
      mdiSkipPrevious,
      mdiSkipNext,
      mdiRepeat,
      mdiPauseCircle,
    },
    activeIdx: -1,
    interval: null,
  }),
  computed: {
    isCurrentFm: get('music/isCurrentFm'),
    albumPicUrl() {
      return `${this.track.al?.picUrl}?param=512y512`;
    },
    lyric() {
      const {tlyric,lrc} = this.track.lyric ?? {};
      let lyric = lrc?.lyric ?  formatLyric(lrc.lyric) : [];
      let _tlyric = tlyric?.lyric ? formatLyric(tlyric.lyric) : [];
      if (_tlyric.length) {
        return lyric.map(i => {
          const _t = _tlyric.find(t => t.time === i.time);
          return {
            sentence: `${i.sentence}${_t?.sentence ? `<br>${_t?.sentence}` : ''}`,
            time: i.time,
          }
        })
      } else {
        return lyric;
      }
    },
    volume: sync('settings/volume'),
    currentTime: sync('music/currentTime'),
    track: get('music/track'),
    showLyricsPage: sync('music/showLyricsPage'),
  },
  watch: {
    showLyricsPage(val) {
      if (val) {
        this.initInterval();
      } else {
        clearInterval(this.interval);
      }
    },
  },
  mounted () {
    this.initInterval();
  },
  destroyed () {
    clearInterval(this.interval);
  },
  methods: {
    initInterval() {
      this.interval = setInterval(() => {
        this.calculate();
      }, 200);
    },
    // 计算歌词位置并滚动
    calculate() {
      const current = this.currentTime;
      const prevActiveIdx = this.activeIdx;
      const activeIdx = findIndex(this.lyric, (o, idx) => {
        const next = this.lyric[idx + 1];
        return (next ? current < next.time : true) && current >= o.time
      });
      this.activeIdx = activeIdx;
      // 当前歌词渲染后计算滚动位置
      this.$nextTick(async () => {
        if (activeIdx >= 0 && prevActiveIdx !== activeIdx) {
          const container = this.$refs.lyricContainer;
          const activeEl = document.querySelector('.frame-lyrics .active');
          if (activeEl) {
            const newY = activeEl.offsetTop - activeEl.clientHeight * 2;
            const offset = await this.$vuetify.goTo(newY, { container });
            console.log('scroll to ' + offset);
          }
        }
      })
    },
    close() {
      this.$emit('close');
    },
    handleSlideChange() {
      this.currentTime = this.$refs['vueLyricSlider'].getValue();
    },
  },
};
</script>

<style lang="scss" scoped>
.frame {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
  overflow: hidden;
  position: relative;
  .frame-header {
    z-index: 1;
    display: flex;
    justify-content: center;
    width: 100%;
    &:before {
      content: '';
      display: block;
      background: #fff;
      filter: brightness(0.7);
      width: 4em;
      height: 0.3em;
      margin: 1em auto;
      z-index: 1;
      border-radius: 15em;
      mix-blend-mode: overlay;
      cursor: pointer;
    }
  }
  .frame-content {
    z-index: 1;
    height: calc(100% - 2.6em - 15px);
    .frame-content-left {
      flex: 1;
      .left-container {
        max-width: 30vw;
        //.album-cover {
        //  box-shadow: 4px 9px 20px #403e3e, -7px -6px 20px #565454;
        //}
      }
    }
    .frame-content-right {
      height: 85vh;
      flex: 1;
      .frame-lyrics {
        height: 100%;
        position: relative;
        z-index: 1;
        font-size: 1.7rem;
        overflow-y: auto;
        font-weight: bold;
        .active {
          font-size: 2.2rem;
        }
        .first {
          margin-top: 50%;
        }
        .last {
          margin-bottom: 50%;
        }
        & > div {
          & + div {
            transition: all .25s;
            margin-top: 0.8em;
          }
        }
        & > div:not(.active) {
          filter: blur(0.7px) brightness(0.8);
        }
      }
    }
  }
  .frame-bg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 0;
    overflow: hidden;
    .album-artwork {
      width: 200%;
      position: absolute;
      border-radius: 100em;
      animation: rotate 100s ease-in-out infinite;
      filter: blur(70px) brightness(0.5);
      mix-blend-mode: multiply;
      z-index: 1;
    }
    .bg-color {
      right: 0;
      top: 0;
    }
    .bg-black {
      left: 0;
      bottom: 0;
      animation-direction: reverse;
    }
  }
}
@keyframes rotate {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
