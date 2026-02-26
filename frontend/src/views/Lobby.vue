<script setup>
import { useRouter } from 'vue-router'
const router = useRouter()

//roomIdを大小英数字で生成
const generateId = () => {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  let result = ''
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return result
}
</script>

<template>
  <button @click="router.push('/test_room')">room</button>
</template>
