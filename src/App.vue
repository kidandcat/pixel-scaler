<template>
  <div class="wrapper">
    <main>
      <h1>{{$t('title')}}</h1>

      <nav>
        <div class="row">
          <div class="col">
            <div class="top-label">{{$t('original-pixel-size')}}</div>
            <label class="radio box hover active flex-item" :class="{on: pixelSize.org == pixel}" v-for="pixel in pixelSize.list" :key="pixel">
              <input type="radio" v-model.number="pixelSize.org" :value="pixel"> {{pixel}}px
            </label>
          </div>

          <div class="col">
            <div class="top-label">{{$t('scale')}}(%)</div>
            <input class="col" type="number" inputmode="decimal" v-model.number="size" step="5" min="100" max="400" :placeholder="$t('scale')">
          </div>
        </div>

        <div class="row">
          <label class="col-big box circle hover active pointer" @dragover.prevent @drop.prevent="setFiles">
            <input type="file" accept="image/png, image/jpeg, image/gif" multiple @change="setFiles">
            <v-fa :icon="['far', 'file-image']"/> {{files.length ? $t('select', {count: files.length}) : $t('no-select')}}
          </label>

          <div class="col box circle hover active pointer" @click="convert">
            <v-fa icon="reply" flip="vertical"/> {{$t('convert')}}
          </div>
        </div>

      </nav>

      <div class="content margin-tb-1">

        <attention-container v-if="flags.showAttention" @close="flags.showAttention = false"/>

        <div class="box block margin-tb-2" v-show="errors.length !== 0">
          <v-fa icon="times-circle" class="close-btn pointer" @click="errors = []"/>
          <ul>
            <li v-for="error in errors" :key="error.id">{{error}}</li>
          </ul>
        </div>

        <div class="box block">
          <exception-container v-if="exception" @close="exception = null"
                               :exception="exception"/>

          <howto-container v-else-if="converted.length === 0"/>

          <template v-else>
            <div class="btn-list">
              <div class="col box circle hover active pointer margin-lr-1" @click="download">
                <v-fa :icon="['far', 'file-archive']"/> {{$t('download-zip')}}
              </div>
              <div class="col box circle hover active pointer margin-lr-1" @click="resetConverted">
                <v-fa icon="eraser"/> {{$t('reset')}}
              </div>

              <Loading v-if="flags.convert"/>

            </div>

            <image-container class="margin-tb-1"
                             v-for="img in converted" :key="img.image.filename"
                             :org="img.org" :converted="img.image"/>

          </template>

        </div>

      </div>

      <language-container v-model="$i18n.locale"/>

      <link-container class="margin-tb-4" v-if="isWeb()"/>

      <version-container class="margin-tb-4"
                         v-if="isElectron() && flags.showVersionContainer"
                         @close="flags.showVersionContainer = false"
                         :checkUpdate="flags.checkUpdate" :isLatest="isLatest()" :latestVersion="latestVersion"/>

    </main>

    <footer v-if="isWeb()">
      (C) {{year()}} ののの茶屋.
    </footer>
  </div>
</template>

<script>
import Archive      from './lib/Archive';
import FileUtil     from './lib/FileUtil';
import PictureScale from './lib/PictureScale';
import System       from './lib/System';
import Version      from './lib/Version';

import Loading          from './components/Loading.vue';
import AttentionContainer from './components/AttentionContainer.vue';
import HowtoContainer   from './components/HowtoContainer.vue';
import ImageContainer   from './components/ImageContainer.vue';
import LanguageContainer from './components/LanguageContainer.vue';
import LinkContainer    from './components/LinkContainer.vue';
import VersionContainer from './components/VersionContainer.vue';
import ExceptionContainer from './components/ExceptionContainer.vue';

export default {
  name: 'app',
  data () {
    return {
      pixelSize: {
        list: [1, 2, 3, 4],
        org: 1,
      },
      size: 200,
      files: [],
      converted: [],
      errors: [],
      zip: null,
      exception: null,
      latestVersion: '',
      flags: {
        convert: false,
        checkUpdate: false,
        showAttention: true,
        showVersionContainer: true,
      },
    };
  },
  methods: {
    /**
     * ファイル配列を変数に入れるやつ
     * @param {Event} e
     * @returns {void}
     */
    setFiles(e) {
      this.files = FileUtil.getFileListOnEvent(e);

      this.converted = [];
      this.zip       = null;
      this.exception = null;
    },

    /**
     * 実際に拡大縮小するやつ
     * @returns {void}
     */
    async convert() {
      this.converted = [];
      this.errors    = [];
      this.exception = null;
      this.size      = PictureScale.adjustSize(this.size);

      if (this.flags.convert || this.files.length === 0) {
        return;
      }

      this.flags.convert = true;

      for (const file of this.files) {
        await PictureScale.scale(file, this.size, this.pixelSize.org).then(scaled => {
          if (scaled.status === 'success') {
            this.converted.push(scaled);
          } else {
            for (const err of Object.values(scaled.messages)) {
              this.errors = this.errors.concat(err);
            }
          }
        }).catch(e => {
          this.exception = e;
        });

        if (this.exception) {
          this.flags.convert = false;
          return;
        }
      }

      const files = [];

      for (const scaled of this.converted) {
        files.push(scaled.image);
      }

      await Archive.ScaledImagestoZip(files).then(content => {
        this.zip = content;
      }).catch(e => {
        this.exception = e;
      });

      this.flags.convert = false;
    },

    /**
     * zipをダウンロードするやつ
     * @returns {void}
     */
    download() {
      if (this.flags.convert) {
        return;
      }

      Archive.download(this.zip, 'images.zip');
    },

    /**
     * コンバート関連変数の初期化
     * @returns {void}
     */
    resetConverted() {
      if (this.flags.convert) {
        return;
      }

      this.files     = [];
      this.converted = [];
      this.errors    = [];
      this.zip       = null;
    },

    /**
     * 現在が最新版かどうか
     * @returns {boolean}
     */
    isLatest() {
      return this.flags.checkUpdate && this.latestVersion !== '';
    },

    /**
     * ウェブかどうか
     * @returns {boolean}
     */
    isWeb () {return System.isWeb()},

    /**
     * electronかどうか
     * @returns {boolean}
     */
    isElectron() {return System.isElectron()},

    /**
     * バージョンアップが必要かどうか
     * @returns {Promise<string>}
     */
    checkUpdate() {return Version.check()},

    /**
     * 今年の年を返す
     * @returns {number}
     */
    year() {return (new Date()).getFullYear();}
  },
  async created () {
    document.title = this.$t('title');

    if (this.isElectron()) {
      this.latestVersion = await this.checkUpdate();
      this.flags.checkUpdate = true;
    }
  },
  components: {
    Loading,
    AttentionContainer,
    HowtoContainer,
    ImageContainer,
    LanguageContainer,
    LinkContainer,
    VersionContainer,
    ExceptionContainer,
  },
}
</script>
