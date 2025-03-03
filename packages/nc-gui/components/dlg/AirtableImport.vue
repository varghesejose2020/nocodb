<script setup lang="ts">
import type { Card as AntCard } from 'ant-design-vue'
import { JobStatus } from '#imports'

const { modelValue, baseId, sourceId, transition } = defineProps<{
  modelValue: boolean
  baseId: string
  sourceId: string
  transition?: string
}>()

const emit = defineEmits(['update:modelValue', 'back'])

const { $api } = useNuxtApp()

const baseURL = $api.instance.defaults.baseURL

const { $state, $poller } = useNuxtApp()

const baseStore = useBase()

const { refreshCommandPalette } = useCommandPalette()

const { loadTables } = baseStore

const { getJobsForBase, loadJobsForBase } = useJobs()

const showGoToDashboardButton = ref(false)

const step = ref(1)

const progress = ref<Record<string, any>[]>([])

const logRef = ref<typeof AntCard>()

const enableAbort = ref(false)

const goBack = ref(false)

const listeningForUpdates = ref(false)

const syncSource = ref({
  id: '',
  type: 'Airtable',
  details: {
    syncInterval: '15mins',
    syncDirection: 'Airtable to NocoDB',
    syncRetryCount: 1,
    apiKey: '',
    appId: '',
    shareId: '',
    syncSourceUrlOrId: '',
    options: {
      syncViews: true,
      syncData: true,
      syncRollup: false,
      syncLookup: true,
      syncFormula: false,
      syncAttachment: true,
      syncUsers: false,
    },
  },
})

const pushProgress = async (message: string, status: JobStatus | 'progress') => {
  progress.value.push({ msg: message, status })

  await nextTick(() => {
    const container: HTMLDivElement = logRef.value?.$el?.firstElementChild
    if (!container) return
    container.scrollTop = container.scrollHeight
  })
}

const onStatus = async (status: JobStatus, data?: any) => {
  if (status === JobStatus.COMPLETED) {
    showGoToDashboardButton.value = true
    await loadTables()
    pushProgress('Done!', status)
    refreshCommandPalette()
    // TODO: add tab of the first table
  } else if (status === JobStatus.FAILED) {
    await loadTables()
    goBack.value = true
    pushProgress(data.error.message, status)
    refreshCommandPalette()
  }
}

const onLog = (data: { message: string }) => {
  pushProgress(data.message, 'progress')
}

const validators = computed(() => ({
  'details.apiKey': [fieldRequiredValidator()],
  'details.syncSourceUrlOrId': [fieldRequiredValidator()],
}))

const dialogShow = computed({
  get: () => modelValue,
  set: (v) => emit('update:modelValue', v),
})

const useForm = Form.useForm

const { validateInfos } = useForm(syncSource, validators)

const disableImportButton = computed(() => !syncSource.value.details.apiKey || !syncSource.value.details.syncSourceUrlOrId)

const isLoading = ref(false)

async function saveAndSync() {
  isLoading.value = true
  await createOrUpdate()
  await sync()
}

