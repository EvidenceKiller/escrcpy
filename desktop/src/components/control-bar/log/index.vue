<template>
  <slot :trigger="handleLog" :loading="loading" />
</template>

<script setup>
defineOptions({
  inheritAttrs: false,
})

const props = defineProps({
  device: {
    type: Object,
    default: () => ({}),
  },
  floating: {
    type: Boolean,
    default: false,
  },
})

const preferenceStore = usePreferenceStore()

const loading = ref(false)

async function handleLog() {
  const device = toRaw(props.device ?? {})

  const deviceConfig = preferenceStore.getDataWithFallback(device.id)
  const saveDir = deviceConfig.saveDir

  if (!saveDir) {
    ElMessage.warning(
      window.t('device.control.log.error.no-save-dir'),
    )
    return
  }

  // file named after the device ip (host part of `ip:port`,
  // falling back to the serial number for usb devices)
  const [ip] = device.id.split(':')
  const savePath = window.$preload.path.join(saveDir, `${ip}.log`)

  loading.value = true

  try {
    const { savePath: resultPath } = await window.$preload.adb.logcat(
      device.id,
      { savePath },
    )

    ElMessage.success(
      `${window.t('device.control.log.started')}${resultPath}`,
    )
  }
  catch (error) {
    ElMessage.warning(error?.message || window.t('device.control.log.name'))
  }
  finally {
    loading.value = false
  }
}
</script>

<style></style>
