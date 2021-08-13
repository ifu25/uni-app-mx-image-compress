## uni-app 图片压缩组件

### 使用方法

```html
<button class="cu-btn block bg-blue margin-tb-sm lg" @click="chooseImg">选择图片</button>
<button class="cu-btn block bg-blue margin-tb-sm lg" @click="upload">上传</button>
<mx-compress ref="mxCompress"></mx-compress>
<image :src="imgSrc"></image>
```

```javascript
import mxCompress from '../../components/mx-compress/mx-compress.vue';
export default {
    components: {
        mxCompress
    },
    data() {
        return {
            imgSrc: '',
            imglist: [],
        }
    },
    methods: {
        upload() {
            console.log(this.imglist[0].file);
            uni.uploadFile({
                url: 'http://127.0.0.1:8080/uploadFile',
                name: "file",
                // #ifdef H5
                filePath: this.imglist[0].base64,
                // #endif
                // #ifdef MP-WEIXIN
                filePath: this.imglist[0].file,
                // #endif
                formData: {
                    'bill_no': '202108010001'
                },
                success: (uploadFileRes) => {
                    console.log(uploadFileRes.data);
                },
                fail: (err) => {
                    console.log(err);
                },
                complete() {
                    console.log('上传完成');
                }
            })
        },
        chooseImg() {
            var vm = this
            uni.chooseImage({
                success: async (res) => {
                    try {
                        //调用组件的compress方法压缩图片
                        let result = await this.$refs.mxCompress.compress(res, {
                            maxLongSize: 1440,      //最大长边尺寸
                            quality: 0.8,           //压缩质量(0-1)，默认为0.8
                            base64: true,           //是否同时返回base64格式，默认false
                            showLoading: true,      //是否显示loading提示，也可以传入一个字符串（相当于loading时的title），默认true
                            drawWaitTime: 1500,     //等待绘图时间，用于微信小程序，解决真机调试压缩图片不完整问题
                            mask: true              //当showLoading为true时，是否显示遮罩层，默认为true
                        });
                        this.imglist = result;
                        this.imgSrc = result[0].base64
                    } catch (error) {
                        console.log('error', error);
                    }
                }
            })
        }
    }
}
```

