<script setup>
import { useRoute } from 'vue-router'
const route = useRoute()
import { useDatabaseObject } from 'vuefire'
import { ref as dbRef, update, increment, set, get } from 'firebase/database'
import { db } from '@/firebase/config'

// データベースの参照先を指定
const roomRef = dbRef(db, `rooms/${route.params.roomId}`)
// リアルタイムでデータを取得(roomもplayerも含まれる)
const room = useDatabaseObject(roomRef)

const joinGame = async () => {
  const name = 'test'

  //room情報を取得
  const room = await get(roomRef)
  const stack = room.val().stack

  //player情報の確認
  const playerRef = dbRef(db, `rooms/${route.params.roomId}/players/${name}`)
  const player = await get(playerRef)

  //playerが存在しなかったら
  if (!player.exists()) {
    await set(playerRef, {
      name: name,
      chips: stack,
      currentBet: 0,
      seat_index: 0,
      state: 0,
    })
  } else {
  }
}

// ボタンを押した時の処理
async function addChips() {
  // potの数値を10増やす
  await update(roomRef, {
    pot: increment(10),
  })
}
</script>

<template>
  <button @click="joinGame">参加</button>

  <h2>現在のポット: {{ room.pot }} 枚</h2>
  <button @click="addChips">ベット</button>
</template>
