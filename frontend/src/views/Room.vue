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