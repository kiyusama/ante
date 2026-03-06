<script setup>
import { useRoute } from 'vue-router'
import { ref, computed } from 'vue'
import { useDatabaseObject } from 'vuefire'
import { ref as dbRef, update, set, increment } from 'firebase/database'
import { db } from '@/firebase/config'
import {
  CurrencyDollarIcon,
  CircleStackIcon,
  ArrowRightEndOnRectangleIcon,
  StarIcon,
  ArrowUpCircleIcon,
  CheckCircleIcon,
  TrophyIcon,
  ForwardIcon,
  ArrowUturnLeftIcon,
  UsersIcon,
} from '@heroicons/vue/24/solid'
import { Dialog, DialogPanel, DialogTitle, TransitionRoot, TransitionChild } from '@headlessui/vue'
const route = useRoute()

//headless ui用の表示管理
//bet用
const isBetDialogOpen = ref(false)
const openBetDialog = () => {
  isBetDialogOpen.value = true
}
const closeBetDialog = () => {
  isBetDialogOpen.value = false
}
//push pots用
const isWinnerDialogOpen = ref(false)
const winners = ref({})
const openWinnerDialog = () => {
  winners.value = {} //毎回初期化
  isWinnerDialogOpen.value = true
}
const closeWinnerDialog = () => {
  isWinnerDialogOpen.value = false
}
//player一覧用
const isPlayersDialogOpen = ref(false)
const openPlayersDialog = () => {
  isPlayersDialogOpen.value = true
}
const closePlayersDialog = () => {
  isPlayersDialogOpen.value = false
}

// データベースの参照先を指定
// リアルタイムでデータを取得(room情報もplayer情報も含まれる)
const roomRef = dbRef(db, `rooms/${route.params.roomId}`)
const room = useDatabaseObject(roomRef)

const playerName = ref('')
const isJoined = ref(false)
const betAmount = ref(0)
const rounds = ref(['waiting', 'pre_flop', 'flop', 'turn', 'river', 'show_down'])

//callに必要な額を常に計算
const callAmount = computed(() => {
  return room.value.current_highest_bet - currentPlayer.value.current_bet
})

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
        state: 'active', //active,fold,all_in,dead
        is_dealer: false,
      },
    })
  }

  isJoined.value = true
}

//支払い可能か
const isAbleToPay = (bill) => {
  return bill <= currentPlayer.value.chips
}

// bet/raise action
const bet = async () => {
  //最大ベットならall in
  if (betAmount.value == currentPlayer.value.chips) {
    await allIn()
    closeBetDialog()
    return
  }

  const currentHighestBet = room.value.current_highest_bet
  const newBet = betAmount.value + currentPlayer.value.current_bet

  if (newBet <= currentHighestBet || !isAbleToPay(betAmount.value)) {
    alert("you can't bet")
    closeBetDialog()
    return
  }

  await saveSnap()

  await update(roomRef, {
    pot: increment(betAmount.value),
    current_highest_bet: newBet,
    [`players/${playerName.value}/chips`]: increment(-betAmount.value),
    [`players/${playerName.value}/current_bet`]: increment(betAmount.value),
  })

  closeBetDialog()
}

//call action
const call = async () => {
  const currentHighestBet = room.value.current_highest_bet

  if (!isAbleToPay(callAmount.value)) {
    await allIn()
    return
  }

  if (currentHighestBet == currentPlayer.value.current_bet) {
    alert("you can't call")
    return
  }

  await saveSnap()

  await update(roomRef, {
    pot: increment(callAmount.value),
    [`players/${playerName.value}/current_bet`]: currentHighestBet,
    [`players/${playerName.value}/chips`]: increment(-callAmount.value),
  })
}

