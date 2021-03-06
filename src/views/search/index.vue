<template>
  <div>
    <cover-row-skeleton
      v-show="loading"
    />
    <page
      v-show="!loading"
      title="搜索结果:"
    >
      <custom-col
        v-show="songs.length"
        title="歌曲"
        more="/search/song"
      >
        <template slot="content">
          <v-row>
            <v-col
              v-for="song in songs"
              :key="song.id"
              cols="3"
              class="pa-2"
            >
              <song-bar
                :song="song"
                class="track-item"
              />
            </v-col>
          </v-row>
        </template>
      </custom-col>
      <custom-col
        v-show="artists.length"
        title="歌手"
        more="/search/artist"
      >
        <template slot="content">
          <cover-list
            type="artist"
            :list="artists"
          />
        </template>
      </custom-col>
      <custom-col
        v-show="albums.length"
        title="专辑"
        more="/search/album"
      >
        <template slot="content">
          <cover-list
            :list="albums"
          />
        </template>
      </custom-col>
      <custom-col
        v-show="playlists.length"
        title="歌单"
        more="/search/playlist"
      >
        <template slot="content">
          <cover-list
            type="playlist"
            :list="playlists"
          />
        </template>
      </custom-col>
      <custom-col
        v-show="mvs.length"
        title="视频"
        more="/search/mv"
      >
        <template slot="content">
          <v-row>
            <v-col
              v-for="mv in mvs"
              :key="mv.id"
              cols="3"
              class="pa-2"
            >
              <video-cover :data="mv" />
            </v-col>
          </v-row>
        </template>
      </custom-col>
    </page>
  </div>
</template>

<script>
import CustomCol from '@components/layout/Col'
import {search} from '@/api/music'
import CoverList from '@components/app/CoverList'
import SongBar from '@components/app/SongBar'
import VideoCover from '@components/app/VideoCover'
import Page from '@components/layout/Page'
import NProgress from 'nprogress'
import CoverRowSkeleton from '@components/skeleton/coverRowSkeleton'
const TypeMap = {
  'song': { type: 1, limit: 8 },
  'album': { type: 10, limit: 6 },
  'artist': { type: 100, limit: 4 },
  'playlist': { type: 1000, limit: 6 },
  'mv': { type: 1004, limit: 4 },
}
export default {
  name: 'Search',
  components: { CoverRowSkeleton, Page, VideoCover, SongBar, CoverList, CustomCol },
  props: {
    keywords: String,
  },
  data: () => ({
    loading: false,
    artists: [],
    songs: [],
    albums: [],
    playlists: [],
    mvs: [],
  }),
  computed: {

  },
  watch: {
    keywords(val) {
      if (val) {
        this.search();
      }
    },
  },
  created () {
    this.search();
  },
  methods: {
    async search() {
      NProgress.start();
      this.loading = true;
      const requests = Object.entries(TypeMap).map(([k, v]) => {
        return search(this.keywords, {type: v.type, limit: v.limit });
      });
      const [songs, albums, artists, playlists, mvs] = await Promise.all(requests);
      this.songs = songs?.result?.songs ?? [];
      this.albums = albums?.result?.albums ?? [];
      this.artists = artists?.result?.artists ?? [];
      this.playlists = playlists?.result?.playlists ?? [];
      this.mvs = mvs?.result?.mvs ?? [];
      this.loading = false;
      NProgress.done();
    },
  },
}
</script>

<style scoped>

</style>
