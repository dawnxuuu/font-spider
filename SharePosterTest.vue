<template>
  <div v-show="show"
       id="cer-canvas"
       class="share-img"
       @click="controlModal(false)">
    <div v-if="showloading"
         class="share-img__loading">
      <van-loading size="40px"
                   type="spinner" />
    </div>
    <div class="share-img__wrap">
      <div class="share-img__box">
        <div class="share-img__ctc">
          <div class="share-img__ctc-left">
            <div class="share-img__info">
              <img class="share-img__avater"
                   crossOrigin="anonymous"
                   :src="avatar">
              <div class="share-img__ctx">
                <div class="share-img__ctx-text">{{ loginInfo.userName }}</div> ·
                <div class="share-img__ctx-long-text">{{ loginInfo.businesslicenceurlname }}</div>
              </div>
            </div>
            <div class="share-img__ctc-text">{{ one.message }}</div>
          </div>
          <div class="share-img__ctc-right" />
          <img class="share-img__ctc-right-img"
               crossOrigin="anonymous"
               :src="qrcodepre">
        </div>
      </div>
    </div>
    <img :src="posterImg"
         crossOrigin="anonymous"
         class="share-poster__poster">
    <canvas id="canvasId"
            class="share-poster__canvas"
            width="640px"
            height="848px" />
  </div>
</template>

<script>
import '../style/share-img.scss'
import Vue from 'vue'
import { eventBus } from 'utils/event-bus.js'
import env from 'env'
import device from 'utils/device.js'
import shareWechat from 'assets/lantern-festival/share-wechat.png'
import shareFriends from 'assets/lantern-festival/share-friends.png'
import qrcodepre from 'assets/lantern-festival/qrcode-pre.png'
import lanternshare from 'assets/lantern-festival/lantern-share1.png'
import defaultAvatar from 'assets/lantern-festival/lantern-default-avatar.png'
import { Toast, Loading } from 'vant'
// import QRCode from 'qrcode'
import { completeFestivalTask, upload, getMineRecordList } from 'api/lantern.js'
import html2canvas from 'html2canvas'
import { lanterclose, lanternApp, lanternShareGift } from '../assets/image.js'
import {
  idToGood, avatarFormat, linkGifts
} from '../util/operation.js'

const axios = require('axios')

Vue.use(Loading)

// Vue.use(QRCode)