const allIn = async () => {
  const allInAmount = currentPlayer.value.chips
  if (allInAmount < 0) {
    alert("you can't all_in")
    return
  }
  await saveSnap()

  const newBetTotal = currentPlayer.value.current_bet + allInAmount
  const currentHighestBet = room.value.current_highest_bet

  const updates = {
    pot: increment(allInAmount),
    [`players/${playerName.value}/chips`]: 0,
    [`players/${playerName.value}/current_bet`]: newBetTotal,
    [`players/${playerName.value}/state`]: 'all_in',
  }

  // オールイン額が最高ベット額を上回っていたら更新
  if (newBetTotal > currentHighestBet) {
    updates.current_highest_bet = newBetTotal
  }

  await update(roomRef, updates)
}

//dealer positionの人が押す
//ゲーム開始の合図
const declareDealer = async () => {
  await saveSnap()

  await update(roomRef, {
    round: 'pre_flop',
    [`players/${playerName.value}/is_dealer`]: true,
  })
}

//ラウンドを進める
//dealer専用
const proceedRound = async () => {
  await saveSnap()

  const currentHighestBet = room.value.current_highest_bet

  const updates = {
    current_highest_bet: 0,
    round: nextRound(),
  }

  const players = room.value.players
  let activePlayerCount = room.value.player_count

  //サイドポットの計算
  updates.pots = calcSidePot(players, room.value.pots)

  Object.keys(room.value.players).forEach((playerNameKey) => {
    const player = room.value.players[playerNameKey]

    //十分な金額をかけていない人を強制フォールド
    //all_inの人は除く
    if (player.state === 'active' && player.current_bet < currentHighestBet) {
      updates[`players/${playerNameKey}/state`] = 'fold'

      activePlayerCount--
    }

    //active以外はactiveCountから減らす
    if (player.state !== 'active') {
      activePlayerCount--
    }
    //すべてのplayerのcurrent_betを初期化
    updates[`players/${playerNameKey}/current_bet`] = 0
  })

  //1人しかactiveが残ってないならshowdownへ
  if (activePlayerCount <= 1) {
    updates.round = 'show_down'
  }

  await update(roomRef, updates)
}

//サイドポットの計算
const calcSidePot = (players, currentPots) => {
  //計算に使うpotのリスト
  const nextPots = currentPots
    ? JSON.parse(JSON.stringify(currentPots))
    : [{ amount: 0, participants: [] }]

  const playerList = Object.entries(players).map(([name, p]) => ({ name, ...p }))

  // 小さいポット額順にソート
  const betLevels = [
    ...new Set(playerList.map((p) => p.current_bet).filter((bet) => bet > 0)),
  ].sort((a, b) => a - b)

  let processedBetAmount = 0

  for (const betLevel of betLevels) {
    const amountToCollect = betLevel - processedBetAmount

    // このポットの参加者
    const payingPlayers = playerList.filter((p) => p.current_bet >= betLevel)
    const activePot = nextPots[nextPots.length - 1]

    // ポット額と参加者を更新
    activePot.amount += amountToCollect * payingPlayers.length
    activePot.participants = payingPlayers.filter((p) => p.state !== 'fold').map((p) => p.name)

    // オールイン発生時に新しいポットを追加
    if (payingPlayers.some((p) => p.current_bet === betLevel && p.state === 'all_in')) {
      nextPots.push({ amount: 0, participants: [] })
    }

    processedBetAmount = betLevel
  }

  return nextPots.filter((pot) => pot.amount > 0)
}

//potsを勝者に渡す
//dealer専用
const pushPots = async () => {
  const pots = room.value.pots

  await saveSnap()

  const updates = {
    pot: 0,
    pots: [],
    current_highest_bet: 0,
    round: 'waiting',
  }

  // 各プレイヤーの賞金
  const playerRewards = {}
  pots.forEach((pot, index) => {
    const winnerName = winners.value[index]
    if (winnerName) {
      playerRewards[winnerName] = (playerRewards[winnerName] || 0) + pot.amount
    }
  })

  Object.keys(room.value.players).forEach((playerNameKey) => {
    const player = room.value.players[playerNameKey]

    const reward = playerRewards[playerNameKey] || 0
    const newChips = player.chips + reward

    updates[`players/${playerNameKey}/chips`] = newChips
    updates[`players/${playerNameKey}/current_bet`] = 0
    updates[`players/${playerNameKey}/is_dealer`] = false

    // chipsが0になったらdead
    if (newChips <= 0) {
      updates[`players/${playerNameKey}/state`] = 'dead'
    } else {
      updates[`players/${playerNameKey}/state`] = 'active'
    }
  })

  await update(roomRef, updates)
  closeWinnerDialog()
}

