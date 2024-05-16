<script setup>
import html2canvas from 'html2canvas';
import { ref, reactive } from 'vue';

const options = [
  { text: '上传图片', value: 0 },
  { text: '上传蒙版', value: 1 },
];
const containerRef = ref(null);
const state = reactive({
  maskImageUrl: '',
  imageUrl: '',
  menu: 1,
})

const afterRead = (file) => {
  const objectUrl = file.objectUrl;
  if (state.menu) {
    state.menu = 0
    state.maskImageUrl = objectUrl;
  } else {
    state.imageUrl = objectUrl;
    state.menu = 1
  }
};

const download = async () => {
  const el = document.getElementById('container');
  html2canvas(el).then(canvas => {
    canvas.toBlob((blob) => {
      downloadImage(blob)
    })
  })
}

const downloadImage = (_url, status = false) => {
  var a = document.createElement('a');
  const url = status ? _url : window.URL.createObjectURL(_url);
  a.href = url;
  a.download = new Date().getTime();
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  !status && URL.revokeObjectURL(url);  
}

const imageLoad = (e) => {
  containerRef.value.style.width = e.naturalWidth + 'px';
  containerRef.value.style.height = e.naturalHeight + 'px';
}
</script>

<template>
  <main>
    <van-dropdown-menu>
      <van-dropdown-item v-model="state.menu" :options="options" />
    </van-dropdown-menu>
    <van-uploader :after-read="afterRead" />
    <van-button type="primary" size="small" @click="download">下载蒙版图片</van-button>
    <van-button type="primary" size="small" @click="downloadImage(state.imageUrl, true)">下载图片</van-button>

    <div class="container" id="container" ref="containerRef">
      <!-- fit="contain" -->
      <van-image class="maskImageUrl" :src="state.maskImageUrl" @load="imageLoad($event.target)" />
      <van-image class="imageUrl" :src="state.imageUrl" />
    </div>
  </main>
</template>

<style scoped>
main {
  width: 100%;
  height: 100%;
  /* background-color: #F3F3F3; */
}

.container {
  width: 100%;
  height: 500px;
  position: relative;
}

.container .van-image {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
}

.container .maskImageUrl {
  z-index: 9;
}

.container .imageUrl {
  z-index: 1;
}
</style>
