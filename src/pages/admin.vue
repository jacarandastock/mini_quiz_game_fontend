<script setup lang='ts'>
import api from '@/utils/api'


type Record = {
  username: string,
  question: string,
  score: number,
  stage: number,
  timestamp: number
}

type Users = {
  username: string,
  password: string,
  isAdmin: string
}

const record = ref<Array<Record>>([])

const users = ref<Array<Users>>([])
const search = ref('')
const usersHeaders = [
  {
    title: 'Username',
    key: 'username',
    align: 'start'
  },
  { title: 'Password', key: 'password' },
  { title: 'isAdmin', key: 'isAdmin' }
]

const clearRecordDialog = ref(false)

function filterUsername(value: string, search: string) {
  //
  return value.toString().toLowerCase().indexOf(search) !== -1
}


async function startGame() {
  console.log('start game')
  const response = await api.post('/admin/game/start')
  alert(response.data.message)
}

async function resetGame() {
  const c = confirm('确定要重置游戏吗？')
  if (c) {
    console.log('reset game')
    const response = await api.post('/admin/game/reset')
    alert(response.data.message)
  }
}

async function get_all_records() {
  console.log('get all records')
  const response = await api.post('/admin/game/record')
  console.log(response)
  record.value = response.data.data
}

async function get_all_user() {
  console.log('get all user')
  const response = await api.post('/admin/users')
  console.log(response)
  users.value = response.data.data
}

async function clearRecord() {
  console.log('clear record')
  const response = await api.post('/admin/game/record_clear')
  clearRecordDialog.value = false
  console.log(response)
}


onMounted(() => {
  get_all_records()
  get_all_user()
  // 自动更新record
  setInterval(() => {
    get_all_records()
    console.log('update records')
  }, 10000)
})

function name2RgbColor(name: string) {
  let hash = 0
  for (let i = 0; i < name.length; i++) {
    hash = name.charCodeAt(i) + ((hash << 5) - hash)
  }
  const c = (hash & 0x00FFFFFF)
    .toString(16)
    .toUpperCase()
  return '#' + '00000'.substring(0, 6 - c.length) + c
}

// 计算每个用户的得分, 从record中计算, 返回score从高到低排序
function calculateScore(record: any) {
  if (record === undefined || record.length === 0) {
    return []
  }
  let score: any = {}
  for (let i = 0; i < record.length; i++) {
    if (score[record[i].username] === undefined) {
      score[record[i].username] = 0
    }
    score[record[i].username] += record[i].score
  }
  let sortable = []
  for (let user in score) {
    sortable.push([user, score[user]])
  }
  sortable.sort((a, b) => {
    return b[1] - a[1]
  })
  return sortable
}

</script>

<template>
  <v-container class="fill-height">
    <v-row>
      <v-col cols="12">
        <v-btn @click="startGame" class="mr-2">Game Start</v-btn>
        <v-btn @click="resetGame" color="red-accent-4">Reset Game</v-btn>
      </v-col>
      <v-col cols="12">
        <!--        计算每个用户的得分-->
        <!--        显示用户排名, 从高到低-->
        <v-card>
          <v-card-title>
            <h3>Rank</h3>
          </v-card-title>
          <v-table>
            <thead>
            <tr>
              <th class="text-left">
                Rank
              </th>
              <th class="text-left">
                Username
              </th>
              <th class="text-left">
                Score
              </th>
            </tr>
            </thead>
            <tbody>
            <tr
              v-for="(item, id) in calculateScore(record)"
              :key="id"
            >
              <td>
                {{ id + 1 }}
              </td>
              <td>{{ item[0] }}</td>
              <td>{{ item[1] }}</td>
            </tr>
            </tbody>
          </v-table>
        </v-card>
      </v-col>
      <v-col cols="12">
        <v-card>
          <v-card-title>
            <h3>Records</h3>
          </v-card-title>
          <v-toolbar>
            <v-btn icon @click="clearRecordDialog = true">
              <v-icon>mdi-delete</v-icon>
            </v-btn>
            <v-spacer></v-spacer>
            <v-btn icon @click="get_all_records">
              <v-icon>mdi-refresh</v-icon>
            </v-btn>
          </v-toolbar>
          <v-table>
            <thead>
            <tr>
              <th class="text-left">
                id
              </th>
              <th class="text-left">
                User
              </th>
              <th class="text-left">
                Question
              </th>
              <th class="text-left">
                Answer
              </th>
              <th class="text-left">
                Score
              </th>
              <th class="text-left">
                Answer Stage
              </th>
              <th class="text-left">
                Time
              </th>
            </tr>
            </thead>
            <tbody>
            <tr
              v-for="(item, id) in record"
              :key="id"
            >
              <td>{{ id }}</td>
              <td>
                <mark :style="'background-color:' + name2RgbColor(item.username) ">{{ item.username }}</mark>
              </td>
              <td>{{ item.question }}</td>
              <td>{{ item.answer }}</td>
              <td>{{ item.score }}</td>
              <td>{{ item.stage }}</td>
              <td>{{ new Date(item.timestamp * 1000).toLocaleString() }}</td>
            </tr>
            </tbody>
          </v-table>
          <v-expansion-panels>
            <v-expansion-panel>
              <v-expansion-panel-title expand-icon="mdi-menu-down">

              </v-expansion-panel-title>
              <v-expansion-panel-text>
                {{ record }}
              </v-expansion-panel-text>
            </v-expansion-panel>
          </v-expansion-panels>
        </v-card>
      </v-col>
      <v-col cols="12">
        <v-expansion-panels class="mb-6">
          <v-expansion-panel>
            <v-expansion-panel-title expand-icon="mdi-menu-down">
              用户管理
            </v-expansion-panel-title>
            <v-expansion-panel-text>
              <!--        <v-expansion-panels>-->
              <!--          <v-expansion-panel-text title="Title">-->
              <v-card elevation="0">
                <v-toolbar>
                  <v-spacer></v-spacer>
                  <v-btn icon @click="get_all_user">
                    <v-icon>mdi-refresh</v-icon>
                  </v-btn>
                </v-toolbar>
                <v-data-table
                  :custom-filter="filterUsername"
                  :headers="usersHeaders"
                  :items="users"
                  :search="search"
                  item-value="name"
                >
                  <template v-slot:top>
                    <v-text-field
                      v-model="search"
                      class="pa-2"
                      label="Search User"
                    ></v-text-field>
                  </template>
                </v-data-table>
              </v-card>
              <!--          </v-expansion-panel-text>-->
              <!--        </v-expansion-panels>-->
            </v-expansion-panel-text>
          </v-expansion-panel>
        </v-expansion-panels>
      </v-col>
    </v-row>
    <v-dialog
      v-model="clearRecordDialog"
      width="auto"
    >
      <v-card
        max-width="400"
        prepend-icon="mdi-alert"
        text="你确定要清空所有记录吗？"
        title="警告！"
      >
        <template v-slot:actions>
          <v-btn
            class="ms-auto"
            text="取消"
            type="primary"
            @click="clearRecordDialog = false"
          ></v-btn>
          <v-btn
            class="ms-2"
            text="确定"
            color="error"
            @click="clearRecord"
          ></v-btn>
        </template>
      </v-card>
    </v-dialog>
  </v-container>
</template>


<style scoped>

</style>

<route lang="yaml">
meta:
  requiresAuth: true
  isAdmin: true
</route>