<script setup>
import { useRoute } from 'vue-router'
import { computed, ref } from 'vue'
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
  const currentPlayersCount = room.value.player_count

  const player = room.value.players?.[playerName.value]

  //playerが存在しなかったら
  if (!player) {
    await update(roomRef, {
      player_count: increment(1),
      [`players/${name}`]: {
        name: name,
        chips: stack,
        current_bet: 0,
        seat_index: currentPlayersCount,
        state: 'active', //active, folded, dead
      },
    })
  }

  isJoined.value = true
}

//支払い可能か
const isAbleToPay = (bill) => {
  if (bill > room.value.players[playerName.value].chips) {
    return false
  }
  return true
}

//自分のターンか
const isMyTurn = computed(() => {
  if (room.value.current_turn_id == room.value.players[playerName.value].seat_index) {
    return true
  }
  return false
})

//次のターンIDを計算
const calcNextTurnId = () => {
  const totalPlayers = room.value.player_count
  let nextTurnId = (room.value.players[playerName.value].seat_index + 1) % totalPlayers
  //playersを配列に落とし込む
  const players = Object.values(room.value.players)

  while (true) {
    //隣の人を探す
    const nextPlayer = players.find((p) => p.seat_index === nextTurnId)
    if (nextPlayer.state == 'active') {
      break // アクティブなプレイヤーを見つけたら終わり
    }
    nextTurnId = (nextTurnId + 1) % totalPlayers
  }
  return nextTurnId
}

// bet/raise action
const bet = async (betAmount) => {
  const currentHighestBet = room.value.current_highest_bet

  if (betAmount <= currentHighestBet || !isAbleToPay(betAmount)) {
    alert("you can't bet")
    return
  }

  await update(roomRef, {
    pot: increment(betAmount),
    current_turn_id: increment(1),
    last_aggressor: room.value.players[playerName.value].seat_index,
    current_highest_bet: betAmount,
    [`players/${playerName.value}/current_bet`]: betAmount,
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

//fold action
const fold = async () => {
  await update(roomRef, {
    current_turn_id: increment(1),
    [`players/${playerName.value}/state`]: 'folded',
  })
}
</script>

<template>
  <div v-if="isJoined">
    <h2>pot: {{ room.pot }} 枚</h2>
    <h2>chips: {{ room.players[playerName].chips }} 枚</h2>

    <div v-if="isMyTurn">
      <button @click="bet(20)">initial bet</button>

      <button @click="bet(Math.floor(room.pot / 2))">bet</button>
      <button @click="call">call</button>
      <button @click="check">check</button>
      <button @click="fold">fold</button>
    </div>
  </div>

  <div v-else>
    <input v-model="playerName" />
    <button @click="joinGame">参加</button>
  </div>
</template>
