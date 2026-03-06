<script setup>
import { ref as dbRef, set } from 'firebase/database'
import { useRouter } from 'vue-router'
import { db } from '../firebase/config'

const router = useRouter()

const navItems = ['Lobby', 'How to play', 'Settings', 'Contact us']

// シンプルな英数字コードを生成し、URL 共有をしやすくする
const generateId = () => {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  let result = ''
  for (let i = 0; i < 5; i += 1) {
    result += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return result
}

// ルームを即時に作成し、完了後ロビーからルーム画面へ遷移させる
const createRoom = async () => {
  const roomId = generateId()
  const newRoomRef = dbRef(db, `rooms/${roomId}`)

  await set(newRoomRef, {
    pot: 0,
    player_count: 0,
    current_highest_bet: 0,
    stack: 1000,
    waiting: true,
    history: [],
  })

  router.push(`/${roomId}`)
}
</script>

<template>
  <section
    class="flex min-h-screen w-full flex-col items-stretch justify-start gap-[clamp(2rem,5vw,3.5rem)] bg-[var(--ante-crimson)] pb-[clamp(3rem,6vw,4.5rem)] text-center text-[var(--ante-cream)]"
  >
    <!-- 上部タブ: 画像イメージのノコギリ状タブを再現 -->
    <nav class="sticky top-0 z-20 mb-6 flex w-full justify-between gap-0 bg-[var(--ante-crimson)] pb-2" aria-label="Primary">
      <span
        v-for="(label, index) in navItems"
        :key="label"
        class="flex-1 px-2 py-2.5 text-[0.85rem] uppercase tracking-[0.08em] [clip-path:polygon(0_0,100%_0,90%_100%,10%_100%)]"
        :class="index % 2 === 1 ? 'bg-[var(--ante-tab-cream)] text-[var(--ante-ink)]' : 'bg-[var(--ante-tab)] text-[var(--ante-cream)]'"
      >
        {{ label }}
      </span>
    </nav>

    <!-- 中央ヒーロー: ANTE タイトルとボタンを縦積みに配置 -->
    <div
      class="mx-auto flex w-[min(420px,90vw)] flex-1 flex-col items-center justify-center gap-8 px-5 pt-[clamp(0.5rem,4vw,1.5rem)]"
    >
      <h1 class="text-[clamp(4rem,14vw,6rem)] tracking-[0.4rem]">ANTE</h1>

      <div class="flex w-full flex-col gap-[clamp(0.4rem,1vw,0.75rem)]">
        <button
          class="border-0 bg-[var(--ante-cream)] px-6 py-4 text-[1.05rem] uppercase tracking-[0.2em] text-[var(--ante-ink)] shadow-[0_20px_35px_var(--ante-shadow)] transition-transform duration-200 [clip-path:polygon(12%_0,88%_0,100%_100%,0_100%)]"
          type="button"
          @click="createRoom"
        >
          Create Room
        </button>

        <!-- Join ボタンは実装待ちのため非活性のダミー -->
        <button
          class="border-0 bg-[var(--ante-ink)] px-6 py-4 text-[1.05rem] uppercase tracking-[0.2em] text-[var(--ante-cream)] [clip-path:polygon(0_0,100%_0,88%_100%,12%_100%)]"
          type="button"
          disabled
        >
          Join
        </button>
      </div>
    </div>
  </section>
</template>