//状態のスナップショットを撮る
const saveSnap = async () => {
  // history情報とそれ以外に分けて取り出す
  const { history, ...currentState } = room.value

  let newHistory = history || []

  // 履歴の最大数を制限(10個まで)
  newHistory = [...newHistory.slice(-9), currentState]

  await update(roomRef, {
    history: newHistory,
  })
}

//巻き戻し
//dealer専用
const undo = async () => {
  let currentHistory = room.value.history || []

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

const nextRound = () => {
  const currentRound = room.value.round
  let round

  for (let i = 0; i < rounds.value.length; i++) {
    if (rounds.value[i] == currentRound) {
      round = rounds.value[(i + 1) % rounds.value.length]
    }
  }

  return round
}
</script>

<template>
  <div
    class="min-h-screen bg-[#7A3E53] text-[#F6F1D4] font-mono flex flex-col items-center py-8 px-4 relative uppercase tracking-wider"
  >
    <div v-if="isJoined" class="w-full max-w-sm flex flex-col gap-6 mt-4">
      
      <div class="w-full flex justify-between items-center mb-6">
        <div class="w-16 h-16">
          <button
            v-if="currentPlayer?.is_dealer"
            @click="undo"
            class="w-full h-full bg-[#F6F1D4] text-[#2A2A2A] rounded-full flex items-center justify-center shadow-[4px_4px_0_#2A2A2A] active:translate-y-1 active:translate-x-1 active:shadow-none transition-all"
          >
            <ArrowUturnLeftIcon class="w-8 h-8 stroke-2" />
          </button>
        </div>

        <h1 class="text-5xl font-normal tracking-widest text-[#F6F1D4]">ANTE</h1>

        <div class="w-16 h-16">
          <button
            @click="openPlayersDialog"
            class="w-full h-full bg-[#F6F1D4] text-[#2A2A2A] rounded-full flex items-center justify-center shadow-[4px_4px_0_#2A2A2A] active:translate-y-1 active:translate-x-1 active:shadow-none transition-all"
          >
            <UsersIcon class="w-8 h-8 stroke-2" />
          </button>
        </div>
      </div>

      <div class="w-full flex flex-col items-center my-4">
        <div
          class="w-full max-w-[280px] bg-[#2A2A2A] flex flex-col items-center justify-center pt-4 pb-2"
          style="clip-path: polygon(25% 0%, 75% 0%, 100% 100%, 0% 100%)"
        >
          <span class="text-[#F6F1D4] text-sm tracking-widest">POT</span>
          <span class="text-[#F6F1D4] text-4xl font-normal mt-1">{{ room?.pot || 0 }}</span>
        </div>

        <div
          class="w-full max-w-[320px] bg-[#F6F1D4] text-[#2A2A2A] flex items-center justify-center py-4 my-2"
        >
          <span class="text-3xl font-normal tracking-widest">
            {{ room?.round !== 'waiting' ? room.round.replace('_', ' ') : 'WAITING' }}
          </span>
        </div>

        <div
          class="w-full max-w-[280px] bg-[#2A2A2A] flex flex-col items-center justify-center pt-2 pb-4"
          style="clip-path: polygon(0% 0%, 100% 0%, 75% 100%, 25% 100%)"
        >
          <span class="text-[#F6F1D4] text-sm tracking-widest capitalize">My chips</span>
          <span class="text-[#F6F1D4] text-4xl font-normal mt-1">{{
            currentPlayer?.chips || 0
          }}</span>
        </div>
      </div>

      <div v-if="room?.round !== 'waiting'" class="w-full flex gap-4 mt-8">
        <button
          @click="call"
          :disabled="currentPlayer?.state !== 'active'"
          class="flex-1 bg-[#B08B42] text-[#F6F1D4] rounded-[3rem] py-6 flex flex-col items-center justify-center disabled:opacity-50 active:scale-95 transition-transform"
        >
          <span class="text-3xl font-normal tracking-wider">Call</span>
          <span v-if="callAmount > 0" class="text-xl mt-1">+{{ callAmount }}</span>
        </button>

        <button
          @click="openBetDialog"
          :disabled="currentPlayer?.state !== 'active'"
          class="flex-1 bg-[#B08B42] text-[#F6F1D4] rounded-[3rem] py-6 flex flex-col items-center justify-center disabled:opacity-50 active:scale-95 transition-transform"
        >
          <span class="text-3xl font-normal tracking-wider">BET</span>
        </button>
      </div>

      <div v-if="room?.round === 'waiting'" class="w-full mt-8">
        <button
          @click="declareDealer"
          class="w-full bg-[#B08B42] text-[#F6F1D4] rounded-[3rem] py-6 flex items-center justify-center active:scale-95 transition-transform"
        >
          <span class="text-3xl font-normal tracking-wider">Dealer</span>
        </button>
      </div>

      <div v-if="currentPlayer?.is_dealer" class="w-full flex flex-col gap-3 mt-4">
        <button
          @click="room?.round === 'show_down' ? openWinnerDialog() : proceedRound()"
          class="w-full rounded-[3rem] py-4 flex items-center justify-center gap-2 active:scale-95 transition-transform"
          :class="
            room?.round === 'show_down'
              ? 'bg-[#B08B42] text-[#F6F1D4]'
              : 'bg-[#2A2A2A] text-[#F6F1D4]'
          "
        >
          <template v-if="room?.round === 'show_down'">
            <span class="text-2xl font-normal tracking-wider">Give Pot</span>
          </template>
          <template v-else>
            <span class="text-2xl font-normal tracking-wider">Next Round</span>
          </template>
        </button>
      </div>
    </div>

    <div
      v-else
      class="w-full max-w-sm m-auto bg-[#2A2A2A] rounded-3xl shadow-2xl p-8 border border-[#F6F1D4]/10 mt-20"
    >
      <div class="text-center mb-8">
        <h1 class="text-4xl font-normal tracking-widest text-[#F6F1D4] mb-2">Join Table</h1>
      </div>

      <div class="space-y-6">
        <div>
          <label class="block text-sm font-normal text-[#F6F1D4]/70 mb-2 tracking-widest"
            >Player Name</label
          >
          <input
            v-model="playerName"
            @keyup.enter="joinGame"
            class="w-full bg-[#7A3E53]/50 border border-[#F6F1D4]/30 rounded-xl px-4 py-4 text-[#F6F1D4] text-xl focus:outline-none focus:border-[#B08B42] transition-all placeholder-[#F6F1D4]/30"
            placeholder="Enter Name"
          />
        </div>
        <button
          @click="joinGame"
          :disabled="!playerName"
          class="w-full bg-[#B08B42] hover:bg-[#A07A30] disabled:opacity-50 disabled:cursor-not-allowed transition-colors text-[#F6F1D4] py-4 px-4 rounded-[2rem] active:scale-95 flex items-center justify-center"
        >
          <span class="text-2xl tracking-widest">Join</span>
        </button>
      </div>
    </div>

    <TransitionRoot appear :show="isWinnerDialogOpen" as="template">
      <Dialog
        as="div"
        @close="closeWinnerDialog"
        class="relative z-50 font-mono tracking-wider uppercase"
      >
        <TransitionChild as="template" enter="duration-300 ease-out" enter-from="opacity-0" enter-to="opacity-100" leave="duration-200 ease-in" leave-from="opacity-100" leave-to="opacity-0">
          <div class="fixed inset-0 bg-black/80 backdrop-blur-sm" />
        </TransitionChild>

        <div class="fixed inset-0 overflow-y-auto">
          <div class="flex min-h-full items-center justify-center p-4 text-center">
            <TransitionChild as="template" enter="duration-300 ease-out" enter-from="opacity-0 scale-95" enter-to="opacity-100 scale-100" leave="duration-200 ease-in" leave-from="opacity-100 scale-100" leave-to="opacity-0 scale-95">
              <DialogPanel
                class="w-full max-w-sm transform overflow-hidden rounded-[2rem] bg-[#2A2A2A] p-6 text-left align-middle shadow-2xl transition-all border border-[#F6F1D4]/10"
              >
                <DialogTitle as="h3" class="text-2xl font-normal leading-6 text-[#F6F1D4] mb-6 text-center">
                  Select Winners
                </DialogTitle>

                <div v-if="!room?.pots || room.pots.length === 0" class="text-[#F6F1D4]/50 text-sm mb-4 text-center">
                  No pots available.
                </div>

                <div v-else class="space-y-5 max-h-[60vh] overflow-y-auto pr-2 custom-scrollbar">
                  <div
                    v-for="(pot, index) in room.pots"
                    :key="index"
                    class="bg-[#7A3E53]/30 p-4 rounded-2xl border border-[#F6F1D4]/10"
                  >
                    <div class="flex justify-between items-end mb-4">
                      <span class="text-sm text-[#F6F1D4]/70">{{ index === 0 ? 'Main Pot' : `Side Pot ${index}` }}</span>
                      <span class="text-2xl text-[#B08B42]">{{ pot.amount }}</span>
                    </div>

                    <div class="space-y-2">
                      <button
                        v-for="playerName in pot.participants"
                        :key="playerName"
                        @click="winners[index] = playerName"
                        class="w-full flex items-center justify-between p-3 rounded-xl border transition-all active:scale-95"
                        :class="winners[index] === playerName ? 'bg-[#B08B42] border-[#B08B42] text-[#F6F1D4]' : 'bg-transparent border-[#F6F1D4]/20 text-[#F6F1D4]/70'"
                      >
                        <span class="text-lg">{{ playerName }}</span>
                        <span v-if="winners[index] === playerName">★</span>
                      </button>
                    </div>
                  </div>
                </div>

                <div class="mt-8 flex gap-3">
                  <button @click="closeWinnerDialog" class="flex-1 bg-transparent border border-[#F6F1D4]/30 text-[#F6F1D4] py-3 rounded-2xl active:scale-95 transition-transform">
                    Cancel
                  </button>
                  <button @click="pushPots" :disabled="!room?.pots || Object.keys(winners).length !== room.pots.length" class="flex-1 bg-[#B08B42] text-[#F6F1D4] py-3 rounded-2xl disabled:opacity-50 active:scale-95 transition-transform">
                    Confirm
                  </button>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </Dialog>
    </TransitionRoot>

    <TransitionRoot appear :show="isBetDialogOpen" as="template">
      <Dialog as="div" @close="closeBetDialog" class="relative z-50 font-mono tracking-wider uppercase">
        <TransitionChild as="template" enter="duration-300 ease-out" enter-from="opacity-0" enter-to="opacity-100" leave="duration-200 ease-in" leave-from="opacity-100" leave-to="opacity-0">
          <div class="fixed inset-0 bg-black/80 backdrop-blur-sm" />
        </TransitionChild>

        <div class="fixed inset-0 overflow-y-auto">
          <div class="flex min-h-full items-center justify-center p-4 text-center">
            <TransitionChild as="template" enter="duration-300 ease-out" enter-from="opacity-0 scale-95" enter-to="opacity-100 scale-100" leave="duration-200 ease-in" leave-from="opacity-100 scale-100" leave-to="opacity-0 scale-95">
              <DialogPanel class="w-full max-w-sm transform overflow-hidden rounded-[2rem] bg-[#2A2A2A] p-8 text-center align-middle shadow-2xl transition-all border border-[#F6F1D4]/10">
                <DialogTitle as="h3" class="text-3xl font-normal leading-6 text-[#F6F1D4] mb-8">
                  Bet Amount
                </DialogTitle>

                <div class="space-y-4">
                  <input type="number" v-model.number="betAmount" :min="room?.current_highest_bet" :max="currentPlayer?.chips" @keyup.enter="bet" class="w-full bg-[#7A3E53]/30 border border-[#F6F1D4]/30 rounded-2xl px-4 py-4 text-4xl text-[#F6F1D4] text-center focus:outline-none focus:border-[#B08B42] transition-all" :placeholder="room?.current_highest_bet" />
                  <p class="text-sm text-[#F6F1D4]/70">Max: <span class="text-[#B08B42]">{{ currentPlayer?.chips }}</span></p>
                </div>

                <div class="mt-8 flex gap-3">
                  <button @click="closeBetDialog" class="flex-1 bg-transparent border border-[#F6F1D4]/30 text-[#F6F1D4] py-3 rounded-2xl active:scale-95 transition-transform">Cancel</button>
                  <button @click="bet" class="flex-1 bg-[#B08B42] text-[#F6F1D4] py-3 rounded-2xl active:scale-95 transition-transform">Bet</button>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </Dialog>
    </TransitionRoot>

    <TransitionRoot appear :show="isPlayersDialogOpen" as="template">
      <Dialog as="div" @close="closePlayersDialog" class="relative z-50 font-mono tracking-wider uppercase">
        <TransitionChild as="template" enter="duration-300 ease-out" enter-from="opacity-0" enter-to="opacity-100" leave="duration-200 ease-in" leave-from="opacity-100" leave-to="opacity-0">
          <div class="fixed inset-0 bg-black/80 backdrop-blur-sm" />
        </TransitionChild>

        <div class="fixed inset-0 overflow-y-auto">
          <div class="flex min-h-full items-center justify-center p-4 text-center">
            <TransitionChild as="template" enter="duration-300 ease-out" enter-from="opacity-0 scale-95" enter-to="opacity-100 scale-100" leave="duration-200 ease-in" leave-from="opacity-100 scale-100" leave-to="opacity-0 scale-95">
              <DialogPanel class="w-full max-w-sm transform overflow-hidden rounded-[2rem] bg-[#2A2A2A] p-6 text-left align-middle shadow-2xl transition-all border border-[#F6F1D4]/10">
                <DialogTitle as="h3" class="text-2xl font-normal leading-6 text-[#F6F1D4] mb-6 flex justify-between items-center">
                  <span>Players</span>
                  <span class="text-sm text-[#B08B42]">Total {{ room?.player_count || 0 }}</span>
                </DialogTitle>

                <div class="space-y-3 max-h-[60vh] overflow-y-auto pr-2">
                  <div v-for="(player, pName) in room.players" :key="pName" class="bg-[#7A3E53]/30 p-4 rounded-xl border border-[#F6F1D4]/10 flex items-center justify-between">
                    <div class="flex flex-col gap-1">
                      <div class="flex items-center gap-2">
                        <span class="text-lg text-[#F6F1D4]">{{ pName }}</span>
                        <span v-if="player.is_dealer" class="text-[#B08B42] text-sm">★</span>
                      </div>
                      <div>
                        <span class="text-xs px-2 py-0.5 rounded-full border border-[#F6F1D4]/30" :class="{ 'text-[#B08B42] border-[#B08B42]': player.state === 'active', 'text-red-400 border-red-400/50': player.state === 'all_in', 'text-[#F6F1D4]/50': player.state === 'fold' || player.state === 'dead' }">
                          {{ player.state }}
                        </span>
                      </div>
                    </div>

                    <div class="text-right flex flex-col items-end">
                      <div class="text-[#F6F1D4] text-xl">{{ player.chips }}</div>
                      <div v-if="player.current_bet > 0" class="text-xs text-[#B08B42] mt-1">Bet: {{ player.current_bet }}</div>
                    </div>
                  </div>
                </div>

                <div class="mt-8">
                  <button @click="closePlayersDialog" class="w-full bg-transparent border border-[#F6F1D4]/30 text-[#F6F1D4] py-3 rounded-2xl active:scale-95 transition-transform">Close</button>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </Dialog>
    </TransitionRoot>
  </div>
</template>