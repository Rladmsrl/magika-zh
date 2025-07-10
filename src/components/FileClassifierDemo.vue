<template>
  <v-card class="pt-4 pb-4 pl-4 pr-4 mx-auto" variant="outlined" color="primary">
    <v-card-text class="text-center pb-6">
      <div class="text-h5 mb-3">文件类型检测</div>
      <div class="text-body-1">
        拖拽文件到下方区域或点击选择文件，AI将自动检测文件类型。
        <br>文件处理完全在浏览器本地进行，不会上传到服务器。
      </div>
    </v-card-text>
    
    <div 
      class="drop-zone" 
      :class="{ 'drag-over': isDragOver }"
      @drop.prevent="handleDrop"
      @dragover.prevent="handleDragOver"
      @dragenter.prevent="handleDragEnter"
      @dragleave.prevent="handleDragLeave"
      @click="triggerFileInput"
    >
      <v-file-input 
        v-model="files" 
        variant="solo-filled" 
        multiple 
        show-size 
        counter
        label="拖拽文件到此处或点击选择文件"
        ref="fileInput"
        style="display: none;"
      />
      
      <div class="drop-zone-content">
        <v-icon size="48" class="mb-3" color="primary">mdi-cloud-upload</v-icon>
        <div class="text-h6 mb-2">拖拽文件到此处</div>
        <div class="text-body-2">或点击选择文件</div>
        <div class="text-caption mt-2">支持多文件同时上传</div>
      </div>
    </div>
    
    <v-alert v-if="message" type="info" :text="message" variant="tonal" class="mt-4 mb-3"></v-alert>
    
    <div v-if="files && files.length > 0" class="mt-4">
      <div class="text-h6 mb-3">检测结果：</div>
      <bars-visualization v-for="(file, index) in files" :key="index" :file="file" :resultData="results[index]" />
    </div>
  </v-card>
</template>

<script setup>
import { onMounted, ref, watch } from "vue";
import { Magika } from "magika";

import BarsVisualization from "./BarsVisualization.vue";

const MAGIKA_MODEL_VERSION = "standard_v3_3"
const MAGIKA_MODEL_URL = `${import.meta.env.BASE_URL}models/${MAGIKA_MODEL_VERSION}/model.json`;
const MAGIKA_MODEL_CONFIG_URL = `${import.meta.env.BASE_URL}models/${MAGIKA_MODEL_VERSION}/config.min.json`;

const files = ref([]);
const results = ref([]);
const message = ref();
const isDragOver = ref(false);
const fileInput = ref(null);

let magika = undefined;

// 拖拽事件处理
const handleDragOver = (e) => {
  e.preventDefault();
  isDragOver.value = true;
};

const handleDragEnter = (e) => {
  e.preventDefault();
  isDragOver.value = true;
};

const handleDragLeave = (e) => {
  e.preventDefault();
  isDragOver.value = false;
};

const handleDrop = (e) => {
  e.preventDefault();
  isDragOver.value = false;
  
  const droppedFiles = Array.from(e.dataTransfer.files);
  if (droppedFiles.length > 0) {
    files.value = droppedFiles;
  }
};

const triggerFileInput = () => {
  fileInput.value?.$el?.querySelector('input')?.click();
};

watch(files, async () => {
  if (!files.value || files.value.length === 0) {
    results.value = [];
    return;
  }

  try {
    if (magika === undefined) {
      message.value = "正在加载AI模型...";
      magika = await Magika.create({
        modelURL: MAGIKA_MODEL_URL,
        configURL: MAGIKA_MODEL_CONFIG_URL,
      });
      message.value = "模型加载完成，正在分析文件...";
    } else {
      message.value = "正在分析文件...";
    }
  } catch (error) {
    console.error("Failed to load Magika model:", error);
    message.value = "模型加载失败，请刷新页面重试";
    results.value = files.value.map(() => ({ topLabel: null, scores: null, error: "模型加载失败" }));
    return;
  }

  results.value = new Array(files.value.length).fill(null);

  const processingPromises = files.value.map(async (file, fileIndex) => {
    try {
      const fileBytes = new Uint8Array(await file.arrayBuffer());
      const magikaResult = await magika.identifyBytes(fileBytes);

      const prediction = magikaResult?.prediction;
      const topLabel = prediction?.output?.label ?? 'unknown';
      const scoresMap = prediction?.scores_map ?? {};

      results.value[fileIndex] = {
        modelVersion: magika.getModelName(),
        topLabel: topLabel,
        scores: scoresMap
      };
    } catch (error) {
      console.error(`Error processing file ${file.name}:`, error);
      results.value[fileIndex] = {
        topLabel: null,
        scores: null,
        error: `处理失败: ${error.message}`
      };
    }
  });

  await Promise.all(processingPromises);
  message.value = "文件分析完成！";
  setTimeout(() => {
    message.value = null;
  }, 3000);
});

onMounted(async () => {
  message.value = "正在初始化AI模型...";
  try {
    if (magika === undefined) {
      magika = await Magika.create({
        modelURL: MAGIKA_MODEL_URL,
        configURL: MAGIKA_MODEL_CONFIG_URL,
      });
    }
    message.value = "模型初始化完成！拖拽或选择文件开始检测";
    setTimeout(() => {
      message.value = null;
    }, 3000);
  } catch (error) {
    console.error("Failed to initialize Magika on mount:", error);
    message.value = "模型初始化失败，请刷新页面重试";
  }
});
</script>

<style scoped>
.drop-zone {
  border: 2px dashed #ccc;
  border-radius: 12px;
  padding: 40px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background-color: #fafafa;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.drop-zone:hover {
  border-color: #1976d2;
  background-color: #f5f5f5;
}

.drop-zone.drag-over {
  border-color: #1976d2;
  background-color: #e3f2fd;
  border-style: solid;
  transform: scale(1.02);
}

.drop-zone-content {
  pointer-events: none;
}

.text-center {
  text-align: center;
}
</style>