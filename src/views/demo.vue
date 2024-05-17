<script setup>
import { ref, reactive } from 'vue';

import maskImage from '../assets/images/qujing.png';

const wrapperRef = ref(null);
const maskImageRef = ref(null);
const videoRef = ref(null);
const state = reactive({
  type: '',//上传类型 update|upload
  cameraShow: false,//启动自定义相机
  status: 0,//自定义相机-拍摄进度:0|未开启 1|开启但未拍摄 2|开启且已拍摄
  imageUrl: '',//自定义相机-抓拍url
  front: true,// 自定义相机-前置与后置转换（未验证）

  //提示部分
  tipVisible: false,
  btntext: '',//倒计时文本
  time: null,//计时器

  imageFile: '',//图片对象
})

function openCamera() {
  // 1. 先展示，因为要从这里获取video标签
  state.cameraShow = true

  // 2. constraints:指定请求的媒体类型和相对应的参数
  var constraints = {
    audio: false,
    video: {
      facingMode: (state.front ? "user" : "environment")
    }
  }

  // 3. 兼容部分：
  // 老的浏览器可能根本没有实现 mediaDevices，所以我们可以先设置一个空的对象
  if (navigator.mediaDevices === undefined) {
    navigator.mediaDevices = {};
  }
  // 一些浏览器部分支持 mediaDevices。我们不能直接给对象设置 getUserMedia
  // 因为这样可能会覆盖已有的属性。这里我们只会在没有getUserMedia属性的时候添加它。
  if (navigator.mediaDevices.getUserMedia === undefined) {
    navigator.mediaDevices.getUserMedia = function (constraints) {
      // 首先，如果有getUserMedia的话，就获得它
      var getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia || navigator.msGetUserMedia || navigator.oGetUserMedia;
      // 一些浏览器根本没实现它 - 那么就返回一个error到promise的reject来保持一个统一的接口
      if (!getUserMedia) {
        return Promise.reject(new Error('getUserMedia is not implemented in state browser'));
      }
      // 否则，为老的navigator.getUserMedia方法包裹一个Promise
      return new Promise(function (resolve, reject) {
        getUserMedia.call(navigator, constraints, resolve, reject);
      });
    }
  }


  // 4. 获取视频流
  navigator.mediaDevices.getUserMedia(constraints)
    .then((stream) => {
      // 进来这里表示能够兼容
      let video = document.querySelector('video');
      video.srcObject = stream;
      video.onloadedmetadata = function (e) {
        video.play();
      };
      // 进入自定义拍摄模式
      state.status = 1
    })
    .catch(function (err) {
      // 进来这里表示不能兼容
      console.log('nonono', err)
      // 调用原始摄像头
      originCamera()
    });
}

function snapPhoto() {
  var canvas = document.querySelector('#mycanvas');
  var video = document.querySelector('video');
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  canvas.getContext('2d').drawImage(video, 0, 0);

  // 保存为文件，用于后续的上传至服务器（尚未实践）——>后续提交
  // state.imageFile = canvasToFile(canvas)

  // blob转url：用于展示
  let p = new Promise((resolve, reject) => {
    canvas.toBlob(blob => {
      let url = URL.createObjectURL(blob)
      resolve(url)
    });
  })
  p.then(value => {
    state.imageUrl = value
    state.status = 2//表示拍摄完成
  })
}

function originCamera() {
  //关闭自定义相机
  state.cameraShow = false
  let promise = new Promise(function (resolve, reject) {
    let file = document.getElementById('file')
    file.click()
    file.onchange = function (event) {
      if (!event) {
        reject('empty')
      }
      //当选中或者拍摄后确定图片后，保存图片文件——>后续提交
      let file = event.target.files[0]
      resolve(file)
    }
  })
  promise.then((value) => {
    submitPhoto('origin', value)
  }
  )
}

function submitPhoto(type, file) {
  if (type == 'origin') {
    state.imageFile = file
  }
  console.log("提交", state.imageUrl);

  state.cameraShow = false
}

