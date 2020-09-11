<template>
  <div v-loading="loading" element-loading-text="文件正在合并中" class="video-upload">
    <sticky :z-index="10" class-name="sub-navbar">
      <el-button>显示帮助</el-button>
      <el-button style="margin-left: 10px;" type="success" @click="submitForm">
        新增视频
      </el-button>
    </sticky>
    <el-form :model="postForm" class="form-container">
      <el-row>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="电影名称：">
            <el-input
              v-model="postForm.name"
              placeholder="电影名称"
              style="width: 100%"
            />
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="电影类型：">
            <el-select v-model="postForm.type" placeholder="请选择电影类型">
              <el-option label="战争" value="war" />
              <el-option label="悬疑" value="suspense" />
            </el-select>
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="地区：">
            <el-select v-model="postForm.area" placeholder="请选择地区">
              <el-option label="华语" value="chinese" />
              <el-option label="香港地区" value="hongkong" />
            </el-select>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="年份：">
            <el-select v-model="postForm.date" placeholder="请选择年份">
              <el-option label="2020" value="2020" />
              <el-option label="2010" value="2010" />
            </el-select>
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="24">
          <input id="file-input" type="file" @change="change">
          <label for="file-input" class="label">
            <span>上传视频</span>
          </label>
        </el-col>
        <el-col :span="24">
          <div v-if="valPercentage !== 100" class="progress-val">
            <el-progress type="circle" :percentage="valPercentage" :width="50" />
            <span>正在校验资源中</span>
          </div>
          <div v-if="valPercentage === 100 && upPercentage !== 100" class="progress-val">
            <el-progress type="circle" :percentage="upPercentage" :width="50" />
            <span>正在上传</span>
          </div>
        </el-col>
      </el-row>
    </el-form>
  </div>
</template>

<script>
const SparkMD5 = require('spark-md5')

import request from '@/utils/request'

import Sticky from '../../components/Sticky'

export default {
  components: { Sticky },
  data() {
    return {
      chunkSize: 5 * 1024 * 1024,
      fileSize: 0,
      file: '',
      hasUploaded: 0,
      chunks: 0,
      valPercentage: 0,
      upPercentage: 0,
      loading: false,
      fileMd5Value: '',
      postForm: {},
      labelWidth: '120px'
    }
  },
  methods: {
    change(e) {
      console.log(e.target.files)
      this.file = e.target.files[0]
      this.fileSize = this.file.size
      this.init(this.file)
    },
    async init(file) {
      // 将文件分片加密转换成md5
      const fileMd5Value = await this.md5File(file)
      console.log(fileMd5Value)

      // 与后端进行校验，是否存在
      const result = await this.checkFileMD5(file.name, fileMd5Value)

      console.log(result)

      // 如果文件已存在, 就秒传
      if (result.data.file) {
        this.$message.success('文件已秒传')
        return
      }

      // 检查并上传MD5
      await this.checkAndUploadChunk(fileMd5Value, result.data.chunkList)

      this.fileMd5Value = fileMd5Value
    },
    submitForm() {
      if (this.fileMd5Value) {
        // 通知服务器所有分片已上传完成并完成合并
        this.loading = true
        this.notifyServer(this.fileMd5Value)
      }
    },
    md5File(file) {
      return new Promise((resolve, reject) => {
        // 切割文件
        const blobSlice =
          File.prototype.slice ||
          File.prototype.mozSlice ||
          File.prototype.webkitSlice

        // 切割成多少个
        const chunks = 100

        // 切割的文件的单个大小
        const chunkSize = file.size / chunks

        // 当前所在的切割文件的索引
        let currentChunk = 0
        const spark = new SparkMD5.ArrayBuffer()
        const fileReader = new FileReader()

        fileReader.onload = (e) => {
          // 加载文件流，并md5加密
          spark.append(e.target.result)

          currentChunk++

          // 进度条
          this.valPercentage = currentChunk

          if (currentChunk < chunks) {
            loadNext()
          } else {
            const result = spark.end()
            resolve(result)
          }
        }

        fileReader.onerror = () => {
          this.$message.error('文件读取失败')
        }

        function loadNext() {
          // 切割文件的开始位置
          const start = currentChunk * chunkSize
          // 切割文件的结束位置
          const end = start + chunkSize >= file.size ? file.size : start + chunkSize

          // 将切割的文件载入文件流
          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end))
        }

        loadNext()
      })
    },
    checkFileMD5(fileName, fileMd5Value) {
      return new Promise((resolve, reject) => {
        request({
          url: '/video/check/file',
          method: 'get',
          params: {
            fileName: fileName,
            fileMd5Value: fileMd5Value
          }
        }).then(data => {
          resolve(data)
        })
      })
    },
    async checkAndUploadChunk(fileMd5Value, chunkList) {
      // 总分片数
      this.chunks = Math.ceil(this.fileSize / this.chunkSize)
      // 当前正在上传分片的索引
      this.hasUploaded = chunkList.length

      for (let i = 0; i < this.chunks; i++) {
        const exit = chunkList.indexOf(i + '') > -1
        // 如果已经存在, 则不用再上传当前块
        if (!exit) {
          const index = await this.upload(i, fileMd5Value, this.chunks)
          this.hasUploaded++

          // 进度条
          this.upPercentage = Math.floor((this.hasUploaded / this.chunks) * 100)
          console.log('index==>', index)
        }
      }
    },
    upload(i, fileMd5Value, chunks) {
      return new Promise((resolve, reject) => {
        // 截取的分片结束位置
        const end = (i + 1) * this.chunkSize >= this.file.size ? this.file.size : (i + 1) * this.chunkSize

        const form = new FormData()

        form.append('data', this.file.slice(i * this.chunkSize, end)) // file对象的slice方法用于切出文件的一部分
        form.append('total', chunks) // 总片数
        form.append('index', i) // 当前是第几片
        form.append('fileMd5Value', fileMd5Value)
        request.post('/video/upload', form).then(res => {
          resolve(res)
        }, err => {
          console.log(err)
        })
      })
    },
    notifyServer(fileMd5Value) {
      request({
        url: '/video/merge',
        method: 'get',
        params: {
          md5: fileMd5Value,
          fileName: this.file.name,
          size: this.file.size
        }
      }).then(data => {
        console.log(data)
        this.loading = false
        this.$message.success(data.msg)
      })
    }
  }
}
</script>

<style lang="scss">
.video-upload {
  #file-input {
    display: none;
  }

  .form-container {
    padding: 20px 45px;
  }

  .label {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 120px;
    height: 40px;
    border-radius: 4px;
    background-color: #409eff;
    font-size: 14px;
    color: #fff;
    font-weight: 500;
    cursor: pointer;
    span {
      display: inline-block;
    }
  }

  .progress-val {
    display: flex;
    flex-direction: column;
    align-items: center;
  }
}
</style>
