<template>
  <div class="container">
    <!-- 左侧布局 -->
    <div class="sidebar">
      <div>
        <div v-for="(value, key) in configDict" :key="key" class="options-block">
          <h2>{{ key }}</h2>
          <div v-if="value.length === 0">没有 {{ key }}</div>
          <el-radio-group v-model="configValue[key]">
            <el-radio v-for="(name, idx) in value" :key="idx" :label="name">{{ name }}</el-radio>
          </el-radio-group>
        </div>
      </div>
 
      <el-divider />
 
      <el-button class="custom-button" @click="clickButton">
            运行
    </el-button>


      <el-divider />

      <!-- 添加额外控件：滑块 -->
      <h3>额外功能: 调整参数</h3>
      <el-slider v-model="sliderValue" :min="0" :max="100"></el-slider>
      <div>当前滑块值: {{ sliderValue }}</div>
    </div>

    <!-- 右侧布局 -->
    <div class="main-content">
      <div>
        <h2>运行结果</h2>
        <div class="running-res">
          <div v-for="(content, idx) in runningOutput" :key="idx" class="output-block">
            <p v-if="content.type == 'string'" class="output-text">{{ content.content }}</p>
            <img
              v-if="content.type == 'image'"
              class="output-image"
              :src="'data:image/jpeg;base64,' + content.content"
            />
          </div>
        </div>
      </div>

      <el-divider />

      

      <!-- 示例：进度条 -->
      <h3>任务进度</h3>
      <el-progress :percentage="progress" status="success"></el-progress>
    </div>

    <!-- 弹窗设置：连接地址 -->
    <el-dialog v-model="showConnectInfoWindow" title="连接地址" :show-close="false">
      <el-form label-width="40px">
        <el-form-item label="Url">
          <el-input v-model="connectUrl" />
        </el-form-item>
        <el-form-item label="Port">
          <el-input v-model="connectPort" />
        </el-form-item>
      </el-form>
      <el-button type="primary" @click="onClickConnect">连接</el-button>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ElMessage } from 'element-plus'
import { ref } from 'vue'

const connectUrl = ref('localhost')
const connectPort = ref('8765')
const showConnectInfoWindow = ref(false)

interface DlFrameConfigInterface {
  [key: string]: string
}
interface DlFrameInspectionInterface {
  [key: string]: Array<string>
}

let ws: WebSocket | null = null
const connect = () => {
  ws = new WebSocket('ws://' + connectUrl.value + ':' + connectPort.value)
  ws.onopen = () => {
    isConnectedToServer.value = true
    showConnectInfoWindow.value = false
    ElMessage({
      message: '连接成功',
      type: 'success',
    })
    ws?.send(
      JSON.stringify({
        type: 'overview',
        params: {},
      })
    )
  }
  ws.onmessage = (evt) => {
    var received_msg = JSON.parse(evt.data)

    if (received_msg.status === 200) {
      const received_msg_data = received_msg.data
      if (received_msg.type === 'overview') {
        configDict.value = received_msg.data

        const tmpDict: DlFrameConfigInterface = {}
        for (let i in configDict.value) {
          tmpDict[i] = ''
        }
        configValue.value = tmpDict
        runningOutput.value = []
      } else if (received_msg.type === 'print') {
        runningOutput.value.push({
          type: 'string',
          content: received_msg_data.content,
        })
        updateProgress(20)
      } else if (received_msg.type === 'imshow') {
        runningOutput.value.push({
          type: 'image',
          content: received_msg_data.content,
        })
        updateProgress(40)
      }
    } else {
      console.error(received_msg.data)
    }
  }
  ws.onclose = () => {
    isConnectedToServer.value = false
    showConnectInfoWindow.value = true
    ElMessage.error('连接已断开')
  }
  ws.onerror = () => {
    ElMessage.error('连接失败(地址错误 / 协议错误 / 服务器错误)')
    showConnectInfoWindow.value = true
  }
}

connect()

const configDict = ref<DlFrameInspectionInterface>({})
const configValue = ref<DlFrameConfigInterface>({})

const isConnectedToServer = ref(false)
const onClickConnect = () => {
  connect()
}

