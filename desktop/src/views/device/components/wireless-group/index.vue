<template>
  <div class="flex items-center flex-none !space-x-2">
    <div class="w-56 flex-none">
      <el-autocomplete
        :key="autocompleteKey"
        ref="elAutocompleteRef"
        v-model="host"
        :placeholder="$t('device.wireless.connect.host')"
        clearable
        :fetch-suggestions="fetchSuggestions"
        class="!w-full el-autocomplete--wireless"
        value-key="host"
        @select="onSelect"
        @focus="onAutocompleteFocus"
        @input="isTyping = true"
        @keydown.enter.prevent="keydownConnect()"
        @keydown.escape="handleUnConnect()"
      >
        <template #prepend>
          <el-button icon="Switch" :title="$t('device.wireless.switch')" @click="onPairToggle()">
          </el-button>
        </template>

        <template #default="{ item }">
          <div
            v-if="item.batch"
            class="text-primary-500"
            @click.stop="handleBatch"
          >
            {{ item.batch }}
          </div>

          <div v-else class="flex items-center">
            <div class="flex-1 w-0 truncate">
              {{ item.id }}
            </div>
            <div
              class="flex-none leading-none"
              @click.prevent.stop="handleRemove(item)"
            >
              <el-icon class="hover:text-primary-500 !active:text-primary-700">
                <Close />
              </el-icon>
            </div>
          </div>
        </template>
      </el-autocomplete>
    </div>

    <div class="w-24 flex-none">
      <el-input
        v-model="port"
        :placeholder="$t('device.wireless.connect.port')"
        :title="$t('device.wireless.connect.port')"
        @keydown.enter.prevent="keydownConnect()"
        @keydown.escape="handleUnConnect()"
      ></el-input>
    </div>

    <div class="flex-none overflow-hidden transition-all duration-300" :class="pairVisible ? 'w-26' : 'w-0 !mx-0'">
      <el-input
        v-model="pairCode"
        class="!w-full"
        :placeholder="$t('device.wireless.pair.code')"
        :title="$t('device.wireless.pair.code')"
        @keydown.enter.prevent="keydownConnect('pair')"
        @keydown.escape="handleUnConnect()"
      ></el-input>
    </div>

    <el-button-group class="flex-none">
      <el-button
        type="default"
        :icon="loading ? '' : 'Connection'"
        :loading="loading"
        class="flex-none"
        @click="handleConnect()"
      >
        {{ $t('device.wireless.connect.name') }}
      </el-button>

      <el-button
        v-if="loading"
        type="default"
        class="flex-none"
        @click="handleUnConnect()"
      >
        {{ $t('common.cancel') }}
      </el-button>
    </el-button-group>

    <div class="w-32 flex-none">
      <el-input
        v-model="password"
        type="password"
        show-password
        :placeholder="$t('device.wireless.login.name')"
        :title="$t('device.wireless.login.name')"
        @keydown.enter.prevent="handleShellLogin()"
        @keydown.escape="handleUnConnect()"
      ></el-input>
    </div>

    <el-button
      type="default"
      :icon="loginLoading ? '' : 'Key'"
      :loading="loginLoading"
      class="flex-none"
      :title="$t('device.wireless.login.name')"
      @click="handleShellLogin()"
    >
      {{ $t('device.wireless.login.name') }}
    </el-button>

    <QrAction v-bind="{ handleRefresh }" />

    <DiscoverAction v-bind="{ handleRefresh }" />
  </div>
</template>

<script setup>
import pLimit from 'p-limit'
import { sleep } from '$/utils'
import { parseDeviceId } from '$/utils/device'
import DiscoverAction from './discover-action/index.vue'
import QrAction from './qr-action/index.vue'

const props = defineProps({
  handleRefresh: {
    type: Function,
    default: () => () => false,
  },
})

const emit = defineEmits(['auto-connected'])

const handleRefresh = props.handleRefresh

const preferenceStore = usePreferenceStore()
const deviceStore = useDeviceStore()

const loading = ref(false)
const loginLoading = ref(false)
const host = ref('')
const port = ref('5555')
const password = useStorage('device-wireless-password', '')
// password used for the last successful connection of each device, keyed by `ip:port`
const passwordMap = useStorage('device-wireless-password-map', {})
const autocompleteKey = ref(0)
const pairCode = ref('')
const pairVisible = useStorage('device-wireless-pair-visible', false)

const elAutocompleteRef = ref()
const isTyping = ref(false)

function onAutocompleteFocus() {
  isTyping.value = false
}

const wirelessList = computed(() =>
  deviceStore.list.filter(item => item.wifi),
)

onMounted(() => {
  const unwatch = watch(
    wirelessList,
    async (val) => {
      unwatch()
      if (!val)
        return
      handleConnectAuto()
    },
  )
  getAddress()
})

function keydownConnect(source) {
  if (source === 'pair' || !pairVisible.value) {
    handleConnect()
  }
}

function onPairToggle(val) {
  pairVisible.value = val ?? !pairVisible.value

  if (!pairVisible.value) {
    pairCode.value = ''
  }
}

async function handleConnectAuto() {
  if (!preferenceStore.data.autoConnect) {
    return
  }

  await handleBatch()
  handleRefresh()
  emit('auto-connected')
}

