<template>
  <div class="rounded-md bg-gray-800 p-4">
    <div class="mb-4 flex justify-between gap-4">
      <!-- filters -->
      <n-space align="center">
        <n-input @keyup.enter="loadBansData" type="text" v-model:value="banQuery.player" placeholder="Player" />

        <n-input @keyup.enter="loadBansData" type="text" v-model:value="banQuery.bannedBy" placeholder="Banned By" />

        <n-select
          style="width: 8rem"
          @update-value="nextTick(loadBansData)"
          v-model:value="banQuery.reason"
          :options="banReasonOptions"
          placeholder="Ban Reason"
        />
      </n-space>

      <div class="flex gap-4">
        <n-button @click="loadBansData"> APPLY FILTER</n-button>
        <n-button @click="clearFilter"> CLEAR </n-button>
      </div>
    </div>

    <!-- servers table -->
    <div class="mb-4">
      <n-data-table
        :columns="columns"
        :data="data"
        :loading="loading"
        :pagination="pagination"
        :row-key="rowKey"
        size="small"
        @update:sorter="handleSorterChange"
      />
    </div>

    <div class="flex justify-end gap-4">
      <n-button @click="loadBansData">REFRESH</n-button>
      <n-button type="error" secondary @click="router.push({ name: 'createban' })">New Ban</n-button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, h, onBeforeMount, nextTick } from "vue"
import { useRouter } from "vue-router"
import { NInput, NDataTable, NButton, NSpace, NSelect, NTooltip, useNotification } from "naive-ui"
import type { DataTableSortState, PaginationInfo, DataTableColumn } from "naive-ui"
import axiosClient from "../axios"
import type { Ban } from "../types"
import { toLocal, renderSteamID, toErrorMsg } from "../utils"
import ActionButton from "../components/ActionButton.vue"

interface BanQuery {
  player?: string
  bannedBy?: string
  reason?: string | null
}

const router = useRouter()
const notification = useNotification()

const banReasonOptions = [
  { label: "Macro", value: "macro" },
  { label: "Auto Bhop", value: "auto-bhop" },
  { label: "Auto Strafe", value: "auto-strafe" },
]

const loading = ref(false)

const banQuery = reactive<BanQuery>({
  player: "",
  bannedBy: "",
  reason: null,
})

const columns = ref<DataTableColumn<Ban>[]>([
  {
    title: "ID",
    key: "id",
    sortOrder: false,
    sorter: "default",
  },
  {
    title: "Name",
    key: "name",
    render(rowData) {
      return rowData.player.name
    },
  },
  {
    title: "Steam ID",
    key: "steam_id",
    render(rowData) {
      return renderSteamID(rowData.player.id, true)
    },
  },
  {
    title: "Reason",
    key: "reason",
    render(rowData) {
      return rowData.reason.replace(/-/g, " ").replace(/\b\w/g, (c) => c.toUpperCase())
    },
  },
  {
    title: "Banned By",
    key: "banned_by",
    render(rowData) {
      return rowData.banned_by.type === "server" ? "Anticheat" : renderSteamID(rowData.banned_by.id.toString(), true)
    },
  },
  {
    title: "Created On",
    key: "created_on",
    sortOrder: false,
    render(rowData) {
      return toLocal(rowData.created_at)
    },
    sorter(rowA, rowB) {
      return new Date(rowA.created_at).getTime() - new Date(rowB.created_at).getTime()
    },
  },
  {
    title: "Actions",
    key: "actions",
    render(rowData) {
      return [
        !rowData.unban &&
          h(
            NTooltip,
            { trigger: "hover" },
            {
              trigger: () =>
                h(ActionButton, {
                  iconName: "edit",
                  style: {
                    marginRight: "0.5rem",
                  },
                  onClick: () => {
                    router.push({
                      name: "updateban",
                      params: {
                        id: rowData.id,
                      },
                    })
                  },
                }),
              default: () => "Update",
            },
          ),
        ,
        !rowData.unban &&
          h(
            NTooltip,
            { trigger: "hover" },
            {
              trigger: () =>
                h(ActionButton, {
                  iconName: "unban",
                  style: {
                    marginRight: "0.5rem",
                  },
                  onClick: () => {
                    router.push({
                      name: "unban",
                      params: {
                        id: rowData.id,
                      },
                    })
                  },
                }),
              default: () => "Unban",
            },
          ),
        ,
        h(
          NTooltip,
          { trigger: "hover" },
          {
            trigger: () =>
              h(ActionButton, {
                iconName: "more",
                style: {
                  marginRight: "0.5rem",
                },
                onClick: () => {
                  router.push({
                    name: "bandetails",
                    params: {
                      id: rowData.id,
                    },
                  })
                },
              }),
            default: () => "Details",
          },
        ),
        ,
      ]
    },
  },
])

const pagination = reactive({
  page: 1,
  pageSize: 10,
  showSizePicker: true,
  pageSizes: [10, 20, 50],
  prefix(info: PaginationInfo) {
    return `Total: ${info.itemCount} bans`
  },
  onChange: (page: number) => {
    pagination.page = page
  },
  onUpdatePageSize: (pageSize: number) => {
    pagination.pageSize = pageSize
    pagination.page = 1
  },
})

const data = ref<Ban[]>([])

onBeforeMount(() => {
  loadBansData()
})

async function loadBansData() {
  loading.value = true
  try {
    const params = {
      player: banQuery.player || null,
      banned_by: banQuery.bannedBy || null,
      reason: banQuery.reason,
    }

    const { data: res } = await axiosClient.get("/bans", {
      params,
    })

    data.value = res?.values || []
  } catch (error) {
    console.error(error)

    notification.error({
      title: "Failed to fetch bans",
      content: toErrorMsg(error),
    })
  } finally {
    loading.value = false
  }
}

function clearFilter() {
  banQuery.player = ""
  banQuery.bannedBy = ""
  banQuery.reason = null
  loadBansData()
}

function rowKey(rowData: Ban) {
  return rowData.id
}

function handleSorterChange(sorter: DataTableSortState) {
  columns.value.forEach((column) => {
    if ("sortOrder" in column) {
      if (!sorter) {
        column.sortOrder = false
        return
      }
      if (column.key === sorter.columnKey) column.sortOrder = sorter.order
      else column.sortOrder = false
    }
  })
}
</script>
