<script setup lang='ts'>
import { useI18n } from 'vue-i18n'
import { useDisplay } from 'vuetify'
import { api, ioUrl } from '@/utils/api'
import { useRouter } from 'vue-router'
import { io } from 'socket.io-client'

const router = useRouter()
const { t } = useI18n()

const imageSrc = ref('https://cdn.devdojo.com/images/november2020/hero-image.jpeg')
//  {data: 'What is the capital of Australia?', choice: Array(4), remaining_time: 941, stage: 1}
type GameData = {
  data: string,
  choice: Array<string>,
  remaining_time: number,
  stage: number
}

const loading = ref(true)
const gameData = ref<GameData>({
  data: '',
  choice: [],
  remaining_time: 0,
  stage: 0
})
const disabled = ref(false)
const gameOver = ref(false)
const rankListDialog = ref(false)

// 创建Socket.IO连接
const socket = io(ioUrl, {
  // 携带token进行连接
  auth: {
    token: localStorage.getItem('token')
  },
  transports: ['websocket'] // 强制使用WebSocket传输
})

socket.on('connect', () => {
  console.log('Connected to server')

})

socket.on('change_question', () => {
  disabled.value = true
  getRankListDialog();
  setTimeout(() => {
    rankListDialog.value = false
  }, 4000)
})

socket.on('started_update_data', async (data) => {
  disabled.value = true
  // 处理从服务器接收的数据更新
  console.log('started_update_data: Received data update from server:', data)
  if (data['data'] === 'pending') {
    loading.value = true
    return
  }
  if (data['data'] === 'over') {
    loading.value = false
    gameOver.value = true
    return
  }
  // 更新本地状态或UI等
  gameOver.value = false
  gameData.value = data
  loading.value = false
  const response: any = await api.get('/answer')
  console.log(response)
  if (response.data['submitted'] === false) {
    disabled.value = false
  }
})


socket.on('update_data', (data) => {
  disabled.value = false
  // 处理从服务器接收的数据更新
  console.log('Received data update from server:', data)
  if (data['data'] === 'pending') {
    loading.value = true
    return
  }
  if (data['data'] === 'over') {
    loading.value = false
    gameOver.value = true
    return
  }
  // 更新本地状态或UI等
  gameOver.value = false
  gameData.value = data
  loading.value = false
})

//  {remaining_time: 660, stage: 1}
socket.on('time_update', (data) => {
  // 处理从服务器接收的数据更新
  console.log('Received data update from server:', data)
  // 更新本地状态
  gameData.value.remaining_time = data.remaining_time
  gameData.value.stage = data.stage
})

socket.on('disconnect', (reason) => {
  console.log('Disconnected from server:', reason)
  // 处理自动重连
  setTimeout(() => {
      socket.connect()
    }
    , 5000)
})

// 格式为 00:00, 需要补零
function formatTime(time: number) {
  const minutes = Math.floor(time / 60)
  let seconds = time % 60
  return `${minutes}:${seconds < 10 ? `0${seconds}` : seconds}`
}

async function answerQuestion(answer: string) {
  console.log('Answer:', answer)
  const response = await api.post('/answer', { answer: answer, question: gameData.value.data })
  if (response.data['status'] === 'success') {
    disabled.value = true
  } else {
    alert('提交失败')
  }
}


const rankList = ref([])

async function getRankListDialog() {
  rankListDialog.value = true
  const response = await api.post('/lastQuestionRank')
  if (response.data['data'].length > 0) {
    rankList.value = response.data['data']
  } else {
    rankList.value = []
  }
}

</script>

<template>
  <v-container class="fill-height">
    <div v-if="loading" class="fill-height d-flex align-center justify-center w-100">
      <v-col>
        <v-col class="w-100 text-center">等待游戏开始</v-col>
        <v-col>
          <v-progress-linear
            color="deep-purple-accent-4"
            indeterminate
            rounded
            height="6"
          ></v-progress-linear>
        </v-col>
      </v-col>
    </div>
    <v-card class="h-100" v-else-if="!gameOver">
      <div class="d-flex flex-column h-100">
        <v-toolbar>
          <!-- rank button-->
          <v-dialog max-width="500" v-model="rankListDialog">
            <template v-slot:activator="{ props: activatorProps }">
              <v-btn icon @click="getRankListDialog">
                <v-icon>mdi-trophy</v-icon>
              </v-btn>
            </template>

            <template v-slot:default="{ isActive }">
              <v-card>
                <v-card-title>
                  上轮得分排名
                </v-card-title>
                <v-divider></v-divider>
                <v-table fixed-header height="300px" density="compact" v-if="rankList.length > 0">
                  <thead>
                  <tr>
                    <th class="text-center">排名</th>
                    <th class="text-center">用户名</th>
                    <th class="text-center">本轮得分</th>
                  </tr>
                  </thead>
                  <tbody >
                  <tr v-for="(item, id) in rankList" :key="id">
                    <td class="text-center">{{ id + 1 }}</td>
                    <td class="text-center">{{ item.username }}</td>
                    <td class="text-center">{{ item.score }}</td>
                  </tr>
                  </tbody>
                </v-table>
                <v-card-text v-else>
                  <v-col class="text-center">暂无数据</v-col>
                </v-card-text>
                <v-divider></v-divider>
                <v-card-actions>
                  <v-spacer></v-spacer>
                  <v-btn
                    color="primary"
                    text="关闭"
                    @click="isActive.value = false"
                  ></v-btn>
                </v-card-actions>
              </v-card>
            </template>
          </v-dialog>
          <v-toolbar-title class="text-h6 text-center">
            Stage {{ gameData.stage }}

            <!--            显示已提交的绿色tag-->
            <v-chip v-if="disabled" color="success" label>已提交</v-chip>
          </v-toolbar-title>

          <template v-slot:append>
            {{ formatTime(gameData.remaining_time) }}
          </template>
        </v-toolbar>
        <!--          背景淡灰色-->
        <v-card-title class="d-flex align-center justify-center text-center flex-grow-1"
                      style="background-color: rgba(0,0,0,0.1);">
          <div class="text-wrap">
            {{ gameData.data }}
          </div>
        </v-card-title>
        <v-card-item>
          <v-row>
            <v-col v-for="(choice, index) in gameData.choice" :key="index" cols="12" md="6">
              <!--              button需要显示所有的文字, 长度过长需要换行-->
              <v-card color="primary" @click="answerQuestion(choice)" :disabled="disabled">
                <v-card-text>
                  {{ choice }}
                </v-card-text>
              </v-card>
            </v-col>
          </v-row>
        </v-card-item>
      </div>
    </v-card>
    <!--    游戏结束的提示-->
    <v-card class="h-100 w-100 d-flex justify-center align-center" v-else>
      <v-card-title>
        游戏结束
      </v-card-title>
    </v-card>
  </v-container>
</template>


<style scoped>

</style>

<route lang="yaml">
meta:
  requiresAuth: true
</route>