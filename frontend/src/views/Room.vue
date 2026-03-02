<script setup>
import { useRoute } from 'vue-router'
import { ref } from 'vue'
import { useDatabaseObject } from 'vuefire'
import { ref as dbRef, update, increment } from 'firebase/database'
import { db } from '@/firebase/config'
const route = useRoute()

// データベースの参照先を指定
// リアルタイムでデータを取得(room情報もplayer情報も含まれる)
const roomRef = dbRef(db, `rooms/${route.params.roomId}`)
const room = useDatabaseObject(roomRef)

const playerName = ref('')
const isJoined = ref(false)

// ゲームへの参加
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
        current_bet: 0,
        seat_index: 0,
        state: 0,
      },
    })
  }

  isJoined.value = true
}

//支払い可能か
const isAbleToPay = (bill) => {
  if (bill > room.value.players[playerName.value]) {
    return false
  }
  return true
}

// bet/raise action
const bet = async (betAmount) => {
  const currentHighestBet = room.value.current_highest_bet
  const totalBet = betAmount + room.value.players[playerName.value].current_bet

  if (totalBet <= currentHighestBet || !isAbleToPay(betAmount)) {
    alert("you can't bet")
    return
  }

  await update(roomRef, {
    pot: increment(betAmount),
    current_turn_id: increment(1),
    current_highest_bet: totalBet,
    [`players/${playerName.value}/chips`]: increment(-betAmount),
    [`players/${playerName.value}/current_bet`]: totalBet,
  })
}

//call action
const call = async () => {
  const currentHighestBet = room.value.current_highest_bet
  const callAmount = currentHighestBet - room.value.players[playerName.value].current_bet

  if (!isAbleToPay(callAmount)) {
    alert("you can't call")
    return
  }

  await update(roomRef, {
    pot: increment(callAmount),
    current_turn_id: increment(1),
    [`players/${playerName.value}/chips`]: increment(-callAmount),
    [`players/${playerName.value}/current_bet`]: currentHighestBet,
  })
}

//check action
const check = async () => {
  if (room.value.players[playerName.value].current_bet < room.value.current_highest_bet) {
    alert("you can't check")
    return
  }

  await update(roomRef, {
    current_turn_id: increment(1),
  })
}
</script>

<template>
  <div v-if="isJoined">
    <h2>pot: {{ room.pot }} 枚</h2>
    <h2>chips: {{ room.players[playerName].chips }} 枚</h2>
    <button @click="bet(Math.floor(room.pot / 2))">bet</button>
    <button @click="call">call</button>
    <button @click="check">check</button>
  </div>

  <div v-else>
    <input v-model="playerName" />
    <button @click="joinGame">参加</button>
  </div>
</template>
