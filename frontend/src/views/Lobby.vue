<script setup>
import { useRouter } from 'vue-router'
const router = useRouter()
import { ref as dbRef, set } from 'firebase/database'
import { db } from '../firebase/config'

//roomIdを大小英数字で生成
const generateId = () => {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  let length = 5
  let result = ''
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return result
}

const createRoom = async () => {
  const roomId = generateId()

  const newRoomRef = dbRef(db, `rooms/${roomId}`)

  await set(newRoomRef, {
    pot: 0,
  })

  //作成後ルームに遷移
  router.push(`/${roomId}`)
}
</script>

<template>
  <button @click="createRoom()">Create Room</button>
</template>
