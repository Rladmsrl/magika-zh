<template>
  <div class="text-center py-6">
    <div class="text-h3 pb-3 text-primary">文件类型检测工具</div>
    <div class="text-body-1 pb-6">
      使用AI技术检测文件类型，支持200多种文件格式
    </div>
    
    <div v-if="isSupported === true" class="pa-4">
      <FileClassifierDemo />
    </div>
    
    <v-alert v-else-if="isSupported === false" type="warning" variant="tonal" class="mt-4 mx-4" title="功能不支持"
      text="抱歉，您的浏览器不支持此功能。请升级您的浏览器或更换设备。">
    </v-alert>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import FileClassifierDemo from "@/components/FileClassifierDemo.vue";

function getMaxTexture() {
  var canvas = document.createElement('canvas');
  var gl = canvas.getContext('webgl');
  return gl.getParameter(gl.MAX_TEXTURE_SIZE);
}

function isInferenceSupported() {
  return getMaxTexture() >= 5804;
}

const isSupported = ref(null);

onMounted(() => {
  try {
    isSupported.value = isInferenceSupported();
  } catch (error) {
    console.error("Error checking browser support:", error);
    isSupported.value = false;
  }
});
</script>

<style scoped lang="scss">
.text-center {
  text-align: center;
}
</style>
