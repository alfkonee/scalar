<script setup lang="ts">
import { useFileDialog } from '@/hooks'
import {
  convertPostmanToOpenApi,
  getOpenApiDocumentDetails,
  getPostmanDocumentDetails,
  isPostmanCollection,
  isUrl,
} from '@/libs'
import { useWorkspace } from '@/store'
import { useActiveEntities } from '@/store/active-entities'
import {
  ScalarButton,
  ScalarCodeBlock,
  ScalarIcon,
  ScalarTooltip,
  useLoadingState,
} from '@scalar/components'
import { useToasts } from '@scalar/use-toasts'
import { computed, ref, watch } from 'vue'

import CommandActionForm from './CommandActionForm.vue'
import CommandActionInput from './CommandActionInput.vue'
import WatchModeToggle from './WatchModeToggle.vue'

const emits = defineEmits<{
  (event: 'close'): void
  (event: 'back', e: KeyboardEvent): void
}>()

const { activeWorkspace } = useActiveEntities()
const { importSpecFile, importSpecFromUrl } = useWorkspace()
const { toast } = useToasts()
const loader = useLoadingState()

const inputContent = ref('')
const watchMode = ref(true)

const documentDetails = computed(() => {
  if (isPostmanCollection(inputContent.value)) {
    return getPostmanDocumentDetails(inputContent.value)
  } else {
    return getOpenApiDocumentDetails(inputContent.value)
  }
})

const documentType = computed(() =>
  documentDetails.value ? documentDetails.value.type : 'json',
)

const isInputUrl = computed(() => isUrl(inputContent.value))
const isInputDocument = computed(() => !!documentDetails.value)

const { open: openSpecFileDialog } = useFileDialog({
  onChange: async (files) => {
    const file = files?.[0]
    if (file) {
      const reader = new FileReader()
      reader.onload = async (e) => {
        const text = e.target?.result as string
        try {
          if (isPostmanCollection(text)) {
            await importSpecFile(
              await convertPostmanToOpenApi(text),
              activeWorkspace.value?.uid ?? '',
            )
          } else {
            await importSpecFile(text, activeWorkspace.value?.uid ?? '')
          }
          toast('Import successful', 'info')
          emits('close')
        } catch (error) {
          console.error(error)
          const errorMessage = (error as Error)?.message || 'Unknown error'
          toast(`Import failed: ${errorMessage}`, 'error')
        }
      }
      reader.readAsText(file)
    }
  },
  multiple: false,
  accept: '.json,.yaml,.yml',
})

// Enable watch mode if the input is a URL
watch(isInputUrl, (newVal) => {
  if (!newVal) watchMode.value = false
})

// Disable watch mode if the input is not a URL
watch(inputContent, (newVal) => {
  if (!isUrl(newVal)) watchMode.value = false
})

async function importCollection() {
  if (!inputContent.value || loader.isLoading) return

  loader.startLoading()
  try {
    if (isInputUrl.value) {
      const [error] = await importSpecFromUrl(
        inputContent.value,
        activeWorkspace.value?.uid ?? '',
        {
          proxyUrl: activeWorkspace.value?.proxyUrl,
          watchMode: watchMode.value,
        },
      )

      if (error) {
        toast(
          'There was a possible CORS error while importing your spec, please make sure this server is allowed in the CORS policy of your OpenAPI document.',
          'error',
          { timeout: 5_000 },
        )
        loader.invalidate(2000, true)
        return
      }
    } else if (isInputDocument.value) {
      if (isPostmanCollection(inputContent.value)) {
        await importSpecFile(
          await convertPostmanToOpenApi(inputContent.value),
          activeWorkspace.value?.uid ?? '',
        )
        toast('Successfully converted Postman collection', 'info')
      } else {
        await importSpecFile(
          inputContent.value,
          activeWorkspace.value?.uid ?? '',
        )
      }
    } else {
      toast('Import failed: Invalid URL or OpenAPI document', 'error')
      loader.invalidate(2000, true)
      return
    }

    loader.clear()

    emits('close')
    toast('Import successful', 'info')
  } catch (error) {
    console.error('[importCollection]', error)
    const errorMessage = (error as Error)?.message || 'Unknown error'
    loader.invalidate(2000, true)
    toast(`Import failed: ${errorMessage}`, 'error')
  }
}
</script>
<template>
  <CommandActionForm
    :disabled="!inputContent.trim()"
    :loading="loader"
    @submit="importCollection">
    <template v-if="!documentDetails || isUrl(inputContent)">
      <CommandActionInput
        v-model="inputContent"
        placeholder="OpenAPI/Swagger/Postman URL or document"
        @onDelete="emits('back', $event)" />
    </template>
    <template v-else>
      <!-- OpenAPI document preview -->
      <div class="flex justify-between">
        <div class="pl-8 text-xs min-h-8 py-2 text-c-2">Preview</div>
        <ScalarButton
          class="ml-auto p-2 max-h-8 gap-1.5 text-xs hover:bg-b-2 relative"
          variant="ghost"
          @click="inputContent = ''">
          Clear
        </ScalarButton>
      </div>
      <ScalarCodeBlock
        v-if="documentDetails && !isUrl(inputContent)"
        class="border max-h-[40dvh] mt-1 bg-b-2 rounded [--scalar-small:--scalar-font-size-4]"
        :content="inputContent"
        :copy="false"
        :lang="documentType" />
    </template>
    <template #options>
      <div class="flex flex-row items-center justify-between gap-3 w-full">
        <!-- Upload -->
        <ScalarButton
          class="p-2 max-h-8 gap-1.5 text-xs hover:bg-b-2 relative"
          variant="outlined"
          @click="openSpecFileDialog">
          JSON, or YAML File
          <ScalarIcon
            class="text-c-3"
            icon="UploadSimple"
            size="md" />
        </ScalarButton>

        <!-- Watch -->
        <ScalarTooltip
          as="div"
          class="z-[10001]"
          side="bottom"
          :sideOffset="5">
          <template #trigger>
            <WatchModeToggle
              v-model="watchMode"
              :disabled="!isInputUrl" />
          </template>
          <template #content>
            <div
              class="grid gap-1.5 pointer-events-none max-w-[320px] w-content shadow-lg rounded bg-b-1 z-100 p-2 text-xxs leading-5 z-10 text-c-1">
              <div class="flex items-center text-c-2">
                <span
                  v-if="isInputUrl"
                  class="text-pretty">
                  Automatically updates the API client when the OpenAPI URL
                  content changes, ensuring your client remains up-to-date.
                </span>
                <span
                  v-else
                  class="text-pretty">
                  Watch Mode is only available for URL imports. It automatically
                  updates the API client when the OpenAPI URL content changes.
                </span>
              </div>
            </div>
          </template>
        </ScalarTooltip>
      </div>
    </template>
    <template #submit>
      Import
      <template v-if="isInputUrl"> from URL </template>
      <template v-else-if="documentDetails && documentType">
        <template v-if="documentDetails.title">
          "{{ documentDetails.title }}"
        </template>
        <template v-else>{{ documentDetails.version }}</template>
      </template>
      <template v-else>Collection</template>
    </template>
  </CommandActionForm>
</template>
