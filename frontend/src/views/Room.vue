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
  //サイドポットの計算
  updates.pots = calcSidePot(players, room.value.pots)

  Object.keys(room.value.players).forEach((playerNameKey) => {
    const player = room.value.players[playerNameKey]

    //十分な金額をかけていない人を強制フォールド
    //all_inの人は除く
    if (player.state === 'active' && player.current_bet < currentHighestBet) {
      updates[`players/${playerNameKey}/state`] = 'fold'
    }

    //すべてのplayerのcurrent_betを初期化
    updates[`players/${playerNameKey}/current_bet`] = 0
  })
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
    class="min-h-screen bg-slate-900 text-slate-100 font-sans flex flex-col items-center py-8 px-4 relative"
  >
    <div v-if="isJoined" class="w-full max-w-md flex flex-col gap-6">
      <div class="bg-slate-800 rounded-2xl shadow-xl p-6 border border-slate-700">
        <div class="flex items-center justify-between mb-4">
          <span class="text-slate-400 font-medium tracking-wider text-sm uppercase"
            >Room status</span
          >
          <span
            v-if="room.round !== 'waiting'"
            class="bg-indigo-500/20 text-indigo-300 px-3 py-1 rounded-full text-xs font-bold uppercase border border-indigo-500/30 ml-3"
          >
            {{ room.round }}
          </span>
          <span
            v-if="currentPlayer?.is_dealer"
            class="bg-blue-500/20 text-blue-400 px-3 py-1 rounded-full text-xs font-bold flex items-center gap-1"
          >
            <StarIcon class="w-4 h-4" /> Dealer
          </span>
        </div>

        <div class="space-y-3">
          <div
            class="flex justify-between items-center bg-slate-900/50 p-4 rounded-xl border border-slate-700/50"
          >
            <div class="flex items-center gap-2 text-slate-300">
              <CurrencyDollarIcon class="w-6 h-6 text-yellow-500" />
              <span class="font-semibold">Pot</span>
            </div>
            <span class="text-3xl font-black text-yellow-400">{{ room?.pot || 0 }}</span>
          </div>
          <div
            class="flex justify-between items-center bg-slate-900/50 p-4 rounded-xl border border-slate-700/50"
          >
            <div class="flex items-center gap-2 text-slate-300">
              <CircleStackIcon class="w-6 h-6 text-emerald-500" />
              <span class="font-semibold">My Chips</span>
            </div>
            <span class="text-3xl font-black text-emerald-400">{{
              currentPlayer?.chips || 0
            }}</span>
          </div>
        </div>

        <div class="mt-4 pt-4 border-t border-slate-700/50">
          <button
            @click="openPlayersDialog"
            class="w-full flex items-center justify-center gap-2 bg-slate-700 hover:bg-slate-600 transition-colors border border-slate-600 text-white font-bold py-3 px-4 rounded-xl active:scale-95"
          >
            <UsersIcon class="w-5 h-5 text-slate-300" />
            <span>Players List</span>
          </button>
        </div>
      </div>

      <div class="bg-slate-800 rounded-2xl shadow-xl p-6 border border-slate-700">
        <div
          v-if="room?.round === 'waiting'"
          class="flex flex-col items-center justify-center py-4"
        >
          <button
            @click="declareDealer"
            class="w-full flex items-center justify-center gap-2 bg-blue-600 hover:bg-blue-500 transition-colors text-white font-bold py-4 px-6 rounded-xl shadow-lg shadow-blue-900/20 active:scale-95"
          >
            <StarIcon class="w-6 h-6" /> Dealer
          </button>
        </div>

        <div v-else class="space-y-5">
          <div class="flex gap-3 pt-2">
            <button
              @click="call"
              :disabled="currentPlayer?.state !== 'active'"
              class="flex-1 flex flex-col items-center justify-center gap-1 bg-slate-700 hover:bg-slate-600 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:bg-slate-700 disabled:active:scale-100 transition-colors border border-slate-600 text-white font-bold py-3 px-4 rounded-xl active:scale-95"
            >
              <CheckCircleIcon
                class="w-6 h-6"
                :class="currentPlayer?.state !== 'active' ? 'text-slate-400' : 'text-emerald-400'"
              />
              <span>Call {{ callAmount > 0 ? '+' + callAmount : '' }}</span>
            </button>

            <button
              @click="openBetDialog"
              :disabled="currentPlayer?.state !== 'active'"
              class="flex-1 flex flex-col items-center justify-center gap-1 bg-indigo-600 hover:bg-indigo-500 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:bg-indigo-600 disabled:active:scale-100 transition-colors border border-indigo-500 text-white font-bold py-3 px-4 rounded-xl active:scale-95 shadow-lg shadow-indigo-900/20 disabled:shadow-none"
            >
              <ArrowUpCircleIcon
                class="w-6 h-6"
                :class="currentPlayer?.state !== 'active' ? 'text-slate-400' : 'text-indigo-300'"
              />
              <span>Bet / Raise</span>
            </button>
          </div>
        </div>
      </div>

      <div
        v-if="currentPlayer?.is_dealer"
        class="bg-blue-900/20 border border-blue-500/30 rounded-2xl p-6"
      >
        <h3
          class="text-blue-400 font-bold text-sm uppercase tracking-wider mb-4 flex items-center gap-2"
        >
          <StarIcon class="w-4 h-4" /> Dealer Controls
        </h3>
        <div class="flex gap-3">
          <button
            @click="undo"
            class="flex items-center justify-center gap-2 bg-red-500/10 hover:bg-red-500/20 border border-red-500/50 text-red-400 transition-colors font-semibold py-3 px-6 rounded-xl active:scale-95"
          >
            <ArrowUturnLeftIcon class="w-5 h-5" /> Undo
          </button>
          <button
            @click="proceedRound"
            class="flex-1 flex items-center justify-center gap-2 bg-blue-600/80 hover:bg-blue-500 transition-colors text-white font-semibold py-3 px-4 rounded-xl active:scale-95"
          >
            <ForwardIcon class="w-5 h-5" /> Continue
          </button>
          <button
            @click="openWinnerDialog"
            class="flex-1 flex flex-col items-center justify-center gap-1 bg-gradient-to-br from-yellow-400 to-amber-600 hover:to-amber-500 transition-colors text-slate-900 font-black py-3 px-4 rounded-xl shadow-lg shadow-amber-900/20 active:scale-95"
          >
            <TrophyIcon class="w-6 h-6" />
            <span>Give Pot</span>
          </button>
        </div>
      </div>
    </div>

    <div
      v-else
      class="w-full max-w-sm m-auto bg-slate-800 rounded-2xl shadow-2xl p-8 border border-slate-700 mt-20"
    >
      <div class="text-center mb-8">
        <div
          class="bg-indigo-500/20 w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4 border border-indigo-500/30"
        >
          <CircleStackIcon class="w-8 h-8 text-indigo-400" />
        </div>
        <h1 class="text-2xl font-black tracking-tight text-white">Join Table</h1>
      </div>

      <div class="space-y-4">
        <div>
          <label class="block text-sm font-medium text-slate-400 mb-1">Player Name</label>
          <input
            v-model="playerName"
            @keyup.enter="joinGame"
            class="w-full bg-slate-900 border border-slate-600 rounded-xl px-4 py-3 text-white focus:outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 transition-all placeholder-slate-500"
          />
        </div>
        <button
          @click="joinGame"
          :disabled="!playerName"
          class="w-full flex items-center justify-center gap-2 bg-indigo-600 hover:bg-indigo-500 disabled:bg-slate-700 disabled:text-slate-500 transition-colors text-white font-bold py-3 px-4 rounded-xl shadow-lg shadow-indigo-900/20 active:scale-95"
        >
          Join Game <ArrowRightEndOnRectangleIcon class="w-5 h-5" />
        </button>
      </div>
    </div>

    <TransitionRoot appear :show="isWinnerDialogOpen" as="template">
      <Dialog as="div" @close="closeWinnerDialog" class="relative z-50">
        <TransitionChild
          as="template"
          enter="duration-300 ease-out"
          enter-from="opacity-0"
          enter-to="opacity-100"
          leave="duration-200 ease-in"
          leave-from="opacity-100"
          leave-to="opacity-0"
        >
          <div class="fixed inset-0 bg-black/60 backdrop-blur-sm" />
        </TransitionChild>

        <div class="fixed inset-0 overflow-y-auto">
          <div class="flex min-h-full items-center justify-center p-4 text-center">
            <TransitionChild
              as="template"
              enter="duration-300 ease-out"
              enter-from="opacity-0 scale-95"
              enter-to="opacity-100 scale-100"
              leave="duration-200 ease-in"
              leave-from="opacity-100 scale-100"
              leave-to="opacity-0 scale-95"
            >
              <DialogPanel
                class="w-full max-w-sm transform overflow-hidden rounded-2xl bg-slate-800 p-6 text-left align-middle shadow-2xl transition-all border border-slate-700"
              >
                <DialogTitle
                  as="h3"
                  class="text-xl font-black leading-6 text-white mb-4 flex items-center gap-2"
                >
                  <TrophyIcon class="w-6 h-6 text-yellow-500" /> Select Winners
                </DialogTitle>

                <div
                  v-if="!room?.pots || room.pots.length === 0"
                  class="text-slate-400 text-sm mb-4"
                >
                  No pots available. Please ensure the round is completed.
                </div>

                <div v-else class="space-y-5 max-h-[60vh] overflow-y-auto pr-2">
                  <div
                    v-for="(pot, index) in room.pots"
                    :key="index"
                    class="bg-slate-900/50 p-4 rounded-xl border border-slate-700/50"
                  >
                    <div class="flex justify-between items-end mb-3">
                      <span class="text-sm font-bold text-slate-300">
                        {{ index === 0 ? 'Main Pot' : `Side Pot ${index}` }}
                      </span>
                      <span class="text-lg font-black text-yellow-400">{{ pot.amount }} chips</span>
                    </div>

                    <div class="space-y-2">
                      <button
                        v-for="playerName in pot.participants"
                        :key="playerName"
                        @click="winners[index] = playerName"
                        class="w-full flex items-center justify-between p-3 rounded-lg border transition-all active:scale-[0.98]"
                        :class="
                          winners[index] === playerName
                            ? 'bg-amber-500/20 border-amber-500 text-white shadow-[0_0_15px_rgba(245,158,11,0.2)]'
                            : 'bg-slate-800 border-slate-600 text-slate-400 hover:bg-slate-700 hover:text-slate-200'
                        "
                      >
                        <span class="font-bold">{{ playerName }}</span>
                        <CheckCircleIcon
                          v-if="winners[index] === playerName"
                          class="w-5 h-5 text-amber-500"
                        />
                      </button>
                    </div>
                  </div>
                </div>

                <div class="mt-6 flex gap-3">
                  <button
                    @click="closeWinnerDialog"
                    class="flex-1 bg-slate-700 hover:bg-slate-600 text-white font-bold py-3 px-4 rounded-xl transition-colors active:scale-95"
                  >
                    Cancel
                  </button>
                  <button
                    @click="pushPots"
                    :disabled="!room?.pots || Object.keys(winners).length !== room.pots.length"
                    class="flex-1 bg-gradient-to-r from-yellow-500 to-amber-600 hover:from-yellow-400 hover:to-amber-500 disabled:from-slate-700 disabled:to-slate-700 disabled:text-slate-500 disabled:cursor-not-allowed text-slate-900 font-black py-3 px-4 rounded-xl shadow-lg transition-all active:scale-95"
                  >
                    Confirm Push
                  </button>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </Dialog>
    </TransitionRoot>

    <TransitionRoot appear :show="isBetDialogOpen" as="template">
      <Dialog as="div" @close="closeBetDialog" class="relative z-50">
        <TransitionChild
          as="template"
          enter="duration-300 ease-out"
          enter-from="opacity-0"
          enter-to="opacity-100"
          leave="duration-200 ease-in"
          leave-from="opacity-100"
          leave-to="opacity-0"
        >
          <div class="fixed inset-0 bg-black/60 backdrop-blur-sm" />
        </TransitionChild>

        <div class="fixed inset-0 overflow-y-auto">
          <div class="flex min-h-full items-center justify-center p-4 text-center">
            <TransitionChild
              as="template"
              enter="duration-300 ease-out"
              enter-from="opacity-0 scale-95"
              enter-to="opacity-100 scale-100"
              leave="duration-200 ease-in"
              leave-from="opacity-100 scale-100"
              leave-to="opacity-0 scale-95"
            >
              <DialogPanel
                class="w-full max-w-sm transform overflow-hidden rounded-2xl bg-slate-800 p-6 text-left align-middle shadow-2xl transition-all border border-slate-700"
              >
                <DialogTitle
                  as="h3"
                  class="text-xl font-black leading-6 text-white mb-4 flex items-center gap-2"
                >
                  <ArrowUpCircleIcon class="w-6 h-6 text-indigo-500" /> Bet Amount
                </DialogTitle>

                <div class="space-y-4">
                  <div>
                    <label class="block text-xs font-medium text-slate-400 uppercase mb-2">
                      Amount to Bet
                    </label>
                    <input
                      type="number"
                      v-model.number="betAmount"
                      :min="room?.current_highest_bet"
                      :max="currentPlayer?.chips"
                      @keyup.enter="bet"
                      class="w-full bg-slate-900 border border-slate-600 rounded-xl px-4 py-3 text-2xl font-black text-white text-center focus:outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 transition-all"
                      :placeholder="room?.current_highest_bet"
                    />
                  </div>

                  <p class="text-xs text-center text-slate-400">
                    Max: <span class="text-emerald-400">{{ currentPlayer?.chips }}</span> chips
                  </p>
                </div>

                <div class="mt-6 flex gap-3">
                  <button
                    @click="closeBetDialog"
                    class="flex-1 bg-slate-700 hover:bg-slate-600 text-white font-bold py-3 px-4 rounded-xl transition-colors active:scale-95"
                  >
                    Cancel
                  </button>
                  <button
                    @click="bet"
                    class="flex-1 bg-indigo-600 hover:bg-indigo-500 text-white font-bold py-3 px-4 rounded-xl shadow-lg shadow-indigo-900/20 transition-colors active:scale-95"
                  >
                    Confirm Bet
                  </button>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </Dialog>
    </TransitionRoot>
    <TransitionRoot appear :show="isPlayersDialogOpen" as="template">
      <Dialog as="div" @close="closePlayersDialog" class="relative z-50">
        <TransitionChild
          as="template"
          enter="duration-300 ease-out"
          enter-from="opacity-0"
          enter-to="opacity-100"
          leave="duration-200 ease-in"
          leave-from="opacity-100"
          leave-to="opacity-0"
        >
          <div class="fixed inset-0 bg-black/60 backdrop-blur-sm" />
        </TransitionChild>

        <div class="fixed inset-0 overflow-y-auto">
          <div class="flex min-h-full items-center justify-center p-4 text-center">
            <TransitionChild
              as="template"
              enter="duration-300 ease-out"
              enter-from="opacity-0 scale-95"
              enter-to="opacity-100 scale-100"
              leave="duration-200 ease-in"
              leave-from="opacity-100 scale-100"
              leave-to="opacity-0 scale-95"
            >
              <DialogPanel
                class="w-full max-w-sm transform overflow-hidden rounded-2xl bg-slate-800 p-6 text-left align-middle shadow-2xl transition-all border border-slate-700"
              >
                <DialogTitle
                  as="h3"
                  class="text-xl font-black leading-6 text-white mb-4 flex items-center justify-between"
                >
                  <div class="flex items-center gap-2">
                    <UsersIcon class="w-6 h-6 text-indigo-400" /> Players List
                  </div>
                  <span
                    class="text-sm font-medium text-slate-400 bg-slate-900 px-3 py-1 rounded-full border border-slate-700"
                  >
                    Total: {{ room?.player_count || 0 }}
                  </span>
                </DialogTitle>

                <div class="space-y-3 max-h-[60vh] overflow-y-auto pr-2">
                  <div
                    v-for="(player, pName) in room.players"
                    :key="pName"
                    class="bg-slate-900/50 p-4 rounded-xl border border-slate-700/50 flex items-center justify-between"
                  >
                    <div class="flex flex-col gap-1">
                      <div class="flex items-center gap-2">
                        <span class="font-bold text-slate-200">{{ pName }}</span>
                        <StarIcon v-if="player.is_dealer" class="w-4 h-4 text-blue-400" />
                      </div>
                      <div>
                        <span
                          class="text-xs font-bold px-2 py-0.5 rounded-full uppercase"
                          :class="{
                            'bg-emerald-500/20 text-emerald-400': player.state === 'active',
                            'bg-red-500/20 text-red-400': player.state === 'all_in',
                            'bg-slate-700 text-slate-400': player.state === 'fold',
                            'bg-slate-900 text-slate-600': player.state === 'dead',
                          }"
                        >
                          {{ player.state }}
                        </span>
                      </div>
                    </div>

                    <div class="text-right">
                      <div class="flex items-center justify-end gap-1 text-emerald-400 font-black">
                        <CircleStackIcon class="w-4 h-4" />
                        {{ player.chips }}
                      </div>
                      <div
                        v-if="player.current_bet > 0"
                        class="text-xs text-yellow-500 font-medium mt-1"
                      >
                        Bet: {{ player.current_bet }}
                      </div>
                    </div>
                  </div>
                </div>

                <div class="mt-6 flex gap-3">
                  <button
                    @click="closePlayersDialog"
                    class="w-full bg-slate-700 hover:bg-slate-600 text-white font-bold py-3 px-4 rounded-xl transition-colors active:scale-95"
                  >
                    Close
                  </button>
                </div>
              </DialogPanel>
            </TransitionChild>
          </div>
        </div>
      </Dialog>
    </TransitionRoot>
  </div>
</template>
