<template>
  <canvas :class="className" canvas-id="k-canvas" :style="computedStyle" />
</template>

<script>
import Pen from "./lib/pen";
import Downloader from "./lib/downloader";

const util = require("./lib/util");

const downloader = new Downloader();

// 最大尝试的绘制次数
const MAX_PAINT_COUNT = 5;

export default {
  props: {
    class: {
      type: String,
      default: ""
    },
    customStyle: {
      type: String,
      default: ""
    },
    palette: {
      type: Object,
      default() {
        return {};
      }
    },
    // 启用脏检查，默认 false
    dirty: {
      type: Boolean,
      default: false
    }
  },
  computed: {
    className() {
      return this.class;
    },
    computedStyle() {
      let str = this.painterStyle + this.customStyle;
      // for (let i in this.customStyle) {
      // str += i + ':' + this.customStyle[i] + ';'
      // }

      return str;
    }
  },
  data() {
    return {
      canvasWidthInPx: 0,
      canvasHeightInPx: 0,
      paintCount: 0,
      picURL: "",
      showCanvas: true,
      painterStyle: ""
    };
  },
  watch: {
    palette(newVal, oldVal) {
      console.log(newVal, oldVal, "?newVal, oldVal");
      this.judegStart(newVal, oldVal, false);
    }
  },
  created() {
    setStringPrototype();
  },
  mounted() {
    if (Object.keys(this.palette).length) {
      this.judegStart({}, {}, true);
    }
  },
  methods: {
    /**
     * 判断一个 object 是否为 空
     * @param {object} object
     */
    isEmpty(object) {
      for (const i in object) {
        return false;
      }
      return true;
    },
    judegStart(newVal, oldVal, mode) {
      console.log(43634643);
      if (this.isNeedRefresh(newVal, oldVal) || mode) {
        this.paintCount = 0;
        this.startPaint();
      }
    },
    isNeedRefresh(newVal, oldVal) {
      if (
        !newVal ||
        this.isEmpty(newVal) ||
        (this.dirty && util.equal(newVal, oldVal))
      ) {
        return false;
      }
      return true;
    },

    startPaint() {
      console.log("startPaint");
      if (this.isEmpty(this.palette)) {
        return;
      }

      if (!(getApp().systemInfo && getApp().systemInfo.screenWidth)) {
        try {
          getApp().systemInfo = uni.getSystemInfoSync();
        } catch (e) {
          const error = `Painter get system info failed, ${JSON.stringify(e)}`;
          this.$emit("imgErr", { error: error });
          console.error(error);
          return;
        }
      }
      screenK = getApp().systemInfo.screenWidth / 750;

      this.downloadImages().then(palette => {
        const { width, height } = palette;
        this.canvasWidthInPx = width.toPx();
        this.canvasHeightInPx = height.toPx();
        if (!width || !height) {
          console.error(
            `You should set width and height correctly for painter, width: ${width}, height: ${height}`
          );
          return;
        }
        this.painterStyle = `width:${width};height:${height};`;
        const ctx = uni.createCanvasContext("k-canvas", this);
        const pen = new Pen(ctx, palette);
        this.ctx = ctx;
        pen.paint(() => {
          this.saveImgToLocal();
        });
      });
    },

    downloadImages() {
      return new Promise((resolve, reject) => {
        let preCount = 0;
        let completeCount = 0;
        const paletteCopy = JSON.parse(JSON.stringify(this.palette));
        if (paletteCopy.background) {
          preCount++;
          downloader.download(paletteCopy.background).then(
            path => {
              paletteCopy.background = path;
              completeCount++;
              if (preCount === completeCount) {
                resolve(paletteCopy);
              }
            },
            () => {
              completeCount++;
              if (preCount === completeCount) {
                resolve(paletteCopy);
              }
            }
          );
        }
        if (paletteCopy.views) {
          for (const view of paletteCopy.views) {
            if (view && view.type === "image" && view.url) {
              preCount++;
              /* eslint-disable no-loop-func */
              downloader.download(view.url).then(
                path => {
                  view.url = path;
                  uni.getImageInfo({
                    src: view.url,
                    success: res => {
                      // 获得一下图片信息，供后续裁减使用
                      view.sWidth = res.width;
                      view.sHeight = res.height;
                    },
                    fail: error => {
                      // 如果图片坏了，则直接置空，防止坑爹的 canvas 画崩溃了
                      view.url = "";
                      console.error(
                        `getImageInfo ${view.url} failed, ${JSON.stringify(
                          error
                        )}`
                      );
                    },
                    complete: () => {
                      completeCount++;
                      if (preCount === completeCount) {
                        resolve(paletteCopy);
                      }
                    }
                  });
                },
                () => {
                  completeCount++;
                  if (preCount === completeCount) {
                    resolve(paletteCopy);
                  }
                }
              );
            }
          }
        }
        if (preCount === 0) {
          resolve(paletteCopy);
        }
      });
    },

    saveImgToLocal() {
      const that = this;
      console.log("saveImgToLocal");
      setTimeout(() => {
        // #ifndef MP-ALIPAY
        uni.canvasToTempFilePath(
          {
            canvasId: "k-canvas",
            success: function(res) {
              that.getImageInfo(res.tempFilePath);
            },
            fail: function(error) {
              console.error(
                `canvasToTempFilePath failed, ${JSON.stringify(error)}`
              );
              that.$emit("imgErr", { error: error });
            }
          },
          this
        );
        // #endif

        // #ifdef MP-ALIPAY
          console.log("this.ctx", this.ctx)
          this.ctx.toTempFilePath({
            success(e) {
              console.log(e, "toTempFilePath")
              that.getImageInfo(e.tempFilePath);
            },
            fail(e){
              console.log(e, "toTempFilePath fail")
            }
          })
        // #endif
      }, 300);
    },

    getImageInfo(filePath) {
      const that = this;
      uni.getImageInfo({
        src: filePath,
        success: infoRes => {
          if (that.paintCount > MAX_PAINT_COUNT) {
            const error = `The result is always fault, even we tried ${MAX_PAINT_COUNT} times`;
            console.error(error);
            that.$emit("imgErr", { error: error });
            return;
          }
          // 比例相符时才证明绘制成功，否则进行强制重绘制
          if (
            Math.abs(
              (infoRes.width * that.canvasHeightInPx -
                that.canvasWidthInPx * infoRes.height) /
                (infoRes.height * that.canvasHeightInPx)
            ) < 0.01
          ) {
            that.$emit("imgOK", { target: { path: filePath } });
          } else {
            that.startPaint();
          }
          that.paintCount++;
        },
        fail: error => {
          console.error(`getImageInfo failed, ${JSON.stringify(error)}`);
          that.$emit("imgErr", { error: error });
        }
      });
    }
  }
};

let screenK = 0.5;

function setStringPrototype() {
  /* eslint-disable no-extend-native */
  /**
   * 是否支持负数
   * @param {Boolean} minus 是否支持负数
   */
  String.prototype.toPx = function toPx(minus) {
    let reg;
    if (minus) {
      reg = /^-?[0-9]+([.]{1}[0-9]+){0,1}(rpx|px)$/g;
    } else {
      reg = /^[0-9]+([.]{1}[0-9]+){0,1}(rpx|px)$/g;
    }
    const results = reg.exec(this);
    if (!this || !results) {
      console.error(`The size: ${this} is illegal`);
      return 0;
    }
    const unit = results[2];
    const value = parseFloat(this);

    let res = 0;
    if (unit === "rpx") {
      res = Math.round(value * screenK);
    } else if (unit === "px") {
      res = value;
    }
    return res;
  };
}
</script>

<style>
</style>