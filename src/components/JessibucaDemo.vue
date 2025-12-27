<template>
  <div class="root">
    <div class="container-shell">
      <div class="container-shell-title">jessibuca demo player <span class="tag-version" v-if="version">({{
          version
        }})</span></div>
      <div class="option">
        <span>缓冲(秒):</span>
        <input
            style="width: 50px"
            type="number"
            ref="buffer"
            value="0.5"
            @change="changeBuffer"
        />
        <input
            type="checkbox"
            v-model="useMSE"
            ref="vod"
            @change="restartPlay('mse')"
        /><span>MediaSource</span>
        <input
            type="checkbox"
            v-model="useWCS"
            ref="vod"
            @change="restartPlay('wcs')"
        /><span>webcodecs</span>

      </div>
      <div id="container" ref="container"></div>
      <div class="err" v-if="err">{{ err }}</div>
      <div class="input">
        <div>输入URL：</div>
        <input
            type="input"
            autocomplete="on"
            ref="playUrl"
            value=""
        />
        <button v-if="!playing" @click="play">播放</button>
        <button v-else @click="pause">停止</button>
      </div>
      <div class="input" v-if="loaded" style="line-height: 30px">
        <button @click="destroy">销毁</button>
        <button v-if="quieting" @click="cancelMute">取消静音</button>
        <template v-else>
          <button @click="mute">静音</button>
          音量
          <select v-model="volume" @change="volumeChange">
            <option value="1">100</option>
            <option value="0.75">75</option>
            <option value="0.5">50</option>
            <option value="0.25">25</option>
          </select>
        </template>
        <span>旋转</span>
        <select v-model="rotate" @change="rotateChange">
          <option value="0">0</option>
          <option value="90">90</option>
          <option value="270">270</option>
        </select>

        <button @click="fullscreen">全屏</button>
        <button @click="screenShot">截图</button>
        <div style="line-height: 30px">
          <input
              type="checkbox"
              ref="operateBtns"
              v-model="showOperateBtns"
              @change="restartPlay"
          /><span>操作按钮</span>
          <input
              type="checkbox"
              ref="operateBtns"
              v-model="showBandwidth"
              @change="restartPlay"
          /><span>网速</span>
          <span v-if="performance">性能：{{ performance }}</span>
        </div>
      </div>
      <div class="input" v-if="loaded">
        <input
            type="checkbox"
            ref="offscreen"
            v-model="useOffscreen"
            @change="restartPlay('offscreen')"
        /><span>离屏渲染</span>

        <select v-model="scale" @change="scaleChange">
          <option value="0">完全填充(拉伸)</option>
          <option value="1">等比缩放</option>
          <option value="2">完全填充(未拉伸)</option>
        </select>
        <button v-if="!playing" @click="clearView">清屏</button>
        <template v-if="playing">
          <select v-model="recordType">
            <option value="webm">webm</option>
            <option value="mp4">mp4</option>
          </select>
          <button v-if="!recording" @click="startRecord">录制</button>
          <button v-if="!recording" @click="stopAndSaveRecord">暂停录制</button>
        </template>

      </div>
    </div>
  </div>
</template>
<script>

