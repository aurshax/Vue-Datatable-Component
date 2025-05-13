<script setup>
import {useCustomersList} from "@/hooks/useCustomersList";
import {prop, propOr} from "ramda";
import {useRoute} from "vue-router";
import {injectBodyClass} from "@/utils/ui"
import CustomerView from "@/views/pages/customers/CustomerView.vue";
import {useSnackbar} from "vue3-snackbar";
import {useCustomersItem} from "@/hooks/useCustomerItem";
import {checkCustomersPermission} from "@/utils/permissions";
import {useFilter} from "@/hooks/useFilter";
import {useAvailableListingStore} from "@/store/useAvailableListingStore";
import ListingCustomViewsWrapper from "@/views/layouts/ListingCustomViewsWrapper.vue";
import {CUSTOMER} from "@/constants/FormTypes";
import SelectedListingCustomizeColumnsDialog from "@/views/pages/shared/SelectedListingCustomizeColumnsDialog.vue";
import {DESC} from "@/constants/SortTypes";
import TableCustomFieldsComponent from "@/views/table/TableCustomFieldsComponent.vue";
import {STATUS} from "@/constants/mapping/contacts";
import {formatBoolActiveStatus} from "@/constants/ActiveTypes";
import TableStaticFieldsComponent from "@/views/table/TableStaticFieldsComponent.vue";
import ability from "@/plugins/casl/ability";
import {toggleBodyHorizontalSpace} from "@/utils/layoutUtils";

const DashboardTitleWrapper = defineAsyncComponent(() => import('@/views/layouts/DashboardTitleWrapper.vue'))
const ConfirmationDialog = defineAsyncComponent(() => import('@/views/dialog/ConfirmationDialog.vue'))

const {
  filters,
  customers,
  tableHeaders,
  detailedView,
  isListLoading,
  totalListCount,
  sortableColumns,

  onCloseView,
  fetchCustomerList
} = useCustomersList();

const {isSubmitting, onDeleteCustomer} = useCustomersItem();

const {onFilterValueChange, onFilterValuesChange} = useFilter()
const availableListingStore = useAvailableListingStore()

const {t} = useI18n({ useScope: 'global' });
const route = useRoute();
const router = useRouter();
const snackbar = useSnackbar();

const deleteItemId = ref(null);
const showDeleteDialog = ref(false);
const showCustomizeColumnsDialog = ref(false);

toggleBodyHorizontalSpace(true)

watch(detailedView, (val) => {
  injectBodyClass('document-full-height', !!val)
}, { immediate: true })

const handleDeleteConfirmation = (itemId) => {
  deleteItemId.value = itemId;
  showDeleteDialog.value = true;
}

const onCloseDialog = () => {
  deleteItemId.value = null;
  showDeleteDialog.value = false;
  showCustomizeColumnsDialog.value = false;
}

const onDeleteItem = (confirmed) => {
  const id = deleteItemId.value

  if(confirmed && id > 0) {
    onDeleteCustomer(id).then(res => {
      snackbar.add({
        type: 'success',
        text: t('Successfully deleted')
      })

      onCloseDialog();
      fetchCustomerList();
    })
  }
}

