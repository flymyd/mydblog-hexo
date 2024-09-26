---
title: uniApp代码-仿微信相机
toc: true
date: 2022-08-02 00:00:00
tags: [uniApp, 代码片段]
categories:
  - [前端技术, 代码片段]
---
在微信小程序中直接使用uni.chooseMedia即可调用微信相机进行摄录。但为了保持体验的相似性，造一个仿微信相机的轮子是有必要的

因为小程序的胶囊很碍事，所以对UI进行了调整，功能按钮均移动到页面下方。

本相机支持照片及视频的摄录，拍摄后可进行预览、重拍。采用页面栈的形式进行传值。

* 使用例：ClockIn.vue（记得先在pages.json中注册）

  ```html
  <template>
  	<view>
      <button @click="doCapture">点击摄录</button>
    	<image v-if="photoSrc" :src="photoSrc"></image>
    </view>
  	<script>
    	export default {
        data() {
          return {
            photoSrc: ""
          }
        },
        methods: {
          doCapture(){
            uni.navigateTo({
            	url: '/pages/TakePhotoAndVideo?dest=photoSrc'
          	})
          }
        }
      }
    </script>
  </template>
  ```

* 源代码：TakePhotoAndVideo.vue

```html
<template>
  <view class="complex-camera">
    <video v-if="videoSrc" :src="videoSrc" style="width: 100vw; height:100vh;z-index: 9800;" autoplay="true" loop="true"
      :controls="false" object-fit="cover">
      <!-- 其它功能按钮，视频预览用 -->
      <cover-image :src="backIcon" class="complex-camera-back" @click="goBack"></cover-image>
      <cover-image :src="confirmIcon" class="complex-camera-previewer-success" @click="save"></cover-image>
    </video>
    <camera v-else device-position="back" @error="bindError" style="width: 100%; height:100vh;">
      <cover-view style="height:100vh;">
        <!-- 其它功能按钮 -->
        <cover-image :src="backIcon" class="complex-camera-back" @click="goBack"></cover-image>
        <cover-image :src="confirmIcon" class="complex-camera-previewer-success" v-if="photoSrc" @click="save">
        </cover-image>
        <!-- 预览 -->
        <cover-image :src="photoSrc" v-if="photoSrc" style="width: 100vw; height:100vh;z-index: 9800;" />
        <!-- 拍摄功能按钮 -->
        <cover-view v-if="!photoSrc"
          :class="[recordStartTime==0?'complex-camera-btn-text':'complex-camera-btn-recording-text']">
          {{hintText}}
        </cover-view>
        <cover-image v-if="!photoSrc" :src="cameraIcon" @touchstart="handleTouchShutter" @touchend="handleTouchShutter"
          @longpress="handleTouchShutter" class="complex-camera-btn" />
      </cover-view>
    </camera>
  </view>
</template>

<script>
  let recorderManager = uni.getRecorderManager();
  let innerAudioContext = uni.createInnerAudioContext('myAudio');
  innerAudioContext.autoplay = true;
  let cameraCtx; //相机上下文
  export default {
    data() {
      return {
        dest: '', //上一页的目标data
        windowHeight: '',
        touchStartTime: 0,
        recordStartTime: 0, //开始录制时的时间戳
        photoSrc: '',
        videoSrc: '',
        backIcon: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAwCAYAAABXAvmHAAAACXBIWXMAACNfAAAjXwHuwDalAAAGkmlUWHRYTUw6Y29tLmFkb2JlLnhtcAAAAAAAPD94cGFja2V0IGJlZ2luPSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1wbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUgWE1QIENvcmUgNy4xLWMwMDAgNzkuZWRhMmIzZmFjLCAyMDIxLzExLzE3LTE3OjIzOjE5ICAgICAgICAiPiA8cmRmOlJERiB4bWxuczpyZGY9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkvMDIvMjItcmRmLXN5bnRheC1ucyMiPiA8cmRmOkRlc2NyaXB0aW9uIHJkZjphYm91dD0iIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtbG5zOmRjPSJodHRwOi8vcHVybC5vcmcvZGMvZWxlbWVudHMvMS4xLyIgeG1sbnM6cGhvdG9zaG9wPSJodHRwOi8vbnMuYWRvYmUuY29tL3Bob3Rvc2hvcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RFdnQ9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZUV2ZW50IyIgeG1wOkNyZWF0b3JUb29sPSJBZG9iZSBQaG90b3Nob3AgMjMuMSAoTWFjaW50b3NoKSIgeG1wOkNyZWF0ZURhdGU9IjIwMjItMDgtMDFUMTY6NTc6MjkrMDg6MDAiIHhtcDpNb2RpZnlEYXRlPSIyMDIyLTA4LTAxVDE3OjEwOjIyKzA4OjAwIiB4bXA6TWV0YWRhdGFEYXRlPSIyMDIyLTA4LTAxVDE3OjEwOjIyKzA4OjAwIiBkYzpmb3JtYXQ9ImltYWdlL3BuZyIgcGhvdG9zaG9wOkNvbG9yTW9kZT0iMyIgeG1wTU06SW5zdGFuY2VJRD0ieG1wLmlpZDo4ZWRhNTUxOC0wYzc0LTRmNWUtYjE4YS0yNTdiZTcxMTRhY2MiIHhtcE1NOkRvY3VtZW50SUQ9InhtcC5kaWQ6YzdmYmNhMDYtZDRhYi00NDNmLTg3ZjAtYmE5NjBiOGFkZmVkIiB4bXBNTTpPcmlnaW5hbERvY3VtZW50SUQ9InhtcC5kaWQ6YzdmYmNhMDYtZDRhYi00NDNmLTg3ZjAtYmE5NjBiOGFkZmVkIj4gPHhtcE1NOkhpc3Rvcnk+IDxyZGY6U2VxPiA8cmRmOmxpIHN0RXZ0OmFjdGlvbj0iY3JlYXRlZCIgc3RFdnQ6aW5zdGFuY2VJRD0ieG1wLmlpZDpjN2ZiY2EwNi1kNGFiLTQ0M2YtODdmMC1iYTk2MGI4YWRmZWQiIHN0RXZ0OndoZW49IjIwMjItMDgtMDFUMTY6NTc6MjkrMDg6MDAiIHN0RXZ0OnNvZnR3YXJlQWdlbnQ9IkFkb2JlIFBob3Rvc2hvcCAyMy4xIChNYWNpbnRvc2gpIi8+IDxyZGY6bGkgc3RFdnQ6YWN0aW9uPSJzYXZlZCIgc3RFdnQ6aW5zdGFuY2VJRD0ieG1wLmlpZDowZWRiZjNkMS1kNjNiLTRiZmItOWM1Ny03MmQ4N2EyZWNhMDMiIHN0RXZ0OndoZW49IjIwMjItMDgtMDFUMTc6MDQ6NTArMDg6MDAiIHN0RXZ0OnNvZnR3YXJlQWdlbnQ9IkFkb2JlIFBob3Rvc2hvcCAyMy4xIChNYWNpbnRvc2gpIiBzdEV2dDpjaGFuZ2VkPSIvIi8+IDxyZGY6bGkgc3RFdnQ6YWN0aW9uPSJzYXZlZCIgc3RFdnQ6aW5zdGFuY2VJRD0ieG1wLmlpZDo4ZWRhNTUxOC0wYzc0LTRmNWUtYjE4YS0yNTdiZTcxMTRhY2MiIHN0RXZ0OndoZW49IjIwMjItMDgtMDFUMTc6MTA6MjIrMDg6MDAiIHN0RXZ0OnNvZnR3YXJlQWdlbnQ9IkFkb2JlIFBob3Rvc2hvcCAyMy4xIChNYWNpbnRvc2gpIiBzdEV2dDpjaGFuZ2VkPSIvIi8+IDwvcmRmOlNlcT4gPC94bXBNTTpIaXN0b3J5PiA8L3JkZjpEZXNjcmlwdGlvbj4gPC9yZGY6UkRGPiA8L3g6eG1wbWV0YT4gPD94cGFja2V0IGVuZD0iciI/PjuntroAAAQQSURBVGje1ZppiI5RFMeJIpN9yZZtUMzYhcEMKQ3DlPHBUvYsCdnCB9mX+MYXe/mgZAllKGMMaShr1FiLJlIYiWyTJT9fzluP4z7Pc5/3vc/MOHXr7b33nnP+95577rnnPLWAWv9zc8msMZAFzAPWA3uAo9L2ARuBBUAO0KwmAegKLAPOAK8Ip7dAIbAK6FGdAPKAs8BvUqMLwISqBJADlOCeSoHRcQPYFaDANwG2A5gFjASGSBsBTAe2yYp/DuBzAKjnGkAn4K6PwCtycFtGWIhmAvKiD8+HtufDRlgfOXimLc914ARGAZcM/D+JV0sJQC8xDU3LY/DpC4AfBllZyQJoA3xUzF4Bg2K8mDKB50pmJZCeDIBHitEzoH2IArmykqmAaAk8ULJfAnWjANhtuHxahwjuImPPO9iJJsALpcMRWwD9DHbY20L5xEFf6cicOgO/lB4jbADcVpOWhQhKB157xp+Xc9LOAYiZSpenYQBGqQn3LVb+jY8v/w48lospPwUQpYpvQRAA7Y+HJam8icqA2Ul6Ji/d9QPQQQ28EcFsotA1oH9EEMWKR4YJwHI1aIajlfej+REA5Ku5mzWAhnL4EvQDaBSj8glaawmgLvBehTGtvACGqninxLHZBNEaSxBnlIPI8wJYqJhuM9wNb4iPbB40K0zuPdG5QXVOU5OvEi/9tAjH9TnY7wWwV3XmqMkDgXcxgyixeAl66YQXwFHVOcTAoDtQETOI3Aj3wamoABIZiDh34kkAgAw19rQXwD7VmR3AqFvMIPwe9tlBJrRJdU4Nscc4zemU5SE+6AWwWHVusUxoRdmJ75KN+Cpex48qgTSDvKVq3EovgGylTJHl5RJkTteBrcAUsd+2QAtxlx0k8l0tsZEmU/R6QrndfC+AJpKvCVsFG3O6lUTKcKRKlq1T/bWVjBvyZv8rmFujVmFKxPzoOwcvsrXCo1j9P0bptt0UjaarQVcjCu8m8wpTfIXtlGCyvue/c0q3vn4PGh0yRI3bxwGLHDwl5wDN1cJ4H0a+LzK9VTdrQBGjOMjFmyaUqQlzq1H5AqXLC5usRJaa9FsOaVUr31q8oZfG2Ca2DhkyY02rUPn6kkL5J/axBVAHKDekvFtUgfJpwB0luwJoEDU32tmwhc9c1LVCsh1lhps5M9n0epahBlYpbs618pOADwbl81ItcAwFvhgYFwHDHSg+SIqFmn4B412VmHpKmtBEZyXsaBBB6XrARAmdTVQODHBd5EsDDgeEwRWi0CpRLkdsN0Oi3QmSSTgWkp45GaUQnsyWj5Vo0DXdS6ZenIrtTpYKZap0XUqw1fapQT/5DuJySP3XW08ulcfO4Jr0sUc7ueqXSJLguJyJ0/L7gCSQxwMda+LXKtXS/gBCwgW6CMP+GQAAAABJRU5ErkJggg==",
        confirmIcon: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAwCAYAAABXAvmHAAAACXBIWXMAAAsTAAALEwEAmpwYAAAFw2lUWHRYTUw6Y29tLmFkb2JlLnhtcAAAAAAAPD94cGFja2V0IGJlZ2luPSLvu78iIGlkPSJXNU0wTXBDZWhpSHpyZVN6TlRjemtjOWQiPz4gPHg6eG1wbWV0YSB4bWxuczp4PSJhZG9iZTpuczptZXRhLyIgeDp4bXB0az0iQWRvYmUgWE1QIENvcmUgNy4xLWMwMDAgNzkuZWRhMmIzZmFjLCAyMDIxLzExLzE3LTE3OjIzOjE5ICAgICAgICAiPiA8cmRmOlJERiB4bWxuczpyZGY9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkvMDIvMjItcmRmLXN5bnRheC1ucyMiPiA8cmRmOkRlc2NyaXB0aW9uIHJkZjphYm91dD0iIiB4bWxuczp4bXA9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC8iIHhtbG5zOmRjPSJodHRwOi8vcHVybC5vcmcvZGMvZWxlbWVudHMvMS4xLyIgeG1sbnM6cGhvdG9zaG9wPSJodHRwOi8vbnMuYWRvYmUuY29tL3Bob3Rvc2hvcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RFdnQ9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZUV2ZW50IyIgeG1wOkNyZWF0b3JUb29sPSJBZG9iZSBQaG90b3Nob3AgMjMuMSAoTWFjaW50b3NoKSIgeG1wOkNyZWF0ZURhdGU9IjIwMjItMDgtMDFUMTc6MDY6MTMrMDg6MDAiIHhtcDpNb2RpZnlEYXRlPSIyMDIyLTA4LTAxVDE3OjA5OjI3KzA4OjAwIiB4bXA6TWV0YWRhdGFEYXRlPSIyMDIyLTA4LTAxVDE3OjA5OjI3KzA4OjAwIiBkYzpmb3JtYXQ9ImltYWdlL3BuZyIgcGhvdG9zaG9wOkNvbG9yTW9kZT0iMyIgeG1wTU06SW5zdGFuY2VJRD0ieG1wLmlpZDozNzUwNDM4Ny0yMmFiLTRhZTItOWI0My1mM2ZmYmRlMDI1ZmMiIHhtcE1NOkRvY3VtZW50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",
        cameraIcon: "data:image/png;base64,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Uk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAAJcEhZcwAACxMAAAsTAQCanBgAAAXRaVRYdFhNTDpjb20uYWRvYmUueG1wAAAAAAA8P3hwYWNrZXQgYmVnaW49Iu+7vyIgaWQ9Ilc1TTBNcENlaGlIenJlU3pOVGN6a2M5ZCI/PiA8eDp4bXBtZXRhIHhtbG5zOng9ImFkb2JlOm5zOm1ldGEvIiB4OnhtcHRrPSJBZG9iZSBYTVAgQ29yZSA3LjEtYzAwMCA3OS5lZGEyYjNmYWMsIDIwMjEvMTEvMTctMTc6MjM6MTkgICAgICAgICI+IDxyZGY6UkRGIHhtbG5zOnJkZj0iaHR0cDovL3d3dy53My5vcmcvMTk5OS8wMi8yMi1yZGYtc3ludGF4LW5zIyI+IDxyZGY6RGVzY3JpcHRpb24gcmRmOmFib3V0PSIiIHhtbG5zOnhtcD0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wLyIgeG1sbnM6ZGM9Imh0dHA6Ly9wdXJsLm9yZy9kYy9lbGVtZW50cy8xLjEvIiB4bWxuczpwaG90b3Nob3A9Imh0dHA6Ly9ucy5hZG9iZS5jb20vcGhvdG9zaG9wLzEuMC8iIHhtbG5zOnhtcE1NPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvbW0vIiB4bWxuczpzdEV2dD0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL3NUeXBlL1Jlc291cmNlRXZlbnQjIiB4bXA6Q3JlYXRvclRvb2w9IkFkb2JlIFBob3Rvc2hvcCAyMy4xIChNYWNpbnRvc2gpIiB4bXA6Q3JlYXRlRGF0ZT0iMjAyMi0wNS0xMlQxMDoyNTo0MSswODowMCIgeG1wOk1vZGlmeURhdGU9IjIwMjItMDgtMDJUMTc6NTg6MjcrMDg6MDAiIHhtcDpNZXRhZGF0YURhdGU9IjIwMjItMDgtMDJUMTc6NTg6MjcrMDg6MDAiIGRjOmZvcm1hdD0iaW1hZ2UvcG5nIiBwaG90b3Nob3A6Q29sb3JNb2RlPSIzIiB4bXBNTTpJbnN0YW5jZUlEPSJ4bXAuaWlkOjRkYzkzMGI4LTE4NmItNDEwMy1iOGY4LTQ5ZTlhMTU2YzZkZiIgeG1wTU06RG9j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",
      }
    },
    computed: {
      hintText() {
        if (this.recordStartTime === 0) {
          return "轻触拍照，长按录像";
        } else {
          return "录像中"
        }
      }
    },
    onLoad(e) {
      this.dest = e.dest;
      cameraCtx = uni.createCameraContext(); //初始化相机上下文
    },
    methods: {
      handleTouchShutter(e) {
        switch (e.type) {
          case "touchstart":
            this.touchStartTime = e.timeStamp;
            break;
          case "longpress":
            this.recordStartTime = new Date().getTime();
            //开始视频录制
            this.takeVideo();
            break;
          case "touchend":
            const touchTime = e.timeStamp - this.touchStartTime; //单位ms
            if (touchTime > 350 || this.recordStartTime != 0) {
              //结束视频录制
              uni.showLoading({
                title: '保存中',
                mask: true,
                success: () => {
                  cameraCtx.stopRecord({
                    // 视频压缩经常出问题
                    // compressed: true,
                    success: (res) => {
                      this.stopRecordVideo(res);
                    },
                    complete: (res) => {
                      uni.hideLoading();
                    }
                  })
                }
              })
            } else {
              //拍照
              this.takePhoto();
            }
            break;
        }
      },
      takePhoto() {
        this.photoSrc = "";
        this.videoSrc = "";
        cameraCtx.takePhoto({
          quality: 'low',
          success: (res) => {
            let photoSrc = res.tempImagePath;
            //图片压缩则交换注释
            this.photoSrc = photoSrc;
            // uni.compressImage({
            //   src: photoSrc, // 图片路径
            //   quality: 50, // 压缩质量
            //   success: (response) => {
            //     this.photoSrc = response.tempFilePath;
            //   }
            // })
          }
        });
      },
      takeVideo() {
        //开始录像
        this.photoSrc = "";
        this.videoSrc = "";
        cameraCtx.startRecord({
          timeoutCallback: (cb) => {
            uni.showLoading({
              title: '保存中',
              mask: true,
              success: (res) => {
                this.stopRecordVideo(cb)
              }
            })
          },
          success: (res) => {
            console.log("开始录像", new Date())
          },
          fail: (err) => {
            console.log(err)
          },
        })
      },
      stopRecordVideo(res) {
        uni.hideLoading();
        this.recordStartTime = 0;
        const {
          tempThumbPath,
          tempVideoPath
        } = res;
        this.videoSrc = tempVideoPath;
      },
      goBack() {
        if (this.photoSrc || this.videoSrc) {
          this.photoSrc = "";
          this.videoSrc = "";
          this.touchStartTime = 0;
          this.recordStartTime = 0;
        } else {
          uni.navigateBack()
        }
      },
      save() {
        let pages = getCurrentPages();
        let prevPage = pages[pages.length - 2];
        prevPage.$vm[this.dest] = this.videoSrc || this.photoSrc;
        if (this.btn) {
          prevPage.$vm[this.btn] = false;
        }
        uni.navigateBack();
      },
      bindError(e) {
        let _this = this;
        uni.showModal({
          title: '提示',
          content: '请允许摄像头授权',
          showCancel: false,
          success: function(res) {
            if (res.confirm) {
              uni.openSetting({
                success(res) {
                  if (res.authSetting['scope.camera']) {
                    uni.navigateBack()
                  }
                }
              })
            }
          }
        });
      },
    },
    onUnload() {
      recorderManager = null;
      innerAudioContext = null;
      cameraCtx = null;
    }
  }
</script>
<style>
  page {
    padding-bottom: 0;
  }
</style>
<style lang="scss" scoped>
  $btn-bottom: 100rpx;

  .complex-camera-btn-text {
    color: white;
    position: absolute;
    left: 260rpx;
    bottom: $btn-bottom+170rpx;
  }

  .complex-camera-btn-recording-text {
    color: red;
    position: absolute;
    left: 330rpx;
    bottom: $btn-bottom+170rpx;
  }

  .complex-camera-btn {
    z-index: 9000;
    position: absolute;
    width: 160rpx;
    height: 160rpx;
    left: 295rpx;
    bottom: $btn-bottom;
  }

  // 返回按钮
  .complex-camera-back {
    position: absolute;
    left: 50rpx;
    bottom: $btn-bottom+56rpx;
    z-index: 9990;
    width: 48rpx;
    height: 48rpx;
  }

  //完成按钮
  .complex-camera-previewer-success {
    position: absolute;
    right: 50rpx;
    bottom: $btn-bottom+56rpx;
    z-index: 9990;
    width: 48rpx;
    height: 48rpx;
  }
</style>
```