function getAddress() {
  const last = wirelessList.value.at(-1)
  if (!last)
    return

  const { host: lastHost, port: lastPort } = parseDeviceId(last.id)
  host.value = lastHost
  port.value = String(lastPort)
}

function fetchSuggestions(value, callback) {
  let results = []

  // only filter while the user is typing; clicking/focusing the input shows the full history
  if (value && isTyping.value) {
    results = wirelessList.value.filter(item =>
      item.id.toLowerCase().startsWith(value.toLowerCase()),
    )
  }
  else {
    results = [...wirelessList.value]
  }

  results.push({
    batch: window.t('device.wireless.connect.batch.name'),
  })

  callback(results)
}

function onSelect(device) {
  const { host: itemHost, port: itemPort } = parseDeviceId(device.id)
  host.value = itemHost
  port.value = String(itemPort)
}

function handleRemove(info) {
  const index = wirelessList.value.findIndex(
    item => item.id === info.id,
  )

  if (index === -1)
    return

  wirelessList.value.splice(index, 1)
  autocompleteKey.value++
}

async function handleBatch() {
  if (loading.value) {
    return
  }

  const list = wirelessList.value

  if (!list.length) {
    return
  }

  loading.value = true

  const concurrencyLimit = Number(window.$preload.store.get('common.concurrencyLimit') ?? 5)
  const limit = pLimit(concurrencyLimit)

  try {
    const tasks = list.map(({ id }) =>
      limit(() =>
        window.$preload.adb
          .connect(id)
          .catch((e) => {
            // console.warn(`Failed to connect ${id}: ${e.message}`)
          }),
      ),
    )

    await Promise.allSettled(tasks)
  }
  finally {
    loading.value = false
  }
}

function handleUnConnect() {
  loading.value = false
  loginLoading.value = false
}

async function handleConnect(addr = '') {
  let targetHost = addr || host.value.trim()
  let targetPort

  if (addr) {
    const parsed = parseDeviceId(addr)
    targetHost = parsed.host
    targetPort = parsed.port

    // sync the bottom inputs with the device being connected from the list
    host.value = targetHost
    port.value = String(targetPort)
    password.value = passwordMap.value[`${targetHost}:${targetPort}`] || ''
  }
  else {
    targetPort = Number.parseInt(port.value, 10)

    if (!(targetPort > 0 && targetPort <= 65535)) {
      targetPort = 5555
    }
  }

  const targetAddress = `${targetHost}:${targetPort}`

  if (!targetHost) {
    ElMessage.warning(
      window.t('device.wireless.connect.error.no-address'),
    )
    return
  }

  if (pairCode.value) {
    try {
      await window.$preload.adb.pair(targetHost, targetPort, pairCode.value)
      onPairToggle(false)

      await sleep()
      ElMessage.success({
        message: window.t('device.wireless.pair.success'),
        duration: 3000,
      })
    }
    catch (error) {
      console.warn(error.message)
      ElMessage.warning(window.t('device.wireless.pair.error'))
    }

    return
  }

  loading.value = true

  try {
    await window.$preload.adb.connect(
      targetHost,
      targetPort,
    )

    await sleep()
    ElMessage.success(window.t('device.wireless.connect.success'))
  }
  catch (error) {
    ElMessage.warning(error?.message || error?.cause?.message || error)
  }
  finally {
    loading.value = false
  }

  handleRefresh()
  loading.value = false
}

/**
 * Runs `adb shell <password>` (fire and forget), then verifies the login by
 * opening an interactive `adb shell`: if the session can be entered, the
 * login is OK; error/closed output marks the login as failed.
 * When called with a device id from the device list, the remembered password
 * of that device is used and the bottom inputs are synced.
 */
async function handleShellLogin(addr = '') {
  let targetHost = addr || host.value.trim()
  let targetPort

  if (addr) {
    const parsed = parseDeviceId(addr)
    targetHost = parsed.host
    targetPort = parsed.port

    // sync the bottom inputs with the device being logged in from the list
    host.value = targetHost
    port.value = String(targetPort)
    password.value = passwordMap.value[`${targetHost}:${targetPort}`] || ''
  }
  else {
    targetPort = Number.parseInt(port.value, 10)

    if (!(targetPort > 0 && targetPort <= 65535)) {
      targetPort = 5555
    }
  }

  if (!targetHost) {
    ElMessage.warning(
      window.t('device.wireless.connect.error.no-address'),
    )
    return
  }

  if (!password.value) {
    ElMessage.warning(
      window.t('device.wireless.login.error.no-password'),
    )
    return
  }

  loginLoading.value = true

  try {
    await window.$preload.adb.shellLogin(targetHost, targetPort, password.value)

    // remember the password used for this device
    passwordMap.value = {
      ...passwordMap.value,
      [`${targetHost}:${targetPort}`]: password.value,
    }

    await sleep()
    ElMessage.success(window.t('device.wireless.login.success'))
  }
  catch (error) {
    ElMessage.warning(error?.message || error?.cause?.message || error)
  }
  finally {
    loginLoading.value = false
  }

  handleRefresh()
}

defineExpose({
  connect: handleConnect,
  shellLogin: handleShellLogin,
})
</script>

<style lang="postcss" scoped>
</style>
