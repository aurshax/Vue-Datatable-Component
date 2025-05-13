<script setup>
import {mergeDeepRight, prop, propOr} from "ramda";
import {extraConfig} from "@themeConfig";
import {isEmpty, isObject} from "@core/utils";
import ScannerField from "@/views/forms/ScannerField.vue";
import {ASC, DESC} from "@/constants/SortTypes";

const props = defineProps({
  headers: {
    type: Array,
    required: true,
    default: () => []
  },
  items: {
    type: Array,
    required: true,
    default: () => []
  },
  isListLoading: {
    type: Boolean,
    required: true,
    default: false
  },
  searchQuery: {
    type: String,
    required: false,
    default: ''
  },
  totalItems: {
    type: Number,
    required: true,
    default: 0,
  },
  rowsPerPage: {
    type: Number,
    required: false,
    default: extraConfig.table.rowsPerPage,
  },
  tableOptions: {
    type: Object,
    required: false,
    default: () => {}
  },
  selectedRows: {
    type: Array,
    required: false,
    default: () => []
  },
  hideFooter: {
    type: Boolean,
    required: false,
    default: false,
  },
  showSearch: {
    type: Boolean,
    required: false,
    default: true,
  },
  showPreHeader: {
    type: Boolean,
    required: false,
    default: true,
  },
  selectable: {
    type: Boolean,
    required: false,
    default: true,
  },
  customizableColumns: {
    type: Boolean,
    required: false,
    default: false,
  },
  hideTableHeader: {
    type: Boolean,
    required: false,
    default: false,
  },
  activeRow: {
    type: [Number, null],
    required: false,
    default: null,
  },
  sortByColumn: {
    type: String,
    required: false,
    default: null
  },
  sortOrder: {
    type: String,
    required: false,
    validator: (value) => [ASC, DESC, null].includes(value),
  },
})

const emit = defineEmits([
  'change:pageSize',
  'change:currentPage',
  'update:search',
  'click:customize',
  'update:sort',
])

const DEFAULT_COL_WIDTH = 200;

const rowsPerPage = ref(props.rowsPerPage)
const selectedRows = ref(props.selectedRows)
const selectAllItems = ref(false)
const currentPage = ref(1)
const tableHeight = ref(200)
const vDataTable = ref(null)
const sortByColumn = ref(null)
const sortOrder = ref(null)
const searchQuery  = ref(null);

watch(() => prop('sortByColumn', props), (val) => {
  sortByColumn.value = !isEmpty(val) ? val : null;
}, {immediate: true})

watch(() => prop('sortOrder', props), (val) => {
  sortOrder.value = !isEmpty(val) ? val : null;
}, {immediate: true})

watch(() => props.searchQuery, (val) => {
  searchQuery.value = val;
}, {immediate: true})

const onChangeRowsPerPage = (size) => {
  emit('change:pageSize', size)
}
const onChangePage = (page) => {
  selectedRows.value = []
  emit('change:currentPage', page)
}

const onSearch = (query) => {
  emit('update:search', query)
}

const rowSize = computed(() => {
  let cols = props.headers.length || 1 + 1;

  if(props.selectable) {
    cols++;
  }

  if(props.customizableColumns) {
    cols++;
  }

  return cols
})

const totalPageCount = computed(() => {
  return Math.ceil(props.totalItems / rowsPerPage.value)
})

const selectUnselectAll = () => {
  selectAllItems.value = !selectAllItems.value
  if (selectAllItems.value) {
    props.items.forEach(item => {
      if (!selectedRows.value.includes(`check${ item.id }`))
        selectedRows.value.push(`check${ item.id }`)
    })
  } else {
    selectedRows.value = []
  }
}

const addRemoveIndividualCheckbox = checkID => {
  if (selectedRows.value.includes(checkID)) {
    const index = selectedRows.value.indexOf(checkID)
    selectedRows.value.splice(index, 1)
    selectAllItems.value = false
  } else {
    selectedRows.value.push(checkID)
    selectAllItems.value = true
  }
}

