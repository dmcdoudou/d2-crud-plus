<template>
  <div class="d2p-file-uploader" :class="{'is-disabled':disabled}">
  <el-upload :class="uploadClass"
             :file-list="fileList"
             :disabled="disabled"
             :http-request="httpRequest"
             :on-exceed="onExceed"
             :on-remove="handleUploadFileRemove"
             :on-success="handleUploadFileSuccess"
             ref="fileUploader"
             v-bind="_elProps"
  >
    <el-button :disabled="disabled" :size="btnSize" type="primary" v-if="_elProps.listType === 'text' || this._elProps.listType === 'picture'">{{btnName}}</el-button>
    <div class="avatar-item-wrapper"  v-else-if="this._elProps.listType === 'picture-card'"> <i class="el-icon-plus avatar-uploader-icon" ></i></div>
    <template v-else-if="_elProps.listType ===  'avatar'">
      <div class="avatar-item-wrapper">
        <img :src="avatarUrl" class="avatar" v-if="avatarUrl!=null">
        <i class="el-icon-plus avatar-uploader-icon" v-else></i>
      </div>
    </template>
  </el-upload>
    <el-dialog :visible.sync="dialogVisible" append-to-body >
      <div style="text-align: center">
        <img  style="max-width: 100%;" :src="dialogImageUrl" alt="">
      </div>
    </el-dialog>
  </div>
</template>

