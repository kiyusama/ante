<script setup>
import { useDatabaseObject } from 'vuefire'
import { ref as dbRef, update, increment } from 'firebase/database'
import { db } from '@/firebase/config'

// データベースの参照先を指定（"rooms/test_room"という場所）
const roomRef = dbRef(db, 'rooms/test_room')

// リアルタイムでデータを取得（room変数の中身が自動で最新に保たれる）
const room = useDatabaseObject(roomRef)

// ボタンを押した時の処理
async function addChips() {
  // potの数値を10増やす
  await update(roomRef, {
    pot: increment(10),
  })
}
</script>

<template>
  <h2>現在のポット: {{ room?.pot || 0 }} 枚</h2>
  <button @click="addChips">ベット</button>
</template>
