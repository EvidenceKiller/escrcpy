<template>
  <slot :trigger="handleClick" />
</template>

<script setup>
defineOptions({
  inheritAttrs: false,
})

const loading = ref(false)

async function handleClick() {
  if (loading.value) {
    return
  }

  loading.value = true

  try {
    await window.$preload.adb.restart()
    ElMessage.success(window.t('device.adb.restart.success'))
  }
  catch (error) {
    console.warn(error?.message)
    ElMessage.warning(error?.message || window.t('device.adb.restart.error'))
  }
  finally {
    loading.value = false
  }
}
</script>

<style lang="postcss"></style>