watchEffect(() => {
  if (currentPage.value > totalPageCount.value && totalPageCount.value > 0)
    currentPage.value = totalPageCount.value
})

const paginationData = computed(() => {
  const firstIndex = props.items.length ? (currentPage.value - 1) * rowsPerPage.value + 1 : 0
  const lastIndex = props.items.length + (currentPage.value - 1) * rowsPerPage.value

  return `${firstIndex}-${lastIndex} of ${props.totalItems}`
})

const tableOptions = computed(() => {
  if(isObject(props.tableOptions)) {
    return mergeDeepRight(extraConfig.table, props.tableOptions)
  }
  return extraConfig.table
})

const onResize = () => {

  const footerHeight = props.hideFooter ? 5 : 68;

  tableHeight.value = window.innerHeight - vDataTable.value?.getBoundingClientRect()?.y - footerHeight
  // console.log(vDataTable.value?.getBoundingClientRect());
  // console.log(vDataTable);
  // console.log(vDataTable.value.getBoundingClientRect());
}

// Table Header TH Class Generator
const thClass = (header) => {
  const className = [];

  if (header.align) {
    className.push(`text-${header.align}`)
  }

  if(header.sortable) {
    className.push('cursor-pointer')
  }

  return className;
}

// Table Body TD Class Generator
const tdClass = (item) => {
  const className = [];

  if (item.align) {
    className.push(`text-${item.align}`)
  }

  return className;
}

const onChangeSortedColumn = (header) => {
  if(!header.sortable) {
    return;
  }

  emit('update:sort', header.value, sortOrder.value === ASC ? DESC : ASC )
}

const colWidth = (item) => {
  let width = DEFAULT_COL_WIDTH;

  if(item?.width) {
    width = item.width
  }

  return `width: ${width}px`
}
</script>