export default {
  name: "DemoPlayer",
  props: {},
  data() {
    return {
      jessibuca: null,
      version: '',
      wasm: false,
      vc: "ff",
      playing: false,
      quieting: true,
      loaded: false, // mute
      showOperateBtns: false,
      showBandwidth: false,
      err: "",
      speed: 0,
      performance: "",
      volume: 1,
      rotate: 0,
      useWCS: false,
      useMSE: true,
      useOffscreen: false,
      recording: false,
      recordType: 'webm',
      scale: 0,
      // 重连机制相关状态
      reconnectCount: 0,
      maxReconnectCount: 5,
      reconnectTimeout: null,
      baseReconnectInterval: 2000
    };
  },
  mounted() {
    this.create();
    window.onerror = (msg) => (this.err = msg);
  },
   async unmounted() {
    // 清除重连计时器
    this.clearReconnectTimeout();
    if(this.jessibuca){
      await this.jessibuca.destroy();
      this.jessibuca = null;
    }
    // 清理全局错误监听
    window.onerror = null;
  },
  methods: {
    create(options) {
      options = options || {};
      this.jessibuca = new window.Jessibuca(
          Object.assign(
              {
                container: this.$refs.container,
                videoBuffer: Number(this.$refs.buffer.value), // 缓存时长
                isResize: false,
                useWCS: this.useWCS,
                useMSE: this.useMSE,
                text: "",
                // background: "bg.jpg",
                loadingText: "疯狂加载中...",
                // hasAudio:false,
                debug: true,
                supportDblclickFullscreen: true,
                showBandwidth: this.showBandwidth, // 显示网速
                operateBtns: {
                  fullscreen: this.showOperateBtns,
                  screenshot: this.showOperateBtns,
                  play: this.showOperateBtns,
                  audio: this.showOperateBtns,
                },
                vod: this.vod,
                forceNoOffscreen: !this.useOffscreen,
                isNotMute: true,
                timeout: 10
              },
              options
          )
      );
      var _this = this;
      this.jessibuca.on("load", function () {
        console.log("on load");
      });

      this.jessibuca.on("log", function (msg) {
        console.log("on log", msg);
      });
      this.jessibuca.on("record", function (msg) {
        console.log("on record:", msg);
      });
      this.jessibuca.on("pause", function () {
        console.log("on pause");
        _this.playing = false;
      });
      this.jessibuca.on("play", function () {
        console.log("on play");
        _this.playing = true;
      });
      this.jessibuca.on("fullscreen", function (msg) {
        console.log("on fullscreen", msg);
      });

      this.jessibuca.on("mute", (msg) => {
        console.log("on mute", msg);
        this.quieting = msg;
      });

      this.jessibuca.on("audioInfo", function (msg) {
        console.log("audioInfo", msg);
      });

      // this.jessibuca.on("bps", function (bps) {
      //   // console.log('bps', bps);
      // });
      // let _ts = 0;
      // this.jessibuca.on("timeUpdate", function (ts) {
      //     console.log('timeUpdate,old,new,timestamp', _ts, ts, ts - _ts);
      //     _ts = ts;
      // });

      this.jessibuca.on("videoInfo", function (info) {
        console.log("videoInfo", info);
      });

      this.jessibuca.on("error", (error) => {
        console.log("error", error);
        this.err = `播放错误: ${error.message || error}`;
        this.handleReconnect();
      });

      this.jessibuca.on("timeout", () => {
        console.log("timeout");
        this.handleReconnect();
      });
      
      this.jessibuca.on("loadingTimeout", () => {
        console.log("加载超时，可提示用户或重试");
        this.err = "加载超时，请检查网络或流地址";
        this.handleReconnect();
      });

      this.jessibuca.on('start', function () {
        console.log('frame start');
      })

      this.jessibuca.on("performance", function (performance) {
        var show = "卡顿";
        if (performance === 2) {
          show = "非常流畅";
        } else if (performance === 1) {
          show = "流畅";
        }
        _this.performance = show;
      });
      this.jessibuca.on('buffer', (buffer) => {
        console.log('buffer', buffer);
        // 缓冲区状态处理
        if (buffer < 0.1) {
          console.log('缓冲区不足，考虑增加缓存');
          // 可以根据缓冲区状态动态调整缓存时间
          const currentBuffer = Number(this.$refs.buffer.value);
          if (currentBuffer < 2) {
            const newBuffer = Math.min(currentBuffer + 0.5, 2);
            this.$refs.buffer.value = newBuffer;
            this.jessibuca.setBufferTime(newBuffer);
          }
        }
      })

      this.jessibuca.on('stats', (stats) => {
        console.log('stats', stats);
      })

      this.jessibuca.on('kBps', (kBps) => {
        console.log('kBps', kBps);
        // 带宽变化处理
        this.speed = kBps;
        // 根据带宽动态调整缓存策略
        if (kBps < 500) {
          console.log('低带宽，增加缓存');
          const currentBuffer = Number(this.$refs.buffer.value);
          if (currentBuffer < 2) {
            const newBuffer = Math.min(currentBuffer + 0.5, 2);
            this.$refs.buffer.value = newBuffer;
            this.jessibuca.setBufferTime(newBuffer);
          }
        } else if (kBps > 2000) {
          console.log('高带宽，减少缓存以降低延迟');
          const currentBuffer = Number(this.$refs.buffer.value);
          if (currentBuffer > 0.5) {
            const newBuffer = Math.max(currentBuffer - 0.5, 0.5);
            this.$refs.buffer.value = newBuffer;
            this.jessibuca.setBufferTime(newBuffer);
          }
        }
      });

      this.jessibuca.on("play", () => {
        this.playing = true;
        this.loaded = true;
        this.quieting = this.jessibuca.isMute();
        // 播放成功，重置重连状态
        this.resetReconnectState();
      });

      this.jessibuca.on('recordingTimestamp', (ts) => {
        console.log('recordingTimestamp', ts);
      })


      // console.log(this.jessibuca);
    },
    play() {
      // this.jessibuca.onPlay = () => (this.playing = true);


      if (this.$refs.playUrl.value) {
        this.jessibuca.play(this.$refs.playUrl.value);
      }
    },
    mute() {
      this.jessibuca.mute();
    },
    cancelMute() {
      this.jessibuca.cancelMute();
    },

    pause() {
      this.jessibuca.pause();
      this.playing = false;
      this.err = "";
      this.performance = "";
      // 停止播放时，清除重连计时器
      this.clearReconnectTimeout();
      this.resetReconnectState();
    },
    volumeChange() {
      this.jessibuca.setVolume(this.volume);
    },
    rotateChange() {
      this.jessibuca.setRotate(this.rotate);
    },
    async destroy() {
      if (this.jessibuca) {
        await this.jessibuca.destroy();
      }
      this.create();
      this.playing = false;
      this.loaded = false;
      this.performance = "";
    },

    fullscreen() {
      this.jessibuca.setFullscreen(true);
    },

    clearView() {
      this.jessibuca.clearView();
    },

    startRecord() {
      const time = new Date().getTime();
      this.jessibuca.startRecord(time, this.recordType);
    },

    stopAndSaveRecord() {
      this.jessibuca.stopRecordAndSave();
    },


    screenShot() {
      this.jessibuca.screenshot();
    },


    async restartPlay(type) {

      if (type === 'mse') {
        this.useWCS = false;
        this.useOffscreen = false;
      } else if (type === 'wcs') {
        this.useMSE = false
      } else if (type === 'offscreen') {
        this.useMSE = false
      }

      await this.destroy();
      setTimeout(() => {
        this.play();
      }, 100)
    },

    changeBuffer() {
      this.jessibuca.setBufferTime(Number(this.$refs.buffer.value));
    },

    // 处理重连逻辑
    handleReconnect() {
      if (!this.playing || this.reconnectCount >= this.maxReconnectCount) {
        return;
      }
      
      this.reconnectCount++;
      // 指数退避策略：2^n * 1000ms，最大不超过30秒
      const reconnectInterval = Math.min(
        Math.pow(2, this.reconnectCount) * this.baseReconnectInterval,
        30000
      );
      
      console.log(`尝试重连 ${this.reconnectCount}/${this.maxReconnectCount}，间隔 ${reconnectInterval}ms`);
      
      this.clearReconnectTimeout();
      this.reconnectTimeout = setTimeout(() => {
        if (this.playing && this.$refs.playUrl.value) {
          this.jessibuca.play(this.$refs.playUrl.value);
        }
      }, reconnectInterval);
    },
    
    // 清除重连计时器
    clearReconnectTimeout() {
      if (this.reconnectTimeout) {
        clearTimeout(this.reconnectTimeout);
        this.reconnectTimeout = null;
      }
    },
    
    // 重置重连状态
    resetReconnectState() {
      this.reconnectCount = 0;
      this.clearReconnectTimeout();
    },
    
    scaleChange() {
      this.jessibuca.setScaleMode(this.scale);
    },
  },
};
</script>
<style>
.root {
  display: flex;
  place-content: center;
  margin-top: 3rem;
}

.container-shell {
  position: relative;
  backdrop-filter: blur(5px);
  background: hsla(0, 0%, 50%, 0.5);
  padding: 30px 4px 10px 4px;
  /* border: 2px solid black; */
  width: auto;
  position: relative;
  border-radius: 5px;
  box-shadow: 0 10px 20px;
}

.container-shell-title {
  position: absolute;
  color: darkgray;
  top: 4px;
  left: 10px;
  text-shadow: 1px 1px black;
}

.tag-version {
}

#container {
  background: rgba(13, 14, 27, 0.7);
  width: 640px;
  height: 398px;
}

.input {
  display: flex;
  align-items: center;
  margin-top: 10px;
  color: white;
  place-content: stretch;
}

.input2 {
  bottom: 0px;
}

.input input[type='input'] {
  flex: auto;
}

.err {
  position: absolute;
  top: 40px;
  left: 10px;
  color: red;
}

.option {
  position: absolute;
  top: 4px;
  right: 10px;
  display: flex;
  place-content: center;
  font-size: 12px;
}

.option span {
  color: white;
}


@media (max-width: 720px) {
  #container {
    width: 90vw;
    height: 52.7vw;
  }
}
</style>
