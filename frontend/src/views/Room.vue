<script setup>
import { useRoute } from 'vue-router'
import { ref, computed } from 'vue'
import { useDatabaseObject } from 'vuefire'
import { ref as dbRef, update, set, increment } from 'firebase/database'
import { db } from '@/firebase/config'
const route = useRoute()

// データベースの参照先を指定
// リアルタイムでデータを取得(room情報もplayer情報も含まれる)
const roomRef = dbRef(db, `rooms/${route.params.roomId}`)
const room = useDatabaseObject(roomRef)

const playerName = ref('')
const isJoined = ref(false)

const betAmount = ref(0)

//自分の情報を取得
const currentPlayer = computed(() => {
  return room.value?.players?.[playerName.value]
})

// ゲームへの参加
const joinGame = async () => {
  const name = playerName.value
  const stack = room.value.stack

  //playerが存在しなかったら
  if (!currentPlayer.value) {
    await update(roomRef, {
      player_count: increment(1),
      [`players/${name}`]: {
        name: name,
        chips: stack,
        current_bet: 0,
        state: 'active', //active,folded,all_in,dead
        is_dealer: false,
      },
    })
  }

  isJoined.value = true
}

const saveSnap = async () => {
  // history情報とそれ以外に分けて取り出す
  const { history, ...currentState } = room.value

  let newHistory = history || []

  // 履歴の最大数を制限
  newHistory = [...newHistory.slice(-4), currentState]

  await update(roomRef, {
    history: newHistory,
  })
}

const undo = async () => {
  let currentHistory = room.value.history

  if (currentHistory.length === 0) {
    alert('no histories')
    return
  }
  const previousState = currentHistory.pop()

  await set(roomRef, {
    ...previousState,
    history: currentHistory,
  })
}

//dealer positionの人が押す
//ゲーム開始の合図
const declareDealer = async () => {
  await saveSnap()

  await update(roomRef, {
    waiting: false,
    [`players/${playerName.value}/is_dealer`]: true,
  })
}

//支払い可能か
const isAbleToPay = (bill) => {
  return bill <= currentPlayer.value.chips
}

// bet/raise action
const bet = async (betAmount) => {
  const currentHighestBet = room.value.current_highest_bet
  const newBet = betAmount + currentPlayer.value.current_bet

  if (newBet <= currentHighestBet || !isAbleToPay(betAmount)) {
    alert("you can't bet")
    return
  }

  await saveSnap()

  await update(roomRef, {
    pot: increment(betAmount),
    current_highest_bet: newBet,
    [`players/${playerName.value}/chips`]: increment(-betAmount),
    [`players/${playerName.value}/current_bet`]: increment(betAmount),
  })
}

//call action
const call = async () => {
  const currentHighestBet = room.value.current_highest_bet
  const callAmount = currentHighestBet - currentPlayer.value.current_bet

  if (!isAbleToPay(callAmount)) {
    alert("you can't call")
    return
  }

  await saveSnap()

  await update(roomRef, {
    pot: increment(callAmount),
    [`players/${playerName.value}/current_bet`]: currentHighestBet,
    [`players/${playerName.value}/chips`]: increment(-callAmount),
  })
}

//ラウンドを進める
//dealer positionのみ可能
const proceedRound = async () => {
  await saveSnap()

  const currentHighestBet = room.value.current_highest_bet

  const updates = {
    current_highest_bet: 0,
  }

  Object.keys(room.value.players).forEach((playerNameKey, player) => {
    //十分な金額をかけていない人を強制フォールド
    if (player.state === 'active' && player.current_bet < currentHighestBet) {
      updates[`players/${playerNameKey}/state`] = 'folded'
    }

    //すべてのplayerのcurrent_betを初期化
    updates[`players/${playerNameKey}/current_bet`] = 0
  })
  await update(roomRef, updates)
}

//potの獲得で1ハンド終了
const take = async () => {
  await saveSnap()

  const potValue = room.value.pot
  const updates = {
    pot: 0,
    current_highest_bet: 0,
    waiting: true,
    [`players/${playerName.value}/chips`]: increment(potValue),
  }
  //すべてのplayerのcurrent_betを初期化
  Object.keys(room.value.players).forEach((playerNameKey) => {
    updates[`players/${playerNameKey}/current_bet`] = 0
    updates[`players/${playerNameKey}/state`] = 'active'
    updates[`players/${playerNameKey}/is_dealer`] = false
  })
  await update(roomRef, updates)
}
</script>

<template>
  <div v-if="isJoined">
    <div>
      <button @click="undo">undo</button>
    </div>

    <h2>pot: {{ room.pot }} 枚</h2>
    <h2>chips: {{ room.players[playerName].chips }} 枚</h2>

    <div v-if="room.waiting">
      <button @click="declareDealer">dealer</button>
    </div>
    <div v-else>
      <div>
        <input
          type="number"
          v-model.number="betAmount"
          :min="room.currentHighestBet"
          :max="currentPlayer?.chips"
        />
        <button @click="bet(betAmount)">bet</button>
      </div>

      <div>
        <button @click="call">call</button>
      </div>

      <div>
        <button @click="take">take</button>
      </div>
    </div>

    <div v-if="currentPlayer.is_dealer">
      <button @click="proceedRound">next round</button>
    </div>
  </div>

  <div v-else>
    <input v-model="playerName" />
    <button @click="joinGame">参加</button>
  </div>
</template>