<script>
import SparkMD5 from 'spark-md5'
import D2pUploader from '../../uploader'
import lodash from 'lodash'
import { d2CrudPlus } from '../../utils/d2-crud-plus'
// 文件上传组件,依赖D2pUploader
export default {
  name: 'd2p-file-uploader',
  mixins: [d2CrudPlus.inputBase],
  props: {
    // 选择文件按钮的大小
    btnSize: { default: 'small' },
    // 选择文件按钮的名称
    btnName: { default: '选择文件' },
    // 可选哪些类型的文件
    accept: {},
    // 上传后端类型，[cos,qiniu,alioss,form]
    type: {
      type: String,
      default: 'cos' // 上传类型：form cos qiniu  alioss
    },
    // 值：url<br/>
    // 或 [url1,url2]<br/>
    // 或 {url:'url',md5:'',size:number}<br/>
    // 或 [{url:'url',md5:'',size:number}]<br/>
    // <br/>
    // limit=1 时 input事件返回 {url:'url',md5:'',size:number}<br/>
    // limit>1 时 input事件返回 数组<br/>
    value: {
      type: [String, Array, Object]
    },
    // 样式后缀 追加到url的后面，进行图片处理，需要到对象存储平台配置样式
    suffix: {
      type: String,
      required: false
    },
    // 返回类型: url=仅返回链接, object=包含md5和size , key=仅返回文件key
    returnType: {
      type: String,
      default: 'url'
    },
    // 自定义参数
    custom: {
      type: Object
    },
    // 内部封装[el-upload](https://element.eleme.cn/#/zh-CN/component/upload)组件的属性参数<br/>
    // 注意，form方式上传的action、name、headers等参数不在此设置
    elProps: {
      type: Object
    },
    // 文件大小限制 <br/>
    // 如果传入{limit,tip(fileSize,limit){vm.$message('可以自定义提示')}}
    sizeLimit: {
      type: Number, Object
    },
    // 构建下载url方法
    buildUrl: {
      type: Function,
      default: function (value, item) {
        return (typeof value === 'object') ? item.url : value
      }
    },
    // 上传组件参数，会临时覆盖全局上传配置参数[d2p-uploader](/guide/extends/uploader.html)
    uploader: {
      type: Object,
      default () {
        return {}
      }
    }
  },
  data () {
    return {
      fileList: [],
      context: {},
      dialogImageUrl: '',
      dialogVisible: false
    }
  },
  created () {
    this.initValue()
  },
  watch: {
    value (val) {
      let arr = []
      if (typeof val === 'string') {
        arr.push(val)
      } else {
        arr = val
      }
      let changed = false
      if (this.fileList.length === arr.length) {
        for (let i = 0; i < arr.length; i++) {
          let curUrl = this.fileList[i].url
          let curValue = this.fileList[i].response ? this.fileList[i].response.value : this.fileList[i].value
          let newUrl = typeof arr[i] === 'string' ? arr[i] : arr[i].url
          if (newUrl !== curUrl && newUrl !== curValue) {
            changed = true
            break
          }
        }
      } else {
        changed = true
      }
      if (changed) {
        this.initValue()
      }
    }
  },
  computed: {
    _elProps () {
      let defaultElProps = this.getDefaultElProps()
      Object.assign(defaultElProps, this.elProps)
      return defaultElProps
    },
    avatarUrl () {
      if (this.fileList.length > 0) {
        let file = this.fileList[0]
        if (file.response != null && file.response.url != null) {
          return file.response.url
        } else if (file.url != null) {
          return file.url
        }
      }
      return null
    },
    uploadClass () {
      if (this._elProps.listType === 'avatar') {
        return 'avatar-uploader'
      } else if (this._elProps.listType === 'picture-card') {
        return 'image-uploader'
      }
      return 'file-uploader'
    }
  },
  methods: {
    getDefaultElProps () {
      return {
        limit: 0,
        listType: 'text',
        showFileList: true,
        action: '',
        onPreview: (file) => {
          if (this._elProps.listType === 'picture-card' || this._elProps.listType === 'picture') {
            this.dialogImageUrl = file.url
            this.dialogVisible = true
          } else {
            window.open(file.url)
          }
        },
        beforeUpload: (file) => {
          if (this.sizeLimit == null) {
            return true
          }
          let limit = this.sizeLimit
          let showMessage = null
          if (typeof limit === 'number') {
            limit = this.sizeLimit
            showMessage = (fileSize, limit) => {
              if (this.$message) {
                let limitTip = this.computeFileSize(limit)
                let fileSizeTip = this.computeFileSize(file.size)
                this.$message({ message: '文件大小不能超过' + limitTip + '，当前文件大小:' + fileSizeTip, type: 'warning' })
              }
            }
          } else {
            limit = this.sizeLimit.limit
            showMessage = this.sizeLimit.tip
          }
          if (file.size > limit) {
            console.log('文件大小超过限制：', file.size)
            showMessage(file.size, limit)
            return false
          }
        }
      }
    },
    setValue (value) {
      this.initValue(value)
      // this.$emit('change', this.fileList)
    },
    getUploader () {
      let type = this.type
      if (this.uploader != null && this.uploader.type != null) {
        type = this.uploader.type
      }
      return D2pUploader.getUploader(type)
    },
    initValue (value) {
      if (value == null) {
        value = this.value
      }
      let fileList = []
      if (value == null) {
      } else if (typeof (value) === 'string') {
        if (value !== '') {
          let fileName = value.substring(value.lastIndexOf('/') + 1)
          fileList = [{ value: value, name: fileName }]
        }
      } else if (value instanceof Array) {
        if (value.length > 0 && typeof (value[0]) === 'string') {
          let tmp = []
          value.forEach(item => {
            let fileName = item.substring(item.lastIndexOf('/') + 1)
            tmp.push({ value: item, name: fileName })
          })
          fileList = tmp
        } else {
          fileList = value
        }
      } else if (value instanceof Object) {
        fileList = [value]
      }
      for (let item of fileList) {
        if (item.value == null) {
          item.value = item.url
        }
        item.url = this.buildUrl(item.value, item)
      }
      this.resetFileList(fileList)
    },
    computeFileSize (fileSize) {
      let sizeTip = fileSize
      if (fileSize > (1024 * 1024 * 1024)) {
        sizeTip = (fileSize / (1024 * 1024 * 1024)).toFixed(2) + 'G'
      } else if (fileSize > (1024 * 1024)) {
        sizeTip = (fileSize / (1024 * 1024)).toFixed(2) + 'M'
      } else {
        sizeTip = Math.round(fileSize / (1024)) + 'K'
      }
      return sizeTip
    },
    resetFileList (fileList) {
      this.$set(this, 'fileList', fileList)
    },
    handleUploadFileSuccess (res, file, fileList) {
      res.size = res.size != null ? res.size : file.size
      res.name = res.name != null ? res.name : file.name
      res.value = this.getReturnValue(res)
      let value = this.returnType === 'object' ? res.url : res.value
      const url = this.buildUrl(value, res)
      file.url = res.url = url
      this.resetFileList(fileList)
      this.$emit('success', res, file)
      let list = []
      for (let item of fileList) {
        if (item.response != null && item.response.url != null) {
          list.push({ ...item.response })
        } else {
          list.push(item)
        }
      }
      console.log('handleUploadFileSuccess list', list, res)
      this.emit(res, list)
    },
    handleUploadFileRemove (file, fileList) {
      this.fileList = fileList
      this.emitList(fileList)
    },
    emit (res, list) {
      if (this._elProps.limit === 1) {
        this.$emit('input', res.value)
        this.$emit('change', res.value)
      } else {
        this.emitList(list)
      }
    },
    emitList (list) {
      const tmp = []
      list.forEach(item => {
        tmp.push(item.value)
      })
      list = tmp
      this.$emit('input', list)
      this.$emit('change', list)
    },
    getReturnValue (item) {
      const value = item[this.returnType]
      if (value != null) {
        return value
      }
      return item
    },
    httpRequest (option) {
      Promise.all([
        this.doUpload(option),
        this.computeMd5(option.file)
      ]).then((ret) => {
        // 得到组合结果， size，md5
        let result = ret[0]
        result.md5 = ret[1]
        option.onSuccess(result)
      })
    },
    doUpload (option) {
      let config = this.uploader
      if (config == null) {
        config = {}
      }
      if (!lodash.isEmpty(this._elProps.action)) {
        config.action = this._elProps.action
      }
      if (!lodash.isEmpty(this._elProps.name)) {
        config.name = this._elProps.name
      }
      if (!lodash.isEmpty(this._elProps.data)) {
        config.data = this._elProps.data
      }
      if (!lodash.isEmpty(this._elProps.headers)) {
        config.headers = this._elProps.headers
      }
      if (!lodash.isEmpty(this.custom)) {
        config.custom = this.custom
      }
      return this.getUploader().then(uploader => {
        return uploader.upload({
          file: option.file,
          fileName: option.file.name,
          onProgress: option.onProgress,
          onError: option.onError,
          config: config
        })
      }).then(ret => {
        if (this.suffix != null) {
          ret.url += this.suffix
        }
        return ret
      })
    },
    onExceed (files, fileList) {
      console.log('文件数量超出限制')
      if (this._elProps.limit === 1) {
        this.clearFiles()
        this.$refs.fileUploader.handleStart(files[0])
        this.$refs.fileUploader.submit()
        return
      }
      this.$message({
        showClose: true,
        message: '已达最大限制数量，请删除一个文件后再上传',
        type: 'warning'
      })
    },
    clearFiles () {
      if (this.$refs.fileUploader != null) {
        this.$refs.fileUploader.clearFiles()
      }
    },
    getFileList () {
      return this.fileList
    },
    computeMd5 (file) {
      return new Promise((resolve, reject) => {
        let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
        let chunkSize = 2097152 // Read in chunks of 2MB
        let chunks = Math.ceil(file.size / chunkSize)
        let currentChunk = 0
        let spark = new SparkMD5.ArrayBuffer()
        let fileReader = new FileReader()

        fileReader.onload = (e) => {
          spark.append(e.target.result) // Append array buffer
          currentChunk++

          if (currentChunk < chunks) {
            loadNext()
          } else {
            let md5 = spark.end()
            console.info('computed hash', md5) // Compute hash

            resolve(md5)
          }
        }

        fileReader.onerror = function (error) {
          // eslint-disable-next-line prefer-promise-reject-errors
          reject('md5 computer error', error)
        }

        function loadNext () {
          let start = currentChunk * chunkSize
          let end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize

          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end))
        }

        loadNext()
      })
    }
  }
}
</script>
<style lang="scss">
  .d2p-file-uploader{
    &.is-disabled{
      .avatar-item-wrapper{
        background-color: #F5F7FA;
        border-color: #E4E7ED;
        color: #C0C4CC;
        cursor: not-allowed
      }
      li{
        cursor: not-allowed
      }
      .el-upload-list__item-actions{
        cursor: not-allowed
      }
    }
    .avatar-uploader{
      display: flex;
      .el-upload{
        width: 100px;
        height: 100px;
        /*display: flex;*/
        /*justify-content: center;*/
        /*align-items: center;*/
        background-color: #fbfdff;
        border: 1px dashed #c0ccda;
        border-radius: 6px;
      }
      .el-upload img{
        max-width: 100px;
        max-height: 100px;
      }
      .el-icon-plus.avatar-uploader-icon {
        vertical-align: top;
        font-size: 28px;
        color: #8c939d;
        line-height: 100px;
      }
    }
    .el-upload--picture-card .el-icon-plus.avatar-uploader-icon {
      vertical-align: top;
      font-size: 28px;
      color: #8c939d;
      line-height: 100px;
    }
    .image-uploader .el-upload-list--picture-card .el-upload-list__item-thumbnail {
      max-width: 100%;
      max-height: 100%;
      width:auto;
      height: auto;
    }
     .el-upload-list--picture .el-upload-list__item-thumbnail {
      max-height: 100%;
      height: auto;
    }
    .image-uploader .el-upload-list--picture-card .el-upload-list__item {
      /*display: flex;*/
      /*justify-content: center;*/
      /*align-items: center;*/
      text-align: center;
      line-height: 125px;
    }
    .image-uploader{
      /*display: flex;flex-wrap: wrap;*/
      .el-upload-list--picture-card .el-upload-list__item-actions{
        line-height: 100px;
      }
      .el-upload-list--picture-card {
        /*display: flex;*/
        /*flex-wrap: wrap;*/
      }
      .el-upload-list__item-status-label{
        line-height: 1;
      }
    }

    .el-upload--picture-card {
      background-color: #fbfdff;
      border: 1px dashed #c0ccda;
      border-radius: 6px;
      box-sizing: border-box;
      width: 100px;
      height: 100px;
      cursor: pointer;
      /*display: flex;*/
      /*justify-content: center;*/
      /*align-items: center;*/
    }
    .el-upload-list--picture-card {
      .el-upload-list__item{
        width: 100px;
        height: 100px;
      }
      .el-progress-circle{
        width: 70px !important;
        height: 70px !important;
      }
      .el-progress{
        width: 70px !important;
        height: 70px !important;
      }
    }
    .avatar-item-wrapper{
      display: flex;
      justify-content: center;
      align-items: center;
      width: 100%;
      height: 100%;
    }
  }

</style>
