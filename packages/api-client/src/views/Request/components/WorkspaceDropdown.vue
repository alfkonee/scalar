<script setup lang="ts">
import DeleteSidebarListElement from '@/components/Sidebar/Actions/DeleteSidebarListElement.vue'
import EditSidebarListElement from '@/components/Sidebar/Actions/EditSidebarListElement.vue'
import { useWorkspace } from '@/store'
import { useActiveEntities } from '@/store/active-entities'
import {
  ScalarButton,
  ScalarDropdown,
  ScalarDropdownDivider,
  ScalarDropdownItem,
  ScalarIcon,
  ScalarListboxCheckbox,
  ScalarModal,
  ScalarTooltip,
  useModal,
} from '@scalar/components'
import { computed, ref } from 'vue'
import { useRouter } from 'vue-router'

const { activeWorkspace } = useActiveEntities()
const { workspaces, workspaceMutators, events } = useWorkspace()
const { push } = useRouter()

const updateSelected = (uid: string) => {
  if (uid === activeWorkspace.value?.uid) return

  push({
    name: 'workspace',
    params: {
      workspace: uid,
    },
  })
}

const isLastWorkspace = computed(() => Object.keys(workspaces).length === 1)

const createNewWorkspace = () =>
  events.commandPalette.emit({ commandName: 'Create Workspace' })

const tempName = ref('')
const tempUid = ref('')
const editModal = useModal()
const deleteModal = useModal()

const openRenameModal = (uid: string) => {
  const workspace = workspaces[uid]
  if (!workspace) return

  tempName.value = workspace.name
  tempUid.value = uid
  editModal.show()
}

const handleWorkspaceEdit = (name: string) => {
  if (!name.trim()) {
    return
  }

  workspaceMutators.edit(tempUid.value, 'name', name.trim())
  editModal.hide()
}

const openDeleteModal = (uid: string) => {
  const workspace = workspaces[uid]
  if (!workspace) return

  tempName.value = workspace.name
  tempUid.value = uid
  deleteModal.show()
}

const deleteWorkspace = async () => {
  if (!isLastWorkspace.value) {
    const deletedActiveWorkspace = activeWorkspace.value?.uid === tempUid.value
    const currentWorkspaces = { ...workspaces }
    delete currentWorkspaces[tempUid.value]

    workspaceMutators.delete(tempUid.value)

    // Switch to another workspace if the active one is deleted
    if (deletedActiveWorkspace) {
      const newWorkspaceUid = Object.keys(currentWorkspaces)[0]

      await push({
        name: 'workspace',
        params: {
          workspace: newWorkspaceUid,
        },
      })
    }
  }

  deleteModal.hide()
}
</script>

<template>
  <div>
    <div class="flex items-center text-sm w-[inherit]">
      <ScalarDropdown>
        <ScalarButton
          class="font-normal h-full justify-start line-clamp-1 py-1.5 px-1.5 text-c-1 hover:bg-b-2 w-fit"
          fullWidth
          variant="ghost">
          <div class="font-bold m-0 flex gap-1.5 items-center">
            <h2 class="line-clamp-1 text-left">
              {{ activeWorkspace?.name }}
            </h2>
          </div>
        </ScalarButton>

        <!-- Workspace list -->
        <template #items>
          <ScalarDropdownItem
            v-for="(workspace, uid) in workspaces"
            :key="uid"
            class="flex gap-1.5 group/item items-center whitespace-nowrap text-ellipsis overflow-hidden w-full"
            @click.stop="updateSelected(uid)">
            <ScalarListboxCheckbox :selected="activeWorkspace?.uid === uid" />
            <span class="text-ellipsis overflow-hidden">{{
              workspace.name
            }}</span>
            <ScalarDropdown
              placement="right-start"
              teleport>
              <ScalarButton
                class="px-0.5 py-0 hover:bg-b-3 group-hover/item:flex aspect-square ml-auto -mr-1 h-fit"
                size="sm"
                type="button"
                variant="ghost">
                <ScalarIcon
                  icon="Ellipses"
                  size="sm" />
              </ScalarButton>
              <template #items>
                <ScalarDropdownItem
                  class="flex gap-2"
                  @mousedown="openRenameModal(uid)"
                  @touchend.prevent="openRenameModal(uid)">
                  <ScalarIcon
                    class="inline-flex"
                    icon="Edit"
                    size="md"
                    thickness="1.5" />
                  <span>Rename</span>
                </ScalarDropdownItem>
                <ScalarTooltip
                  v-if="isLastWorkspace"
                  class="z-overlay"
                  side="bottom">
                  <template #trigger>
                    <ScalarDropdownItem
                      class="flex gap-2 w-full"
                      disabled
                      @mousedown.prevent
                      @touchend.prevent>
                      <ScalarIcon
                        class="inline-flex"
                        icon="Delete"
                        size="md"
                        thickness="1.5" />
                      <span>Delete</span>
                    </ScalarDropdownItem>
                  </template>
                  <template #content>
                    <div
                      class="grid gap-1.5 pointer-events-none min-w-48 w-content shadow-lg rounded bg-b-1 z-context p-2 text-xxs leading-5 z-10 text-c-1">
                      <div class="flex items-center text-c-2">
                        <span>Only workspace cannot be deleted.</span>
                      </div>
                    </div>
                  </template>
                </ScalarTooltip>
                <ScalarDropdownItem
                  v-else
                  class="flex !gap-2"
                  @mousedown.prevent="openDeleteModal(uid)"
                  @touchend.prevent="openDeleteModal(uid)">
                  <ScalarIcon
                    class="inline-flex"
                    icon="Delete"
                    size="sm"
                    thickness="1.5" />
                  <span>Delete</span>
                </ScalarDropdownItem>
              </template>
            </ScalarDropdown>
          </ScalarDropdownItem>
          <ScalarDropdownDivider />

          <!-- Add new workspace -->
          <ScalarDropdownItem
            class="flex items-center gap-1.5"
            @click="createNewWorkspace">
            <div class="flex items-center justify-center h-4 w-4">
              <ScalarIcon
                icon="Add"
                size="sm" />
            </div>
            <span>Create Workspace</span>
          </ScalarDropdownItem>
        </template>
      </ScalarDropdown>
    </div>
    <ScalarModal
      :size="'xxs'"
      :state="deleteModal"
      title="Delete workspace">
      <DeleteSidebarListElement
        :variableName="tempName"
        warningMessage="This cannot be undone. You’re about to delete the workspace and everything inside it."
        @close="deleteModal.hide()"
        @delete="deleteWorkspace" />
    </ScalarModal>
    <ScalarModal
      :size="'xxs'"
      :state="editModal"
      title="Rename Workspace">
      <EditSidebarListElement
        :name="tempName"
        @close="editModal.hide()"
        @edit="handleWorkspaceEdit" />
    </ScalarModal>
  </div>
</template>
