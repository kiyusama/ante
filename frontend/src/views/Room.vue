<script setup>
import { useRoute } from 'vue-router'
import { ref } from 'vue'
import { useDatabaseObject } from 'vuefire'
import { ref as dbRef, update, increment } from 'firebase/database'
import { db } from '@/firebase/config'
const route = useRoute()

// データベースの参照先を指定
// リアルタイムでデータを取得(roomもplayerも含まれる)
const roomRef = dbRef(db, `rooms/${route.params.roomId}`)
const room = useDatabaseObject(roomRef)

const playerName = ref('')
const isJoined = ref(false)

const joinGame = async () => {
  const name = playerName.value
  const stack = room.value.stack

  const player = room.value.players?.[playerName.value]

  //playerが存在しなかったら
  if (!player) {
    await update(roomRef, {
      [`players/${name}`]: {
        name: name,
        chips: stack,
        currentBet: 0,
        seat_index: 0,
        state: 0,
      },
    })
  }

  isJoined.value = true
}

// ボタンを押した時の処理
async function addChips() {
  // potの数値を10増やす
  await update(roomRef, {
    pot: increment(10),
  })

  await update(roomRef, {
    [`players/${playerName.value}/chips`]: increment(-10),
  })
}
</script>

<template>
  <div v-if="isJoined">
    <h2>pot: {{ room.pot }} 枚</h2>
    <h2>chips: {{ room.players[playerName].chips }} 枚</h2>
    <button @click="addChips">ベット</button>
  </div>

  <div v-else>
    <input v-model="playerName" />
    <button @click="joinGame">参加</button>
  </div>
</template>
