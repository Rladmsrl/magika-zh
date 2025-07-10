<template>
  <v-card class="pt-4 pb-4 pl-4 pr-4 mx-auto" variant="outlined" color="primary">

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

    <div v-if="processedFiles && processedFiles.length > 0" class="mt-4">
      <div class="text-h6 mb-3">处理结果：</div>
      <div v-for="(file, index) in processedFiles" :key="index" class="mb-2">
        <v-card variant="outlined" class="pa-3">
          <div class="d-flex justify-space-between align-center">
            <div>
              <div v-if="file.newName" class="text-subtitle-1">{{ file.newName }}</div>
              <div v-else-if="file.unsupported" class="text-subtitle-1 text-error">{{ file.originalName }}</div>
              <div v-else class="text-subtitle-1 text-error">{{ file.originalName }}</div>

              <div class="text-caption">{{ file.originalName }}</div>
              <div v-if="file.unsupported" class="text-caption text-error">
                不支持的文件格式: {{ file.detectedType }} (仅支持图片和视频格式)
              </div>
              <div v-else-if="file.error" class="text-caption text-error">
                {{ file.error }}
              </div>
              <div v-else class="text-caption text-success">
                检测为: {{ file.detectedType }}
              </div>
            </div>
            <v-btn
              v-if="file.newName && file.data"
              color="primary"
              variant="contained"
              size="small"
              @click="downloadFile(file)"
            >
              下载
            </v-btn>
            <v-chip
              v-else-if="file.unsupported"
              color="error"
              size="small"
              variant="outlined"
            >
              不支持
            </v-chip>
            <v-chip
              v-else-if="file.error"
              color="error"
              size="small"
              variant="outlined"
            >
              错误
            </v-chip>
          </div>
        </v-card>
      </div>
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
const processedFiles = ref([]);

// 支持的图片和视频格式扩展名映射
const supportedExtensions = {
  // 图片格式
  'png': '.png',
  'jpg': '.jpg',
  'jpeg': '.jpeg',
  'gif': '.gif',
  'bmp': '.bmp',
  'webp': '.webp',
  'svg': '.svg',
  'ico': '.ico',
  'tiff': '.tiff',
  'tif': '.tif',
  'psd': '.psd',
  'raw': '.raw',
  'heic': '.heic',
  'avif': '.avif',
  // 视频格式
  'mp4': '.mp4',
  'avi': '.avi',
  'mov': '.mov',
  'wmv': '.wmv',
  'flv': '.flv',
  'webm': '.webm',
  'mkv': '.mkv',
  '3gp': '.3gp',
  'mpg': '.mpg',
  'mpeg': '.mpeg',
  'ogv': '.ogv',
  'ts': '.ts',
  'm4v': '.m4v',
  'asf': '.asf'
};

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

// 下载文件函数
const downloadFile = (processedFile) => {
  const blob = new Blob([processedFile.data], { type: 'application/octet-stream' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = processedFile.newName;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
};

// 生成新文件名
const generateNewFileName = (originalName, detectedType) => {
  const baseName = originalName.split('.')[0] || 'unknown';
  const extension = supportedExtensions[detectedType];
  return extension ? `${baseName}${extension}` : null;
};

watch(files, async () => {
  if (!files.value || files.value.length === 0) {
    results.value = [];
    processedFiles.value = [];
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
    return;
  }

  processedFiles.value = [];
  const newProcessedFiles = [];

  const processingPromises = files.value.map(async (file) => {
    try {
      const fileBytes = new Uint8Array(await file.arrayBuffer());
      const magikaResult = await magika.identifyBytes(fileBytes);

      const prediction = magikaResult?.prediction;
      const topLabel = prediction?.output?.label ?? 'unknown';

      // 检查是否是支持的格式
      if (supportedExtensions[topLabel]) {
        const newName = generateNewFileName(file.name, topLabel);
        if (newName) {
          newProcessedFiles.push({
            originalName: file.name,
            newName: newName,
            data: fileBytes,
            detectedType: topLabel
          });
        }
      } else {
        // 不支持的格式，添加到未处理列表
        newProcessedFiles.push({
          originalName: file.name,
          newName: null,
          data: null,
          detectedType: topLabel,
          unsupported: true
        });
      }
    } catch (error) {
      console.error(`Error processing file ${file.name}:`, error);
      newProcessedFiles.push({
        originalName: file.name,
        newName: null,
        data: null,
        error: `处理失败: ${error.message}`
      });
    }
  });

  await Promise.all(processingPromises);
  processedFiles.value = newProcessedFiles;

  const supportedCount = newProcessedFiles.filter(f => !f.unsupported && !f.error).length;
  const unsupportedCount = newProcessedFiles.filter(f => f.unsupported).length;

  if (supportedCount > 0 && unsupportedCount > 0) {
    message.value = `处理完成！${supportedCount}个文件可下载，${unsupportedCount}个文件格式不支持`;
  } else if (supportedCount > 0) {
    message.value = `处理完成！${supportedCount}个文件可下载`;
  } else if (unsupportedCount > 0) {
    message.value = `检测完成，但所有文件格式都不支持（仅支持图片和视频格式）`;
  } else {
    message.value = "文件处理完成";
  }

  setTimeout(() => {
    message.value = null;
  }, 5000);
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