export default {
  name: 'ShareImg',
  props: {
    loginInfo: {
      type: Object,
      default () {
        return {}
      }
    },
    taskInfo: {
      type: Object,
      default () {
        return {}
      }
    }
  },
  data () {
    return {
      lanterclose,
      lanternApp,
      lanternshare,
      lanternShareGift,
      shareWechat,
      shareFriends,
      qrcodepre,
      show: true,
      list: [],
      active: 0,
      one: {
        message: '【智联招聘】奥利给！逢抽必中666！快来和我一起抽盲盒吧！'
      },
      two: {
        title: 'HR抗“疫”集结号吹响！智联福利大礼包速来领取！',
        message: '【智联招聘】奥利给！逢抽必中666！快来和我一起抽盲盒吧！',
        img: lanternShareGift
      },
      goodSrcNetName: 'lantern-share-gift',
      shareimgSrc: '',
      blob: '',
      baseImg: '',
      imgDrawed: false,
      showloading: false,
      posterImg: ''
    }
  },
  computed: {
    avatar () {
      // return avatarFormat(this.loginInfo.avatar)
      return 'https://storage-public.zhaopin.cn/wechat/avatar/1576021936838364930/132'
    }
  },
  mounted () {
    // this.useqrcode()

    this.$nextTick(() => {
      const w = 640
      const h = 840
      const scaleBy = this.getDpr()
      const canvas = document.getElementById('canvasId')
      canvas.width = w * scaleBy
      canvas.height = h * scaleBy
      canvas.style.width = `${w}px`
      canvas.style.height = `${h}px`
      const context = canvas.getContext('2d')
      // 然后将画布缩放，将图像放大两倍画到画布上
      context.scale(scaleBy, scaleBy)
      html2canvas(document.querySelector('#cer-canvas'), {
        background: '#fff',
        allowTaint: true,
        width: w,
        height: h,
        canvas,
        scale: 1
      }).then((canvas) => {
        // document.querySelector('#cer-canvas').appendChild(canvas)
        context.mozImageSmoothingEnabled = false
        context.webkitImageSmoothingEnabled = false
        context.msImageSmoothingEnabled = false
        context.imageSmoothingEnabled = false
        this.posterImg = canvas.toDataURL('image/png', 1)
      })
    })
  },
  created () {
    this.getRecordData()
    eventBus.$on('controlShare', (data) => {
      this.controlModal(data)
    })
    eventBus.$on('recordList', (data = []) => {
      this.list = data
      this.shareText()
    })
  },
  methods: {
    getDpr () {
      if (window.devicePixelRatio && window.devicePixelRatio > 1) {
        return window.devicePixelRatio
      }
      return 1
    },

    // 分享文案
    shareText () {
      const data = this.list
      if (!data.length) {
        return
      }
      const lastGift = data[0] || {}
      const productType = lastGift.productType
      if (productType !== 40 && productType !== 30) {
        this.one = {
          message: `【智联招聘】奥利给！逢抽必中666！快来和我一起抽盲盒吧！`
        }
        this.two.message = `【智联招聘】奥利给！逢抽必中666！快来和我一起抽盲盒吧！`
        this.two.src = lanternShareGift
        this.goodSrcNetName = 'lantern-share-gift'
        return
      }
      this.one.message = `【智联招聘】奥利给！${lastGift.lotteryDesc}竟然被我抽到了！快来和我一起抽盲盒吧！`
      this.two.message = `【智联招聘】奥利给，中奖啦！${lastGift.lotteryDesc}竟然被我抽到了！快来和我一起抽盲盒吧！`
      if (productType !== 40) {
        this.two.img = lanternShareGift
        this.goodSrcNetName = 'lantern-share-gift'
      } else {
        const good = idToGood(lastGift.productId)
        if (good === 'gift1' || good === 'gift2' || good === 'gift3' || good === 'gift16' || good === 'gift17') {
          this.two.img = lanternShareGift
          this.goodSrcNetName = 'lantern-share-gift'
        } else {
          this.goodSrcNetName = good
          this.two.img = linkGifts[good]
        }
      }
    },
    // 微信分享
    toShareWx (scene) {
      // if (this.active === 0) {
      //   this.sharePoster(scene)
      //   return
      // }
      const { two, goodSrcNetName } = this
      const params = {
        title: two.title,
        description: two.message,
        path: `${env.rd5Url}/m/lantern-main?source=2&fromPage=4`,
        imageUrl: `https://img01.zhaopin.cn/IHRNB/bapp/${goodSrcNetName}.png`,
        scene
      }
      this.shareComplete()
      window.shareCompleteCallback = this.shareComplete
      if (device.ios) {
        window.webkit.messageHandlers.ZLWeexShareModule.postMessage({
          methodName: 'webShare:callBack:',
          params,
          callBack: 'shareCompleteCallback'
        })
      } else {
        /*eslint-disable*/
        ZLWeexShareModule.webShare(JSON.stringify(params), (result) => {
          this.shareComplete(result)
        })
      }
    },
    // 海报分享
    async sharePoster (scene) {
      if (!this.imgDrawed) {
        this.showloading = true
        setTimeout(()=>{
          this.sharePoster(scene)
        }, 1500)
        return
      }
      if(!this.shareimgSrc){
        this.showloading = true
        await this.uploadImg()
      }
        this.showloading = false

      const { one, shareimgSrc } = this
      const params = {
        imageUrl: shareimgSrc,
        scene
      }

      this.shareComplete()
      window.shareCompleteCallback = this.shareComplete
      if (device.ios) {
        window.webkit.messageHandlers.ZLWeexShareModule.postMessage({
          methodName: 'imageShare:callBack:',
          params,
          callBack: 'shareCompleteCallback'
        })
      } else {
        /*eslint-disable*/
        ZLWeexShareModule.imageShare(JSON.stringify(params), (result) => {
          // this.shareComplete(result)
        })
      }
    },
    async shareComplete () {
      const {
        validActivitySharedNum = 0
      } = this.taskInfo
      if (validActivitySharedNum > 0 ) {
        return
      }
      const { at, rt } = this.loginInfo
      const res = await completeFestivalTask({
        'x-zp-at': at,
        'x-zp-rt': rt
      }, {
        taskType: 'ACTIVITY_SHARED'
      })
      if(res.code === 200){
        eventBus.$emit('refreshTask')
      }
    },
    
    useqrcode(){
      const { source, at, rt } = this.loginInfo
      if(source === '2'){
        return
      }
        const canvas = document.getElementById('canvasId')
        const url = env.rd5Url + '/m/lantern-main?source=2&fromPage=3'
        // QRCode.toCanvas(canvas, url, (error) => {
        //     if (error) {
        //         console.log(error)
        //         return false
        //     }
            let imgBg = new Image()
            imgBg.crossOrigin = "Anonymous"
            imgBg.src = this.lanternshare

            let ctx = canvas.getContext('2d')
            imgBg.addEventListener('load', () => {
              ctx.drawImage(imgBg, 0, 0, 640, 840)
              const left = 40
              const top = 700

              // 画头像 defaultAvatar
              let imgAvatar = new Image()
              imgAvatar.crossOrigin = 'Anonymous'
              imgAvatar.src = defaultAvatar || this.loginInfo.avatar || defaultAvatar
              imgAvatar.addEventListener('load', () => {
                  // 绘制圆，参数（x坐标，y坐标，圆半径，起始角度，终止角度）
                  ctx.arc(left + 40, top - 20, 40, 0, 2*Math.PI)
                  ctx.save()
                  // 剪切形状
                  ctx.clip()
                  ctx.drawImage(imgAvatar, left, top - 60, 80, 80)
              })
              // 画第二张图 - 二维码 
              let imgQrcode = new Image()
              imgQrcode.crossOrigin = 'Anonymous'
              imgQrcode.src = qrcodepre
              imgQrcode.addEventListener('load', () => {
                ctx.drawImage(imgQrcode, 490, 676, 128, 128)
              })

              // 设置字体大小
              ctx.font = "28px Microsoft YaHei"
              // 更改字号后，必须重置对齐方式，否则居中麻烦。设置文本的垂直对齐方式
              ctx.textBaseline = 'middle'
              ctx.textAlign = 'left'
              // 文字颜色
              ctx.fillStyle = "#333"
              // 文字在画布的位置
              const {userName = '耿炎', businesslicenceurlname: bussnessname = '智联招聘'} = this.loginInfo
              let name = userName.length > 4 ? userName.slice(0, 4) + "..." : userName
              let bussname = bussnessname.length > 6 ? (bussnessname || 'SUNNER').slice(0, 6) + "..." :bussnessname 
              ctx.fillText(`${name} · ${bussname}`, left + 96, top)

              this.drawText(this.one.message, left -8, top + 12, 640 - 220, ctx)

              setTimeout(() => {
                try {
                  this.baseImg = canvas.toDataURL("image/png", 1.0)
                  this.imgDrawed = true
                  Toast('稍安勿躁，测试用的'+this.baseImg.slice(0, 4))
                  this.uploadImg()
                } catch (error) {
                  Toast('兼容性又不过')
                }
                
                // console.log(this.baseImg)
              // Toast('oopp1')
                // canvas.toBlob((blob) =>{
                  // this.blob = this.convertBase64UrlToBlob(blobs)
                  // this.uploadImg(this.blob)
                // }, 'image/png')
              }, 1500)
            })
        // });
    },
    convertBase64UrlToBlob (dataurl) {
      const arr = dataurl.split(',')
      const bstr = window.atob(arr[1])
      let n = bstr.length
      const u8arr = new Uint8Array(n)
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n)
      }
      return new Blob([u8arr], { type: 'image/png' })
    },
    async uploadImg () {
      const token = await this.uploadToken()
      if(!token){
        return
      }
      return new Promise((resolve, reject) => {
        const file = this.convertBase64UrlToBlob(this.baseImg)
        file.name = 'lantern'
        const formData = new FormData()
        formData.set('multipartFile', file, file.name)
        axios({
          url: `https://encryptedstorage-storage.zhaopin.com/storage/putFile?token=${token}&clientIp=127.0.0.1`,
          data: formData,
          rocessData: false,
          contentType: false,
          method: 'POST'
        }).then((res) => {
          if (res.status === 200) {
            if(res.data.code === 200){
              this.shareimgSrc = res.data.fileUrl
            }else{
              Toast(res.data.message || '图片上传失败，请稍后再试')
            }
          } else {
            Toast('图片上传失败，请稍后再试')
          }

          resolve()
        }).catch((err) => {
          resolve()
          Toast('图片上传失败，请稍后再试')
        })
      })
      
    },
    // 自动换行
    drawText(text,x,y,w,ctx){
      var chr = text.split("")
      var temp = ""
      var row = []
      
      ctx.font = "24px Arial"
      ctx.fillStyle = "#666"
      ctx.textBaseline = "left"
      ctx.textAlign = 'left'
      
      for(var a = 0; a < chr.length; a++){
        if( ctx.measureText(temp).width < w ){
          ;
        }
        else{
          row.push(temp)
          temp = ""
        }
        temp += chr[a]
      }
      
      row.push(temp)
      
      for(var b = 0; b < row.length; b++){
        ctx.fillText(row[b],x,y+(b+1)*40)
      }
  },
  // 获取token
  async uploadToken(){
    const { at, rt } = this.loginInfo
    const { code, data, message } = await upload({
      'x-zp-at': at,
      'x-zp-rt': rt
    })
    if( code === 200 ){
      return data.uploadInfo.token
    }else{
      Toast(message || '获取凭证失败')
    }
  },
  async getRecordData () {
      const { at, rt } = this.loginInfo
      if (!at) {
        return
      }
      const { code, data} = await getMineRecordList({
        'x-zp-at': at || '',
        'x-zp-rt': rt || ''
      })
      if (code === 200) {
        this.list = data
        this.shareText()
      }
    },

    tab (data) {
      this.active = data
    },
    controlModal (data) {
      this.show = !!data
    },
    touchmove (e) {
      e.preventDefault()
    }
  }
}
</script>
<style>
/** @define share-poster; weak */
.share-poster {
  /* position: absolute; */
}

.share-img__wrap {
  left: -1000px;
  position: absolute;
}

.share-poster__canvas {
  left: 0;
  position: absolute;
  top: 0;
  z-index: 10;
}

.share-poster__poster {
  left: 0;
  position: absolute;
  top: 0;
  z-index: 10;
}
</style>