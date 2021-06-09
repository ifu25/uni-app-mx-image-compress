<template>
  <view class="mx-compress-container">
    <!-- #ifndef H5 -->
    <canvas :style="{width: W + 'px', height: H + 'px', visibility: 'hidden'}" class="canvas" canvas-id="canvas"></canvas>
    <!-- #endif -->
  </view>
</template>

<script>
export default {
  data() {
    return {
      W: '',                        //最终图片宽
      H: '',                        //最终图片高
      canvas: null,                 //画布，用于压缩
      ctx: null,                    //画布上下文
      maxLongSize: 1280,            //长边最大长度
      quality: 0.8,                 //压缩质量，0.1-1.0
      base64: false,                //是否base64
      waitDrawTime: 1500,           //等待绘图完成时间，用于解决微信小程序真机调试时图片压缩不完整
      showLoading: '正在压缩..',     //提示信息
      mask: true                    //遮罩
    }
  },
  methods: {
    //压缩
    async compress(args, options = {}) {
      return new Promise(async (resolve, reject) => {
        let files;
        if (arguments[0].tempFiles || arguments[0].tempFilePaths) {
          files = arguments[0].tempFilePaths || arguments[0].tempFiles;
        };
        if (arguments[0].files) {
          files = arguments[0].files;
        }
        if (!files instanceof Array) {
          reject('数据格式错误');
        }
        if (!files.length) {
          reject('数据不能为空');
        }
        this.maxLongSize = options.maxLongSize || 1280;
        this.quality = options.quality || 0.8;
        this.base64 = options.base64 || false;
        this.waitDrawTime = options.waitDrawTime || 1500;
        this.showLoading = options.showLoading === true ? '正在压缩..' : typeof options.showLoading === 'string' ? options.showLoading : false;
        this.mask = options.mask || true;
        if (this.showLoading) {
          uni.showLoading({
            title: this.showLoading,
            mask: this.mask,
          })
        }
        try {
          const result = await this.doCompress(files);
          resolve(result);
          uni.hideLoading();
        } catch (error) {
          reject(error);
          uni.hideLoading();
        }
      })
    },
    //压缩
    async doCompress(files) {
      // #ifdef H5
      if (typeof files[0] === 'object') {
        let result = await this.toBase64H5(files);
        return await this.compressImageH5(result);
      }
      if (typeof files[0] === 'string') {
        let result = files.map(item => {
          return {
            base64: item
          }
        });
        return await this.compressImageH5(result);
      }
      return [];
      // #endif

      // #ifndef H5
      if (typeof files[0] === 'string') {
        const result = await this.getImgInfoWX(files);
        return await this.compressImageWX(result);
      }
      if (typeof files[0] === 'object') {
        files = files.map(item => {
          return item.path
        });
        const result = await this.getImgInfoWX(files);
        return await this.compressImageWX(result);
      }
      return [];
      // #endif
      return [];
    },
    compressImageH5(base64Result) {
      let result = [];
      return new Promise(async (resolve, reject) => {
        for (let i = 0; i < base64Result.length; i++) {
          console.log(base64Result[i]);
          let res = await this.compressResultH5(base64Result[i]);
          result.push(res);
          if (result.length === base64Result.length) {
            resolve(result);
            this.canvas = null;
          }
        }
      })
    },
    compressResultH5(base64Item) {
      return new Promise((resolve, reject) => {
        let image = new Image();
        image.src = base64Item.base64;
        image.addEventListener('load', () => {
          let oriWidth = image.naturalWidth
          let oriHeight = image.naturalHeight

          //不需要压缩尺寸
          if (oriWidth < this.maxLongSize && oriHeight < this.maxLongSize) {
            this.W = oriWidth
            this.H = oriHeight
          }
          //需要压缩尺寸
          else {
            if (oriWidth > oriHeight) {
              this.W = this.maxLongSize
              this.H = Math.trunc(this.maxLongSize * oriHeight / oriWidth)
            } else {
              this.H = this.maxLongSize
              this.W = Math.trunc(this.maxLongSize * oriWidth / oriHeight)
            }
          }

          if (!this.canvas) {
            this.canvas = document.createElement('canvas');
          }
          this.canvas.width = this.W;
          this.canvas.height = this.H;
          const ctx = this.canvas.getContext('2d');
          ctx.clearRect(0, 0, this.W, this.H);
          ctx.drawImage(image, 0, 0, this.W, this.H);
          const compressImg = this.canvas.toDataURL('image/jpeg', this.quality);
          let file = this._dataURLtoFile(compressImg, base64Item.filename);
          if (this.base64) {
            resolve({
              base64: compressImg,
              file,
              width: this.W,
              height: this.H
            });
          } else {
            resolve({
              file,
              width: this.W,
              height: htis.H
            });
          }
          image = null;
        })
      })
    },
    //微信获取图片信息：主要是长和宽
    getImgInfoWX(tempFilePaths) {
      let result = [];
      return new Promise((resolve, reject) => {
        for (let i = 0; i < tempFilePaths.length; i++) {
          uni.getImageInfo({
            src: tempFilePaths[i],
            success: (info) => {
              result.push({
                filePath: tempFilePaths[i],
                fileInfo: info
              });
              if (result.length === tempFilePaths.length) {
                resolve(result);
              }
            }
          });
        }
      })
    },
    //微信压缩图片
    compressImageWX(tempFiles) {
      let result = [];
      return new Promise(async (resolve, reject) => {
        for (let i = 0; i < tempFiles.length; i++) {
          let res = await this.compressResultWX(tempFiles[i]);
          result.push(res);
          if (result.length === tempFiles.length) {
            resolve(result);
            this.ctx = null;
          }
        }
      })
    },
    //微信压缩图片
    compressResultWX(tempFile) {
      return new Promise((resolve, reject) => {
        let oriWidth = tempFile.fileInfo.width;
        let oriHeight = tempFile.fileInfo.height;
        let filePath = tempFile.filePath

        //不需要压缩
        if (oriWidth < this.maxLongSize && oriHeight < this.maxLongSize) {
          if (this.base64) {
            let base64 = uni.getFileSystemManager().readFileSync(filePath, 'base64');
            base64 = `data:image/jpeg;base64,${base64}`
            resolve({
              file: filePath,
              base64,
              width: oriWidth,
              height: oriHeight
            });
          } else {
            resolve({
              file: filePath,
              width: oriWidth,
              height: oriHeight
            });
          }

          return
        }

        if (oriWidth > oriHeight) {
          this.W = this.maxLongSize
          this.H = Math.trunc(this.maxLongSize * oriHeight / oriWidth)
        } else {
          this.H = this.maxLongSize
          this.W = Math.trunc(this.maxLongSize * oriWidth / oriHeight)
        }

        if (!this.ctx) {
          this.ctx = uni.createCanvasContext('canvas', this);
        }
        this.ctx.clearRect(0, 0, this.W, this.H);
        this.ctx.drawImage(filePath, 0, 0, this.W, this.H);
        this.ctx.draw(false, setTimeout(() => {
          uni.canvasToTempFilePath({
            x: 0,
            y: 0,
            width: this.W,
            height: this.H,
            destWidth: this.W,
            destHeight: this.H,
            canvasId: 'canvas',
            quality: this.quality,
            fileType: 'jpg',
            success: (res) => {
              let newFilePath = res.tempFilePath;
              if (this.base64) {
                let base64 = uni.getFileSystemManager().readFileSync(newFilePath, 'base64');
                base64 = `data:image/jpeg;base64,${base64}`
                resolve({
                  file: newFilePath,
                  base64,
                  width: this.W,
                  height: this.H
                });
              } else {
                resolve({
                  file: newFilePath,
                  width: this.W,
                  height: this.H
                });
              }
            }
          }, this)
        }, this.waitDrawTime)); //等待几秒确保绘图完成（用于解决微信小程序真机调试压缩图片不完整的不完美方案）
      })
    },
    toBase64H5(file) {
      return new Promise((resolve, reject) => {
        let result = [];
        for (let i = 0; i < file.length; i++) {
          let reader = new FileReader();
          let base64Result;
          reader.addEventListener('load', (e) => {
            base64Result = reader.result || e.target.result;
            let filename = file[i].name.slice(0, file[i].name.lastIndexOf('.'));
            result.push({
              base64: base64Result,
              filename
            });
            reader = null;
            if (result.length === file.length) {
              resolve(result);
            }
          });
          reader.readAsDataURL(file[i]);
        }
      })
    },
    _dataURLtoFile(dataurl, filename) {
      let arr = dataurl.split(',');
      let mime = arr[0].match(/:(.*?);/)[1],
        bstr = atob(arr[1]),
        n = bstr.length,
        u8arr = new Uint8Array(n);
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      return new File([u8arr], filename, {
        type: mime
      });
    }
  }
}
</script>

<style scoped>
/* 将画布移出屏幕使其不可见，仅用于压缩图片 */
.mx-compress-container {
  width: 0;
  height: 0;
  margin: 0;
  padding: 0;
  overflow: hidden;
  position: absolute;
  left: -9000px;
  top: -9000px;
  z-index: -100000;
}
</style>