</script>
<template>
  <div class="data-table-list h-full">
    <DashboardTitleWrapper>
      <template #action>
        <ListingCustomViewsWrapper
            :default-title="$t('Customers')"
            :form="CUSTOMER"
            @update:view="val => availableListingStore.setSelectedAvailableListing(val, CUSTOMER)"
            @edit:view="val => router.push({'name': 'custom-views', query:{ type: CUSTOMER, id: prop('id', val) }})"
        >
          <template #actions>
            <RouterLink
                :to="{name:'custom-views', query:{ type: CUSTOMER }}"
                class="d-block text-center w-100 text-body-2 text-primary font-weight-bold cursor-pointer text-decoration-none"
            >
              <VIcon
                  size="14"
                  icon="mdi-add"
                  class="mr-1"
              /> {{ $t('New Custom View') }}
            </RouterLink>
          </template>
        </ListingCustomViewsWrapper>
      </template>
    </DashboardTitleWrapper>

    <ConfirmationDialog
        v-model:isDialogVisible="showDeleteDialog"
        :confirmation-msg="$t('Are you sure you want to delete the Customer?')"
        @confirm="(val) => onDeleteItem(val)"
    />

    <SelectedListingCustomizeColumnsDialog
        v-if="showCustomizeColumnsDialog"
        v-model:isDialogVisible="showCustomizeColumnsDialog"
        :form="CUSTOMER"
        @on-submit-success="onCloseDialog"
        @on-dismiss="onCloseDialog"
    />

    <div class="content-table" :class="detailedView ? 'detailed' :'expanded'">
      <DataTable
          :headers="tableHeaders"
          :items="customers"
          :is-list-loading="isListLoading || isSubmitting"
          :total-items="totalListCount"
          :search-query="propOr('', 'searchKey', filters)"
          :rows-per-page="prop('limit', filters)"
          :selectable="false"
          :customizable-columns="!detailedView"
          :hide-table-header="detailedView"
          :sort-by-column="prop('sortBy', filters)"
          :sort-order="prop('sortAs', filters)"
          :active-row="prop('customerId', route.query)"
          @change:pageSize="limit => onFilterValueChange('limit', limit)"
          @change:currentPage="page => onFilterValueChange('page', page)"
          @update:search="query => onFilterValueChange('searchKey', query)"
          @click:customize="showCustomizeColumnsDialog = true"
          @update:sort="(col, order) => onFilterValuesChange({ sortBy: col, sortAs: order })"
      >
        <template #header-middle>
          <div class="d-flex ml-2">
            <VBtn
                v-if="checkCustomersPermission('create')"
                prepend-icon="mdi-plus"
                :disabled="isListLoading"
                :to="{ name: 'customers-create' }"
            >
              {{ $t(detailedView ? 'New' :'Add Customer') }}
            </VBtn>

            <VBtn
                icon
                class="mr-2"
                variant="text"
                color="default">
              <VTooltip
                  activator="parent"
                  location="top"
              >
                {{ $t('Sort By') }}
              </VTooltip>

              <VIcon
                  :size="24"
                  icon="mdi-sort-alphabetical-ascending"
              />

              <VMenu activator="parent">
                <VList density="compact" min-width="100px">
                  <VListItem v-for="(column, ind) in sortableColumns" :key="ind" @click="() => { onFilterValuesChange({ sortBy: propOr('', 'code', column), sortAs: DESC }) }">
                    <VListItemTitle>{{ propOr('', 'name', column) }}</VListItemTitle>
                  </VListItem>
                </VList>
              </VMenu>
            </VBtn>

            <VBtn
                icon
                variant="text"
                color="default">
              <VIcon
                  :size="24"
                  icon="mdi-dots-vertical"
              />

              <VMenu activator="parent">
                <VList density="compact" min-width="100px">
                  <VListItem v-if="ability.can('update', 'Settings Preferences')" :to="{ name: 'settings-preferences-contact' }">
                    <template #prepend>
                      <VIcon
                          color="primary"
                          size="20"
                          icon="mdi-cog"
                      />
                    </template>
                    <VListItemTitle>{{ $t('Preferences') }}</VListItemTitle>
                  </VListItem>
                  <VDivider />
                  <VListItem @click="() => { fetchCustomerList() }">
                    <template #prepend>
                      <VIcon
                          color="primary"
                          size="20"
                          icon="mdi-refresh"
                      />
                    </template>
                    <VListItemTitle>{{ $t('Refresh List') }}</VListItemTitle>
                  </VListItem>

                </VList>
              </VMenu>
            </VBtn>
          </div>
        </template>

        <template v-for="(header, idx) in tableHeaders" :key="idx" v-slot:[`header-${header.value}-text`]>
          {{ propOr(false, 'customField', header) ? header.text : $t(header.text) }}
        </template>

        <template v-for="(header, idx) in tableHeaders" :key="idx" v-slot:[`column-${header.value}`]="{item, column}">
          <template v-if="prop('value', header) === 'actions'">
            <VBtn
                icon
                variant="plain"
                color="default"
                size="x-small"
            >
              <VIcon
                  :size="24"
                  icon="mdi-dots-vertical"
              />

              <VMenu activator="parent">
                <VList>
                  <VListItem
                      v-if="checkCustomersPermission('update')"
                      :to="{name: 'customers-edit-id', params: { id: item.id }}"
                      exact-path value="edit">
                    <template #prepend>
                      <VIcon
                          :size="24"
                          class="mr-2"
                          icon="mdi-pencil"
                      />
                    </template>

                    <VListItemTitle>{{ $t('Edit') }}</VListItemTitle>
                  </VListItem>

                  <VListItem
                      v-if="checkCustomersPermission('delete')"
                      @click.prevent="handleDeleteConfirmation(item.id)" value="delete">
                    <template #prepend>
                      <VIcon
                          :size="24"
                          class="mr-2"
                          icon="mdi-delete-outline"
                      />
                    </template>

                    <VListItemTitle>{{ $t('Delete') }}</VListItemTitle>
                  </VListItem>

                </VList>
              </VMenu>
            </VBtn>
          </template>

          <template v-else-if="prop('customField', header) === true">
            <TableCustomFieldsComponent :item="item" :name="prop('value', header)" />
          </template>

          <template v-else>
            <template v-if="prop('value', header) === 'NAME'">
              <div v-if="detailedView" class="detailed-list-row">
                <router-link
                    :to="{name:'customers',query:{ customerId: item.id }}"
                    class="text--primary font-weight-semibold text-truncate cursor-pointer text-decoration-none"
                >
                  {{ propOr('', 'name', item) }}
                </router-link>
              </div>

              <div class="h-100 d-flex align-center" v-else>
                <router-link
                    :to="{name:'customers',query:{ customerId: item.id }}"
                    class="text--primary font-weight-semibold text-truncate cursor-pointer text-decoration-none"
                >
                  {{ propOr('', 'name', item) }}
                </router-link>
              </div>
            </template>

            <template v-else-if="prop('value', header) === STATUS">
              <VChip
                  size="small"
                  :color="(item?.active ? 'success' : 'warning')"
                  variant="elevated"
              >
                {{ $t(formatBoolActiveStatus(item?.active)) }}
              </VChip>
            </template>

            <template v-else>
              <TableStaticFieldsComponent
                  :form-type="CUSTOMER"
                  :item="item"
                  :header="header"
              />
            </template>
          </template>
        </template>

      </DataTable>
    </div>

    <div v-if="detailedView" class="content-detail-view">

      <CustomerView @on-close-view="onCloseView" :customer-id="$route.query.customerId" />

      <VIcon
          class="icon-close"
          icon="mdi-close"
          size="23"
          @click="$router.push({query: {customerId: undefined}})"
      >
      </VIcon>
    </div>
  </div>
</template>

<style lang="scss">
@import "@/assets/scss/global.scss";
@import "@/assets/scss/data-hybrid-list-details-view.scss";
</style>

<route>
{
  name: "customers",
  meta: {
    layout: "default",
    action: "read",
    subject: "Customers",
    navActiveLink: "customers"
  }
}
</route>
