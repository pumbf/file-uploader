<template>
  <div id="app">
    <div>
      <input
        type="file"
        :disabled="status !== Status.wait"
        @change="handleFileChange"
      />
      <el-button @click="handleUpload" :disabled="uploadDisabled"
        >上传</el-button
      >
      <el-button @click="handleResume" v-if="status === Status.pause"
        >恢复</el-button
      >
      <el-button
        v-else
        :disabled="status !== Status.uploading || !container.hash"
        @click="handlePause"
        >暂停</el-button
      >
    </div>
    <div>
      <div>计算文件 hash</div>
      <el-progress :percentage="hashPercentage"></el-progress>
      <div>总进度</div>
      <el-progress :percentage="fakeUploadPercentage"></el-progress>
    </div>
    <el-table :data="data">
      <el-table-column
        prop="hash"
        label="切片hash"
        align="center"
      ></el-table-column>
      <el-table-column label="大小(KB)" align="center" width="120">
        <template v-slot="{ row }">
          {{ row.size | transformByte }}
        </template>
      </el-table-column>
      <el-table-column label="进度" align="center">
        <template v-slot="{ row }">
          <el-progress
            :percentage="row.percentage"
            color="#909399"
          ></el-progress>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
const SIZE = 10 * 1024 * 1024; // 切片大小
const BASE_URL = "http://10.14.31.79:8888";
// const BASE_URL = "http://127.0.0.1:8888";
const DefaultBucketName = "test";
const Status = {
  wait: "wait",
  pause: "pause",
  uploading: "uploading"
};