function imageLoad(e) {
  const pageWidth = window.innerWidth;
  const scale = pageWidth / e.naturalWidth;
  const height = (e.naturalHeight * scale) + 'px'
  console.log(height, scale);
  wrapperRef.value.style.width = pageWidth + 'px';
  wrapperRef.value.style.height = height;
  videoRef.value.style.height = height;
}
</script>

<template>
  <div class="uploadFacePic">
    <!-- 启动相机按钮-->
    <main>
      <van-uploader>
        <van-button icon="photo" class="btn">上传蒙版图片</van-button>
      </van-uploader>
      <div class="btn" @click="openCamera">开启人脸采集</div>
    </main>

    <!-- 1 原始相机 不兼容时的回退方案-->
    <input id="file" type="file" accept="image/*" capture="camera" style="display:none">


    <!-- 2 自定义相机 -->
    <div class="customCamera" v-if="state.cameraShow">
      <!-- 中间部分 -->
      <video ref="videoRef" class="fixed"></video>
      <div ref="wrapperRef" class="fixed">
        <!-- a.拍摄时展示取景框。把取景框图片换下就可以了 -->
        <img ref="maskImageRef" :src="maskImage" alt=""
          style="width: 100%;height: 100%;opacity: 0.8; position: absolute;top: 0;left: 0;"
          @load="imageLoad($event.target)"
          >
        <!-- b.拍摄完后展示抓拍图片 -->
        <img :src="state.imageUrl" alt="" v-if="state.status == 2" style="width:100%;height:100%">
      </div>

      <!-- 底下控制部分-->
      <div class="control">
        <!-- 拍照前 -->
        <div class="control_before" v-if="state.status == 1">
          <div class="control_before_top">照片</div>
          <div class="control_before_bottom">
            <div class="smaller" @click="state.cameraShow = false">取消</div>
            <van-button type="primary" size="small" @click="snapPhoto">拍照</van-button>
          </div>
        </div>

        <!-- 拍照后 -->
        <div class="control_after" v-if="state.status == 2">
          <div class="smaller" @click="state.status = 1">重拍</div>
          <div class="smaller" @click="submitPhoto('custom')">使用照片</div>
        </div>

      </div>

      <!-- 抓拍 -->
      <canvas id="mycanvas" style="display: none;"></canvas>
    </div>
  </div>

</template>

<style lang="less" scoped>
.uploadFacePic {
  .img {
    width: 100%;
    height: 100%
  }

  .bigger {
    font-weight: 600;
    font-size: 3em;
  }

  .small {
    font-size: 2em;
  }

  .smaller {
    font-size: 1.2em;
  }

  .control {
    width: 100%;
    position: fixed;
    left: 0;
    bottom: 0;
    top: 75vh;
    right: 0;
    background: black;

    .control_before {
      position: relative;
      width: 100%;
      height: 100%;
      display: flex;
      flex-direction: column;

      .control_before_top {
        z-index: 1;
        flex: 1;
        color: orange;
        display: flex;
        justify-content: center;
        align-items: center;
      }

      .control_before_bottom {
        flex: 2;
        display: flex;
        justify-content: space-around;
        color: white;
        align-items: center;
        margin-bottom: 1.5em;
      }
    }

    .control_after {
      position: relative;
      width: 100%;
      height: 100%;
      display: flex;
      color: white;
      align-items: center;
      justify-content: space-around;
    }
  }

  main {
    margin-top:20px;
    display: flex;
    justify-content: space-around;
    text-align: center;
  }

  .btn {
    height: 34px;
    width: 45vw;
    background: #f68618;
    color: white;
    display: flex;
    justify-content: center;
    align-items: center;
    border-radius: 1em;
    border: none;
  }

  .customCamera {
    width: 100%; 
    position: fixed; 
    left: 0; 
    bottom: 0; 
    top: 0; 
    right: 0; 
    background:black;

    .fixed {
      width: 100vw;
      position: fixed;
      top: 10vh;
      left: 0;
    }
  }
}
</style>