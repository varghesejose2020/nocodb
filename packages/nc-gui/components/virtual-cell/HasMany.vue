<script setup lang="ts">
import type { ColumnType } from 'nocodb-sdk'
import { isSystemColumn } from 'nocodb-sdk'
import type { Ref } from 'vue'

const column = inject(ColumnInj)!

const cellValue = inject(CellValueInj)!

const row = inject(RowInj)!

const reloadRowTrigger = inject(ReloadRowDataHookInj, createEventHook())

const isForm = inject(IsFormInj)

const readOnly = inject(ReadonlyInj, ref(false))

const isUnderLookup = inject(IsUnderLookupInj, ref(false))

const listItemsDlg = ref(false)

const childListDlg = ref(false)

const isOpen = ref(false)

const hideBackBtn = ref(false)

const { isUIAllowed } = useRoles()

const { state, isNew, removeLTARRef } = useSmartsheetRowStoreOrThrow()

const { relatedTableMeta, loadRelatedTableMeta, relatedTableDisplayValueProp, unlink } = useProvideLTARStore(
  column as Ref<Required<ColumnType>>,
  row,
  isNew,
  reloadRowTrigger.trigger,
)

await loadRelatedTableMeta()

const localCellValue = computed<any[]>(() => {
  if (cellValue?.value) {
    return cellValue?.value ?? []
  } else if (isNew.value) {
    return state?.value?.[column?.value.title as string] ?? []
  }
  return []
})

const cells = computed(() =>
  localCellValue.value.reduce((acc, curr) => {
    if (!relatedTableDisplayValueProp.value) return acc

    const value = curr[relatedTableDisplayValueProp.value]

    return [...acc, { value, item: curr }]
  }, []),
)

const unlinkRef = async (rec: Record<string, any>) => {
  if (isNew.value) {
    await removeLTARRef(rec, column.value)
  } else {
    await unlink(rec)
  }
}

const hasManyColumn = computed(
  () =>
    relatedTableMeta.value?.columns?.find((c: any) => c.title === relatedTableDisplayValueProp.value) as ColumnType | undefined,
)

const onAttachRecord = () => {
  childListDlg.value = false
  listItemsDlg.value = true
  hideBackBtn.value = false
}

const onAttachLinkedRecord = () => {
  listItemsDlg.value = false
  childListDlg.value = true
}

const openChildList = () => {
  if (isUnderLookup.value) return

  childListDlg.value = true
  listItemsDlg.value = false

  isOpen.value = true
  hideBackBtn.value = false
}

const openListDlg = () => {
  if (isUnderLookup.value) return

  listItemsDlg.value = true
  childListDlg.value = false
  isOpen.value = true
  hideBackBtn.value = true
}

useSelectedCellKeyupListener(inject(ActiveCellInj, ref(false)), (e: KeyboardEvent) => {
  switch (e.key) {
    case 'Enter':
      listItemsDlg.value = true
      e.stopPropagation()
      break
  }
})

watch([childListDlg, listItemsDlg], () => {
  isOpen.value = childListDlg.value || listItemsDlg.value
})

watch(
  isOpen,
  (next) => {
    if (!next) {
      listItemsDlg.value = false
      childListDlg.value = false
    }
  },
  { flush: 'post' },
)
</script>

<template>
  <LazyVirtualCellComponentsLinkRecordDropdown v-model:is-open="isOpen">
    <div class="nc-cell-field flex items-center gap-1 w-full chips-wrapper min-h-4">
      <div class="chips flex items-center img-container flex-1 hm-items flex-nowrap min-w-0 overflow-hidden">
        <template v-if="cells">
          <VirtualCellComponentsItemChip
            v-for="(cell, i) of cells"
            :key="i"
            :item="cell.item"
            :value="cell.value"
            :column="hasManyColumn"
            :show-unlink-button="true"
            @unlink="unlinkRef(cell.item)"
          />

          <span v-if="cellValue?.length === 10" class="caption pointer ml-1 grey--text" @click="openChildList"> more... </span>
        </template>
      </div>

      <div v-if="!isUnderLookup && !isSystemColumn(column)" class="flex justify-end gap-1 min-h-4 items-center">
        <GeneralIcon
          icon="maximize"
          class="select-none transform text-sm nc-action-icon text-gray-500/50 hover:text-gray-500 nc-arrow-expand h-3 w-3"
          @click.stop="openChildList"
        />

        <GeneralIcon
          v-if="(!readOnly && isUIAllowed('dataEdit')) || isForm"
          icon="plus"
          class="select-none text-sm nc-action-icon text-gray-500/50 hover:text-gray-500 nc-plus"
          @click.stop="openListDlg"
        />
      </div>
    </div>
    <template #overlay>
      <LazyVirtualCellComponentsUnLinkedItems
        v-if="listItemsDlg"
        v-model="listItemsDlg"
        :column="hasManyColumn"
        :hide-back-btn="hideBackBtn"
        @attach-linked-record="onAttachLinkedRecord"
      />

      <LazyVirtualCellComponentsLinkedItems
        v-if="childListDlg"
        v-model="childListDlg"
        :cell-value="localCellValue"
        :column="hasManyColumn"
        @attach-record="onAttachRecord"
      />
    </template>
  </LazyVirtualCellComponentsLinkRecordDropdown>
</template>

<style scoped>
.nc-action-icon {
  @apply hidden cursor-pointer;
}

.chips-wrapper:hover .nc-action-icon {
  @apply flex;
}
</style>
