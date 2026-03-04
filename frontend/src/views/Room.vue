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
//push pot用
const isWinnerDialogOpen = ref(false)
const openWinnerDialog = () => {
  isWinnerDialogOpen.value = true
}
const closeWinnerDialog = () => {
  isWinnerDialogOpen.value = false
}

// データベースの参照先を指定
// リアルタイムでデータを取得(room情報もplayer情報も含まれる)
const roomRef = dbRef(db, `rooms/${route.params.roomId}`)
const room = useDatabaseObject(roomRef)

const playerName = ref('')
const isJoined = ref(false)

const betAmount = ref(0)

//callに必要な額を常に計算
const callAmount = computed(() => {
  return room.value.current_highest_bet - currentPlayer.value.current_bet
})

//自分の情報を取得
const currentPlayer = computed(() => {
  return room.value?.players?.[playerName.value]
})

//activeなユーザ一覧を取得
const activePlayers = computed(() => {
  if (!room.value?.players) return []
  return Object.values(room.value.players).filter(
    (p) => p.state === 'active' || p.state === 'all_in',
  )
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

  closeBetDialog()
}

//call action
const call = async () => {
  const currentHighestBet = room.value.current_highest_bet

  if (!isAbleToPay(callAmount.value) || currentHighestBet == currentPlayer.value.current_bet) {
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

//dealer positionの人が押す
//ゲーム開始の合図
const declareDealer = async () => {
  await saveSnap()

  await update(roomRef, {
    waiting: false,
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
  }

  Object.keys(room.value.players).forEach((playerNameKey, player) => {
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

//potを勝者に渡す
//dealer専用
const pushPot = async (winnerName) => {
  await saveSnap()

  const potValue = room.value.pot
  const updates = {
    pot: 0,
    current_highest_bet: 0,
    waiting: true,
    [`players/${winnerName}/chips`]: increment(potValue),
  }
  //すべてのplayerのcurrent_betを初期化
  Object.keys(room.value.players).forEach((playerNameKey) => {
    updates[`players/${playerNameKey}/current_bet`] = 0
    updates[`players/${playerNameKey}/state`] = 'active'
    updates[`players/${playerNameKey}/is_dealer`] = false
  })
  await update(roomRef, updates)
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
      </div>

      <div class="bg-slate-800 rounded-2xl shadow-xl p-6 border border-slate-700">
        <div v-if="room?.waiting" class="flex flex-col items-center justify-center py-4">
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
              :disabled="currentPlayer?.state === 'fold'"
              class="flex-1 flex flex-col items-center justify-center gap-1 bg-slate-700 hover:bg-slate-600 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:bg-slate-700 disabled:active:scale-100 transition-colors border border-slate-600 text-white font-bold py-3 px-4 rounded-xl active:scale-95"
            >
              <CheckCircleIcon
                class="w-6 h-6"
                :class="currentPlayer?.state === 'fold' ? 'text-slate-400' : 'text-emerald-400'"
              />
              <span>Call {{ callAmount > 0 ? '+' + callAmount : '' }}</span>
            </button>

            <button
              @click="openBetDialog"
              :disabled="currentPlayer?.state === 'fold'"
              class="flex-1 flex flex-col items-center justify-center gap-1 bg-indigo-600 hover:bg-indigo-500 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:bg-indigo-600 disabled:active:scale-100 transition-colors border border-indigo-500 text-white font-bold py-3 px-4 rounded-xl active:scale-95 shadow-lg shadow-indigo-900/20 disabled:shadow-none"
            >
              <ArrowUpCircleIcon
                class="w-6 h-6"
                :class="currentPlayer?.state === 'fold' ? 'text-slate-400' : 'text-indigo-300'"
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
            <ForwardIcon class="w-5 h-5" /> Next Round
          </button>
          <button
            @click="openWinnerDialog"
            class="flex-1 flex flex-col items-center justify-center gap-1 bg-gradient-to-br from-yellow-400 to-amber-600 hover:to-amber-500 transition-colors text-slate-900 font-black py-3 px-4 rounded-xl shadow-lg shadow-amber-900/20 active:scale-95"
          >
            <TrophyIcon class="w-6 h-6" />
            <span>Push Pot</span>
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
                  <TrophyIcon class="w-6 h-6 text-yellow-500" /> Select Winner
                </DialogTitle>

                <p class="text-sm text-slate-400 mb-4">
                  Pot: <span class="font-bold text-yellow-500">{{ room?.pot }} chips</span>
                </p>

                <div class="space-y-3">
                  <button
                    v-for="player in activePlayers"
                    :key="player.name"
                    @click="pushPot(player.name)"
                    class="w-full flex items-center justify-between p-4 rounded-xl bg-slate-700/50 hover:bg-slate-600 transition-colors border border-slate-600 active:scale-95"
                  >
                    <span class="text-white font-bold">{{ player.name }}</span>
                    <span
                      class="text-xs font-semibold px-2 py-1 rounded-full bg-slate-800 text-slate-300"
                    >
                      {{ player.state }}
                    </span>
                  </button>

                  <div
                    v-if="activePlayers.length === 0"
                    class="text-center text-slate-500 text-sm py-4"
                  >
                    No active players found.
                  </div>
                </div>

                <div class="mt-6 text-center">
                  <button
                    @click="closeWinnerDialog"
                    class="text-sm font-medium text-slate-400 hover:text-white transition-colors"
                  >
                    Cancel
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
                      @keyup.enter="bet(betAmount)"
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
                    @click="bet(betAmount)"
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
  </div>
</template>
