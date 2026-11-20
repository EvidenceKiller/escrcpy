<template>
  <el-button
    type="primary"
    text
    circle
    :loading="loading"
    :icon="loading ? '' : 'Key'"
    :title="$t('device.wireless.login.name')"
    @click="handleClick(device)"
  >
  </el-button>
</template>

<script setup>
const props = defineProps({
  device: {
    type: Object,
    default: () => ({}),
  },
  handleLogin: {
    type: Function,
    default: () => false,
  },
})

const loading = ref(false)

async function handleClick(device) {
  loading.value = true

  await props.handleLogin(device.id)

  loading.value = false
}
</script>

<style></style>