const clickButton = () => {
  if (!isConnectedToServer.value) {
    ElMessage.error('与 server 的连接已断开。请重启 python 服务后刷新页面')
    return
  }
  for (let k in configDict.value) {
    if (configDict.value[k].length === 0) {
      ElMessage.error('没有 ' + k)
      return
    }
  }
  for (let k in configValue.value) {
    if (configValue.value[k] == '') {
      ElMessage.error('您的选项不完整')
      return
    }
  }
  runningOutput.value = []
  ws?.send(
    JSON.stringify({
      type: 'run',
      params: configValue.value,
    })
  )
}



// 滑块值
const sliderValue = ref(50)

// 示例：进度条
const progress = ref(0)
const updateProgress = (value: number) => {
  progress.value = Math.min(progress.value + value, 100)
}

// 运行输出结果


interface RunningOutputInterface {
  [key: string]: string;
}
const runningOutput = ref<Array<RunningOutputInterface>>([]);
</script>



<style scoped>
.container {
  display: flex;
  flex-direction: row;
  width: 100%;
  height: 100vh;
  
}

.sidebar {
	
  width: 30%;
  padding: 20px;
  overflow-y: auto;
  background-color: #f7f7f7;
  border-right: 1px solid #ddd;
  box-shadow: 2px 0 5px rgba(0, 0, 0, 0.1);
}

.main-content {
	
  width: 70%;
  padding: 20px;
  overflow-y: auto;
  background-color: #fff;
  border-left: 1px solid #ddd;
  box-shadow: -2px 0 5px rgba(0, 0, 0, 0.1);
}
.sidebar::before,
.main-content::before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(90deg, #03a9f4, #f441a5, #ffeb3b, #03a9f4);
  background-size:1000%;
  border-radius: inherit; /* 继承父元素的border-radius */
  z-index: -1; /* 确保伪元素位于内容之后 */
  animation: sun 8s infinite;
}
 
@keyframes sun {
  0% {
    background-position: 0% 0;
  }
  100% {
    background-position: -400% 0;
  }
}
 

.options-block {
  margin-bottom: 20px;
  padding: 10px;
  background-color: #fff;
  border: 1px solid #ddd;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.running-res {
  text-align: left;
  width: 100%;
  font-size: large;
  background: #f0f0f0;
  border: 1px solid #ddd;
  border-radius: 5px;
  padding: 10px;
}

.output-block {
  margin-top: 20px;
}

.output-item {
  margin-bottom: 15px;
  padding: 15px;
  background: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.output-image {
  max-width: 100%;
  margin-top: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
}

.el-slider,
.el-progress {
  margin: 20px 0;
}

.custom-button {
    text-decoration: none;  
	position: relative;
	    display: inline-flex;
	    align-items: center;
	    justify-content: center;
        
        /* 设置字体大小 */
        font-size: 24px;
        background: linear-gradient(90deg,#03a9f4,#f441a5,#ffeb3b,#03a9f4);
        background-size: 400%;
        /* 使用百分比设置宽度和高度 */
            width: 20vw; /* 假设窗口宽度为100vw，则按钮宽度为20% */
            height: 5vw; /* 假设窗口高度为100vh，但这里我们使用vw以保持比例，按钮高度为5% */
            line-height: 5vw; /* 与高度相同，以保持文本垂直居中 */
        text-align: center;
        color: #fff;
        /* 字母变大写 */
        text-transform: uppercase;
        /* 设置成胶囊状 */
        border-radius: 50px;
        z-index: 1;
    
}

.custom-button::before {
    content: "";
    position: absolute;
    left: -5px;
    right: -5px;
    top: -5px;
    bottom: -5px;
    background: linear-gradient(90deg, #03a9f4, #f441a5, #ffeb3b, #03a9f4);
    background-size: 400%;
    border-radius: 50px;
    filter: blur(20px);

    z-index: -1;
}

.custom-button:hover::before,
.custom-button:hover {
    animation: sun 8s infinite;
}

@keyframes sun {
    100% {
        background-position: -400% 0;
    }
}
</style>
