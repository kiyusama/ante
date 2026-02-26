<script setup>
import { useRouter } from 'vue-router'
const router = useRouter()
</script>

<template>
  <button @click="router.push('/test_room')">room</button>
</template>