async function createOrUpdate() {
  try {
    const { id, ...payload } = syncSource.value

    if (id !== '') {
      await $fetch(`/api/v1/db/meta/syncs/${id}`, {
        baseURL,
        method: 'PATCH',
        headers: { 'xc-auth': $state.token.value as string },
        body: payload,
      })
    } else {
      syncSource.value = await $fetch(`/api/v1/db/meta/projects/${baseId}/syncs/${sourceId}`, {
        baseURL,
        method: 'POST',
        headers: { 'xc-auth': $state.token.value as string },
        body: payload,
      })
    }
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

async function listenForUpdates(id?: string) {
  if (listeningForUpdates.value) return

  listeningForUpdates.value = true

  await loadJobsForBase(baseId)

  const jobs = await getJobsForBase(baseId)

  const job = id
    ? { id }
    : jobs
        // sort by created_at desc (latest first)
        .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
        .find((j) => j.base_id === baseId && j.status !== JobStatus.COMPLETED && j.status !== JobStatus.FAILED)

  if (!job) {
    listeningForUpdates.value = false
    return
  }

  $poller.subscribe(
    { id: job.id },
    (data: {
      id: string
      status?: string
      data?: {
        error?: {
          message: string
        }
        message?: string
        result?: any
      }
    }) => {
      if (data.status !== 'close') {
        step.value = 2
        if (data.status) {
          onStatus(data.status as JobStatus, data.data)
        } else {
          onLog(data.data as any)
        }
      } else {
        listeningForUpdates.value = false
        isLoading.value = false
      }
    },
  )
}

async function loadSyncSrc() {
  const data: any = await $fetch(`/api/v1/db/meta/projects/${baseId}/syncs/${sourceId}`, {
    baseURL,
    method: 'GET',
    headers: { 'xc-auth': $state.token.value as string },
  })

  const { list: srcs } = data

  if (srcs && srcs[0]) {
    srcs[0].details = srcs[0].details || {}
    syncSource.value = migrateSync(srcs[0])
    syncSource.value.details.syncSourceUrlOrId =
      srcs[0].details.appId && srcs[0].details.appId.length > 0 ? srcs[0].details.syncSourceUrlOrId : srcs[0].details.shareId
    listenForUpdates()
  } else {
    syncSource.value = {
      id: '',
      type: 'Airtable',
      details: {
        syncInterval: '15mins',
        syncDirection: 'Airtable to NocoDB',
        syncRetryCount: 1,
        apiKey: '',
        appId: '',
        shareId: '',
        syncSourceUrlOrId: '',
        options: {
          syncViews: true,
          syncData: true,
          syncRollup: false,
          syncLookup: true,
          syncFormula: false,
          syncAttachment: true,
          syncUsers: false,
        },
      },
    }
  }
}

async function sync() {
  try {
    const jobData: any = await $fetch(`/api/v1/db/meta/syncs/${syncSource.value.id}/trigger`, {
      baseURL,
      method: 'POST',
      headers: { 'xc-auth': $state.token.value as string },
    })
    listeningForUpdates.value = false
    listenForUpdates(jobData.id)
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

async function abort() {
  Modal.confirm({
    title: 'Are you sure you want to abort this job?',
    type: 'warn',
    content:
      "This is a highly experimental feature and only marks job as not started, please don't abort the job unless you are sure job is stuck.",
    onOk: async () => {
      try {
        await $fetch(`/api/v1/db/meta/syncs/${syncSource.value.id}/abort`, {
          baseURL,
          method: 'POST',
          headers: { 'xc-auth': $state.token.value as string },
        })
        step.value = 1
        progress.value = []
        goBack.value = false
        enableAbort.value = false
      } catch (e: any) {
        message.error(await extractSdkResponseErrorMsg(e))
      }
    },
  })
}

function cancel() {
  step.value = 1
  progress.value = []
  goBack.value = false
  enableAbort.value = false
}

function migrateSync(src: any) {
  if (!src.details?.options) {
    src.details.options = {
      syncViews: false,
      syncData: true,
      syncRollup: false,
      syncLookup: true,
      syncFormula: false,
      syncAttachment: true,
    }
    src.details.options.syncViews = src.syncViews
    delete src.syncViews
  }

  return src
}

watch(
  () => syncSource.value.details.syncSourceUrlOrId,
  (v) => {
    if (syncSource.value.details) {
      const m = v && v.match(/(exp|shr).{14}/g)
      syncSource.value.details.shareId = m ? m[0] : ''
      const m2 = v && v.match(/(app).{14}/g)
      syncSource.value.details.appId = m2 ? m2[0] : ''
    }
  },
)

onMounted(async () => {
  if (syncSource.value.id) {
    listenForUpdates()
  }
  await loadSyncSrc()
})

function downloadLogs(filename: string) {
  let text = ''
  for (const o of document.querySelectorAll('.nc-modal-airtable-import .log-message')) {
    text += `${o.textContent}\n`
  }
  const element = document.createElement('a')
  element.setAttribute('href', `data:text/plain;charset=utf-8,${encodeURIComponent(text)}`)
  element.setAttribute('download', filename)

  element.style.display = 'none'
  document.body.appendChild(element)

  element.click()

  document.body.removeChild(element)
}

const isInProgress = computed(() => {
  return (
    !progress.value ||
    !progress.value.length ||
    ![JobStatus.COMPLETED, JobStatus.FAILED].includes(progress.value[progress.value.length - 1]?.status)
  )
})

const detailsIsShown = ref(false)
const collapseKey = ref('')
</script>

<template>
  <a-modal
    v-model:visible="dialogShow"
    class="!top-[25vh]"
    :class="{ active: dialogShow }"
    :closable="false"
    :transition-name="transition"
    :keyboard="step !== 2"
    :mask-closable="step !== 2"
    width="448px"
    wrap-class-name="nc-modal-airtable-import"
    hide
    @keydown.esc="dialogShow = false"
  >
    <div class="text-base font-weight-bold flex items-center gap-4 mb-6" @dblclick="enableAbort = true">
      <GeneralIcon icon="airtable" class="w-6 h-6" />

      <span v-if="step === 1">
        {{ $t('title.quickImportAirtable') }}
      </span>
      <span v-else-if="isInProgress"> {{ `${$t('labels.importingFromAirtable')}...` }} </span>
      <span v-else> {{ $t('labels.airtableBaseImported') }} </span>

      <a
        v-if="step === 1"
        href="https://docs.nocodb.com/bases/import-base-from-airtable#get-airtable-credentials"
        class="!text-gray-500 prose-sm ml-auto"
        target="_blank"
        rel="noopener"
      >
        Docs
      </a>
      <nc-button v-else-if="step === 2" type="text" size="xs" class="ml-auto" @click="detailsIsShown = !detailsIsShown">
        {{ detailsIsShown ? 'Hide' : 'Show' }} Details
        <GeneralIcon icon="chevronDown" class="ml-2 transition-all transform" :class="{ 'rotate-180': detailsIsShown }" />
      </nc-button>
    </div>

    <div v-if="step === 1">
      <a-form ref="form" :model="syncSource" name="quick-import-airtable-form" layout="horizontal" class="m-0">
        <a-form-item v-bind="validateInfos['details.apiKey']">
          <div class="flex items-end">
            <label> {{ $t('labels.personalAccessToken') }} </label>
            <a
              href="https://docs.nocodb.com/bases/import-base-from-airtable#get-airtable-credentials"
              class="!text-brand prose-sm ml-auto"
              target="_blank"
              rel="noopener"
            >
              {{ $t('labels.whereToFind') }}
            </a>
          </div>
          <a-input-password
            v-model:value="syncSource.details.apiKey"
            placeholder="Enter your Airtable Personal Access Token"
            class="!rounded-lg mt-2 nc-input-api-key"
          >
            <template #iconRender="isVisible">
              <GeneralIcon :icon="!isVisible ? 'ncEye' : 'ncEyeOff'" />
            </template>
          </a-input-password>
        </a-form-item>

        <a-form-item v-bind="validateInfos['details.syncSourceUrlOrId']" class="!mt-4 !mb-4">
          <label> {{ `${$t('labels.sharedBase')} ID/URL` }} </label>
          <a-input
            v-model:value="syncSource.details.syncSourceUrlOrId"
            placeholder="Paste the Base URL or Base ID from Airtable"
            class="!rounded-lg !mt-2 nc-input-shared-base"
          />
        </a-form-item>

        <nc-button type="text" size="small" @click="collapseKey = !collapseKey ? 'advanced-settings' : ''">
          {{ $t('title.advancedSettings') }}
          <GeneralIcon
            icon="chevronDown"
            class="ml-2 !transition-all !transform"
            :class="{ '!rotate-180': collapseKey === 'advanced-settings' }"
          />
        </nc-button>

        <a-collapse v-model:active-key="collapseKey" ghost class="nc-import-collapse">
          <a-collapse-panel key="advanced-settings">
            <div class="mb-2">
              <a-checkbox v-model:checked="syncSource.details.options.syncData">{{ $t('labels.importData') }}</a-checkbox>
            </div>

            <div class="my-2">
              <a-checkbox v-model:checked="syncSource.details.options.syncViews">
                {{ $t('labels.importSecondaryViews') }}
              </a-checkbox>
            </div>

            <div class="my-2">
              <a-checkbox v-model:checked="syncSource.details.options.syncRollup">
                {{ $t('labels.importRollupColumns') }}
              </a-checkbox>
            </div>

            <div class="my-2">
              <a-checkbox v-model:checked="syncSource.details.options.syncLookup">
                {{ $t('labels.importLookupColumns') }}
              </a-checkbox>
            </div>

            <div class="my-2">
              <a-checkbox v-model:checked="syncSource.details.options.syncAttachment">
                {{ $t('labels.importAttachmentColumns') }}
              </a-checkbox>
            </div>

            <div class="my-2">
              <a-checkbox v-model:checked="syncSource.details.options.syncFormula" disabled>
                {{ $t('labels.importFormulaColumns') }}
              </a-checkbox>
            </div>
          </a-collapse-panel>
        </a-collapse>
      </a-form>
    </div>

    <div v-if="step === 2">
      <a-card
        v-if="detailsIsShown"
        ref="logRef"
        class="nc-import-logs-container"
        :body-style="{
          'backgroundColor': '#101015',
          'height': '200px',
          'overflow': 'auto',
          'borderRadius': '0.5rem',
          'padding': '16px !important',
          'scrollbar-color': 'var(--scrollbar-thumb) var(--scrollbar-track)',
          'scrollbar-width': 'thin',
          '--scrollbar-thumb': '#E7E7E9',
          '--scrollbar-track': 'transparent',
        }"
      >
        <a-button
          v-if="showGoToDashboardButton || goBack"
          class="!absolute z-1 right-2 bottom-2 opacity-75 hover:opacity-100 !rounded-md !w-8 !h-8"
          size="small"
          @click="downloadLogs('at-import-logs.txt')"
        >
          <nc-tooltip>
            <template #title>Download Logs</template>
            <component :is="iconMap.download" />
          </nc-tooltip>
        </a-button>

        <div v-for="({ msg, status }, i) in progress" :key="i" class="my-1">
          <div v-if="status === JobStatus.FAILED" class="flex items-start">
            <span class="text-red-400 ml-2 log-message">{{ msg }}</span>
          </div>
          <div v-else class="flex items-start">
            <span class="text-green-400 ml-2 log-message">{{ msg }}</span>
          </div>
        </div>
      </a-card>
      <div v-else class="flex items-start gap-2">
        <template v-if="isInProgress">
          <component :is="iconMap.loading" class="text-primary animate-spin mt-1" />
          <span>
            {{ progress?.[progress?.length - 1]?.msg ?? '---' }}
          </span>
        </template>
        <template v-else-if="progress?.[progress?.length - 1]?.status === JobStatus.FAILED">
          <a-alert class="!rounded-lg !bg-transparent !border-gray-200 !p-3 !w-full">
            >
            <template #message>
              <div class="flex flex-row items-center gap-2 mb-2">
                <GeneralIcon icon="ncAlertCircleFilled" class="text-red-500 w-4 h-4" />
                <span class="font-weight-700 text-[14px]">Import error</span>
              </div>
            </template>
            <template #description>
              <div class="text-gray-500 text-[13px] leading-5 ml-6">
                {{ progress?.[progress?.length - 1]?.msg ?? '---' }}
              </div>
            </template>
          </a-alert>
        </template>
        <div v-else class="flex items-start gap-3">
          <GeneralIcon icon="checkFill" class="text-white w-4 h-4 mt-0.75" />
          <span> {{ $t('msg.airtableImportSuccess') }} </span>
        </div>
      </div>

      <div v-if="!isInProgress" class="text-right mt-4">
        <nc-button v-if="progress?.[progress?.length - 1]?.status === JobStatus.FAILED" size="small" @click="step = 1">
          Retry import
        </nc-button>
        <nc-button v-else size="small" @click="dialogShow = false"> Go to base </nc-button>
      </div>
    </div>

    <template #footer>
      <div v-if="step === 1" class="flex justify-between mt-2">
        <nc-button
          key="back"
          type="text"
          size="small"
          @click="
            () => {
              dialogShow = false
              emit('back')
            }
          "
        >
          {{ $t('general.back') }}
        </nc-button>

        <nc-button
          key="submit"
          v-e="['c:sync-airtable:save-and-sync']"
          type="primary"
          class="nc-btn-airtable-import"
          size="small"
          :loading="isLoading"
          :disabled="disableImportButton"
          @click="saveAndSync"
        >
          {{ $t('activity.import') }} Base
        </nc-button>
      </div>
    </template>
  </a-modal>
</template>

<style lang="scss" scoped>
.nc-import-collapse :deep(.ant-collapse-header) {
  display: none !important;
}
</style>

<style>
.nc-modal-airtable-import .ant-modal-footer {
  @apply !border-none p-0;
}
.nc-modal-airtable-import .ant-collapse-content-box {
  padding-left: 6px;
}
</style>