<template>
  <div>
    <div class="d-flex flex-column v-data-table-header-top"><slot name="header-top"></slot></div>
    <VCard v-resize="onResize" class="v-data-table-container" variant="flat">
      <VCardText v-if="props.showPreHeader" class="v-data-table-header">
        <div class="d-flex v-data-table-header-middle">
          <div v-if="props.showSearch" class="flex-grow-1 table-list-search">
            <ScannerField
                :model-value="searchQuery"
                :placeholder="$t('Search')"
                density="compact"
                @update:model-value="onSearch"
            >
            </ScannerField>
          </div>
          <slot name="header-middle"></slot>
        </div>
        <div class="d-flex v-data-table-header-bottom"><slot name="header-bottom"></slot></div>
      </VCardText>

      <div ref="vDataTable"></div>
      <VTable :fixed-header="true" :height="tableHeight" class="v-data-table">
        <thead>
        <tr>
          <th v-if="props.customizableColumns" class="col-customize-cols-item">
            <IconBtn class="ml-2 mt-1" size="small" @click.stop="$emit('click:customize')">
              <VIcon color="primary" size="large" icon="mdi-window-shutter-cog" />
            </IconBtn>
          </th>
          <th v-if="props.selectable" scope="col" class="col-checkbox-item">
            <div class="mb-n2">
              <VCheckbox
                  :model-value="selectAllItems"
                  :indeterminate="(items.length !== selectedRows.length) && !!selectedRows.length"
                  @click="selectUnselectAll"
              />
            </div>
          </th>
          <th
              v-if="propOr(false, 'hideTableHeader', props) === false"
              v-for="header in headers" :key="header.name"
              @click="onChangeSortedColumn(header)"
              class="v-data-table-header" :class="thClass(header)">
            <slot :name="`header-${header.value}`" :header="header">
              <div class="flex align-items-center" :class="header.value === 'actions' ? 'float-right' : ''" :style="colWidth(header)">
                <slot :name="`header-${header.value}-text`">{{ $t(header.text) }}</slot>

                <VIcon
                    v-if="sortByColumn === header.value"
                    size="16"
                    :icon="sortOrder === ASC ? 'mdi-menu-up' : 'mdi-menu-down'"
                />
              </div>
            </slot>
          </th>
        </tr>
        <tr v-if="isListLoading">
          <td :colspan="rowSize" class="v-data-table-loader">
            <VProgressLinear
                indeterminate
                color="primary"
            ></VProgressLinear>
          </td>
        </tr>
        </thead>
        <tbody v-if="!isListLoading && items.length > 0">
        <tr v-for="(dataItem, ind) in items" :key="dataItem.id" :class="activeRow && Number(activeRow) === Number(dataItem.id) ? 'active-row' : ''">
          <td v-if="props.customizableColumns" class="col-customize-cols-item"></td>

          <td v-if="props.selectable" class="col-checkbox-item">
            <div class="mb-n2">
              <VCheckbox
                  :id="`check${dataItem.id}`"
                  :model-value="selectedRows.includes(`check${dataItem.id}`)"
                  @click="addRemoveIndividualCheckbox(`check${dataItem.id}`)"
              />
            </div>
          </td>

          <td v-for="column in headers" :key="column.value" :class="tdClass(column)">
            <slot :name="`column-${column.value}`" :column="column" :item="dataItem" :index="ind">{{ dataItem[column.value] }}</slot>
          </td>
        </tr>
        </tbody>
        <tfoot v-show="items.length === 0 || isListLoading">
        <tr>
          <td
              :colspan="rowSize"
              class="text-center text-body-1"
          >
            <div v-if="isListLoading" class="v-data-table-loading-row">
              <slot name="loading">{{ $t('Loading...') }}</slot>
            </div>
            <div v-else class="v-data-table-no-data-row">
              <slot name="no-data-available">{{ $t('No data available') }}</slot>
            </div>
          </td>
        </tr>
        </tfoot>
      </VTable>

      <!-- SECTION Pagination -->
      <VCardText v-if="!props.hideFooter" class="v-data-table-footer d-flex flex-wrap justify-end gap-4 pa-2">
        <!-- 👉 Rows per page -->
        <div class="v-data-table-footer-pagination d-flex align-center me-3">
          <span class="footer-rows-per-page-text text-no-wrap me-3">
            <slot name="footer-rows-per-page">{{ $t('Rows per page:') }}</slot>
          </span>
          <VSelect
              v-model="rowsPerPage"
              density="compact"
              variant="outlined"
              class="v-data-table-footer-pagination-page-size-select"
              :items="tableOptions.paginationOptions"
              @update:model-value="onChangeRowsPerPage"
          />
        </div>

        <!-- 👉 Pagination and pagination meta -->
        <div class="d-flex align-center">
          <h6 class="text-sm font-weight-regular footer-pagination-text">
            {{ paginationData }}
          </h6>

          <VPagination
              v-model="currentPage"
              :total-visible="1"
              :length="totalPageCount"
              size="small"
              @next="onChangePage"
              @prev="onChangePage"
          />
        </div>
      </VCardText>
    </VCard>
  </div>
</template>


<style lang="scss" scoped>
.v-data-table-container {
  display: flex;
  flex-direction: column;
  height: 100%;

  & > .v-card-text {
    flex: none;
  }
}
.v-data-table {
  flex-grow: 1;

  tr, td {
    min-width: 130px;
  }

  tbody {
    tr.active-row, tr:hover {
      background-color: rgba(var(--v-theme-on-background), 0.07)
    }
  }

  .col-checkbox-item {
    min-width: 40px;
  }

  .col-customize-cols-item {
    padding: 0;
    width: 10px !important;
    min-width: 10px !important;
  }

  &-loader {
    height: 4px !important;
    padding: 0 !important;
  }

  .col-checkbox-item {
    padding: 0;
    width: 40px;
  }

  &-loading-row {
    text-align: center;
  }

  &-no-data-row {
    text-align: center;
  }

  &-header {
    padding: 10px;
    border-bottom: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));
  }

  &-footer {
    border-top: 1px solid rgba(var(--v-border-color), var(--v-border-opacity));

    &-pagination {
      width: 245px;

      &-page-size-select {
        padding-top: 0;
      }
    }
  }
}
</style>

<style lang="scss">
  .layout-page-content {
    margin-top: -10px;
    padding-top: 3px;
    padding-bottom: 2px;
  }
</style>