export default {
  name: "app",
  filters: {
    transformByte(val) {
      return Number((val / 1024).toFixed(0));
    }
  },
  data: () => ({
    Status,
    container: {
      file: null,
      hash: "",
      worker: null
    },
    hashPercentage: 0,
    data: [],
    requestList: [],
    status: Status.wait,
    // 当暂停时会取消 xhr 导致进度条后退
    // 为了避免这种情况，需要定义一个假的进度条
    fakeUploadPercentage: 0
  }),
  computed: {
    uploadDisabled() {
      return (
        !this.container.file ||
        [Status.pause, Status.uploading].includes(this.status)
      );
    },
    uploadPercentage() {
      if (!this.container.file || !this.data.length) return 0;
      const loaded = this.data
        .map(item => item.size * item.percentage)
        .reduce((acc, cur) => acc + cur);
      return parseInt((loaded / this.container.file.size).toFixed(2));
    }
  },
  watch: {
    uploadPercentage(now) {
      if (now > this.fakeUploadPercentage) {
        this.fakeUploadPercentage = now;
      }
    }
  },
  methods: {
    handlePause() {
      this.status = Status.pause;
      this.resetData();
    },
    resetData() {
      this.requestList.forEach(xhr => xhr?.abort());
      this.requestList = [];
      if (this.container.worker) {
        this.container.worker.onmessage = null;
      }
    },
    async handleResume() {
      this.status = Status.uploading;
      const { uploadedList } = await this.verifyUpload(
        DefaultBucketName,
        this.container.file.name,
        this.container.hash,
        this.container.file.size
      );
      await this.uploadChunks(uploadedList);
    },
    // xhr
    request({
      url,
      method = "post",
      data,
      headers = {},
      onProgress = e => e,
      requestList
    }) {
      return new Promise(resolve => {
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = onProgress;
        xhr.open(method, url);
        Object.keys(headers).forEach(key =>
          xhr.setRequestHeader(key, headers[key])
        );
        xhr.setRequestHeader(
          "Authorization",
          "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJuYmYiOjE2MDczMzA3NjUsInBheWxvYWQiOiJ0ZXN0IiwiaXNzIjoidW5paW4tbWluaW8iLCJsYWJlbCI6IjJiOGNiNGZiN2NhNjQwNzFhMjVhYmMxNjNmYTYyMzBkIiwiZXhwIjoxNjA3MzM3OTY1fQ.ONry6uT5O9fusZNJq2JgxmBEQqSOE3AMiv0YusNxEZ3N6Z9eqLEDWkRAQsMBo9Q6BGsXLgAXTDmfJ9G4W4ksRA"
        );
        xhr.send(data);
        xhr.onload = e => {
          // 将请求成功的 xhr 从列表中删除
          if (requestList) {
            const xhrIndex = requestList.findIndex(item => item === xhr);
            requestList.splice(xhrIndex, 1);
          }
          resolve({
            data: e.target.response
          });
        };
        // 暴露当前 xhr 给外部
        requestList?.push(xhr);
      });
    },
    // 生成文件切片
    createFileChunk(file, size = SIZE) {
      const fileChunkList = [];
      let cur = 0;
      while (cur < file.size) {
        fileChunkList.push({ file: file.slice(cur, cur + size) });
        cur += size;
      }
      return fileChunkList;
    },
    // 生成文件 hash（web-worker）
    calculateHash(fileChunkList) {
      return new Promise(resolve => {
        this.container.worker = new Worker("/hash.js");
        this.container.worker.postMessage({ fileChunkList });
        this.container.worker.onmessage = e => {
          const { percentage, hash } = e.data;
          this.hashPercentage = percentage;
          if (hash) {
            resolve(hash);
          }
        };
      });
    },
    handleFileChange(e) {
      const [file] = e.target.files;
      if (!file) return;
      this.resetData();
      Object.assign(this.$data, this.$options.data());
      this.container.file = file;
    },
    async handleUpload() {
      if (!this.container.file) return;
      this.status = Status.uploading;
      const fileChunkList = this.createFileChunk(this.container.file);
      this.container.hash = await this.calculateHash(fileChunkList);

      const { completed, completedChunks } = await this.verifyUpload(
        DefaultBucketName,
        this.container.file.name,
        this.container.hash,
        this.container.file.size
      );
      if (completed) {
        this.$message.success("秒传：上传成功");
        this.status = Status.wait;
        return;
      }
      this.data = fileChunkList.map(({ file }, index) => ({
        identifier: this.container.hash,
        index: index,
        hash: this.container.hash,
        chunk: file,
        size: file.size,
        percentage: completedChunks.includes(index) ? 100 : 0
      }));

      await this.uploadChunks(completedChunks);
    },
    // 上传切片，同时过滤已上传的切片
    async uploadChunks(uploadedList = []) {
      const requestList = this.data
        .filter(({ index }) => !uploadedList.includes(index))
        .map(({ chunk, index }) => {
          const formData = new FormData();
          formData.append("file", chunk);
          formData.append("chunkNumber", index);
          formData.append("bucketName", DefaultBucketName);
          formData.append("fileName", this.container.file.name);
          formData.append("identifier", this.container.hash);
          return { formData, index };
        })
        .map(async ({ formData, index }) =>
          this.request({
            url: BASE_URL + "/oss/break-point-upload",
            data: formData,
            onProgress: this.createProgressHandler(this.data[index]),
            requestList: this.requestList
          })
        );
      await Promise.all(requestList);
      // 之前上传的切片数量 + 本次上传的切片数量 = 所有切片数量时
      // 合并切片
    },
    // 根据 hash 验证文件是否曾经已经被上传过
    // 没有才进行上传
    async verifyUpload(bucketName, fileName, identifier, fileSize) {
      const { data } = await this.request({
        url: BASE_URL + "/oss/break-point-upload-init",
        headers: {
          "content-type": "application/json"
        },
        data: JSON.stringify({
          fileName,
          identifier,
          bucketName,
          fileSize
        })
      });
      console.log(data);
      return JSON.parse(data).data;
    },
    // 用闭包保存每个 chunk 的进度数据
    createProgressHandler(item) {
      return e => {
        item.percentage = parseInt(String((e.loaded / e.total) * 100));
      };
    }
  }
};
</script>
