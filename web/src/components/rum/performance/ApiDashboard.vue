<!-- Copyright 2023 Zinc Labs Inc.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<!-- eslint-disable vue/v-on-event-hyphenation -->
<!-- eslint-disable vue/attribute-hyphenation -->
<template>
  <q-page
    class="relative-position"
    :key="store.state.selectedOrganization.identifier"
  >
    <div
      class="api-performance-dashboards"
      :style="{ visibility: isLoading.length ? 'hidden' : 'visible' }"
    >
      <div class="q-px-md">
        <VariablesValueSelector
          :variablesConfig="apiDashboard?.variables"
          :selectedTimeDate="dateTime"
          @variablesData="variablesDataUpdated"
        />
      </div>
      <div class="row q-px-md">
        <div class="col-6 q-px-xs q-py-xs">
          <div class="view-error-table q-pa-sm">
            <div class="q-pb-sm text-bold q-pl-xs">Top Slowest Resources</div>
            <AppTable
              :columns="slowResourceColumn"
              :rows="topSlowResources"
              :virtualScroll="false"
              height="200px"
            />
          </div>
        </div>
        <div class="col-6 q-px-xs q-py-xs">
          <div class="view-error-table q-pa-sm">
            <div class="q-pb-sm text-bold q-pl-xs">Top Heaviest Resources</div>
            <AppTable
              :columns="heavyResourceColumn"
              :rows="topHeavyResources"
              :virtualScroll="false"
              height="200px"
            />
          </div>
        </div>
        <div class="col-6 q-px-xs q-py-xs">
          <div class="view-error-table q-pa-sm">
            <div class="q-pb-sm text-bold q-pl-xs">Top Error Resources</div>
            <AppTable
              :columns="errorResourceColumns"
              :rows="topErrorResources"
              :virtualScroll="false"
              height="200px"
            />
          </div>
        </div>
      </div>
    </div>
    <div
      v-show="isLoading.length"
      class="q-pb-lg flex items-center justify-center text-center absolute full-width"
      style="height: calc(100vh - 250px); top: 0"
    >
      <div>
        <q-spinner-hourglass
          color="primary"
          size="40px"
          style="margin: 0 auto; display: block"
        />
        <div class="text-center full-width">Loading Dashboard</div>
      </div>
    </div>
  </q-page>
</template>

<script lang="ts">
// @ts-nocheck
import { defineComponent, ref, watch, onMounted, onActivated } from "vue";
import { useStore } from "vuex";
import { useI18n } from "vue-i18n";
import { useRouter } from "vue-router";
import { reactive } from "vue";
import { useRoute } from "vue-router";
import AppTable from "@/components/AppTable.vue";
import searchService from "@/services/search";
import VariablesValueSelector from "@/components/dashboards/VariablesValueSelector.vue";
import apiDashboard from "@/utils/rum/api.json";

export default defineComponent({
  name: "ApiDashboard",
  components: {
    AppTable,
    VariablesValueSelector,
  },
  props: {
    dateTime: {
      type: Object,
      default: () => ({}),
    },
    selectedDate: {
      type: Object,
      default: () => ({}),
    },
  },
  setup(props) {
    const { t } = useI18n();
    const route = useRoute();
    const router = useRouter();
    const store = useStore();
    const currentDashboardData = reactive({
      data: {},
    });
    const showDashboardSettingsDialog = ref(false);
    const viewOnly = ref(true);
    const eventLog = ref([]);
    const topSlowResources = ref([]);
    const topHeavyResources = ref([]);
    const topErrorResources = ref([]);
    const variablesData = ref(null);

    const refDateTime: any = ref(null);
    const refreshInterval = ref(0);
    const topCount = 10;
    const isLoading: Ref<boolean[]> = ref([]);

    // variables data
    const variablesDataUpdated = (data: any) => {
      if (JSON.stringify(variablesData.value) === JSON.stringify(data)) return;

      variablesData.value = data;
      if (variablesData.value?.values?.length) {
        const areVariablesLoaded = variablesData.value.values.every(
          (element: any) => element.value
        );
        if (areVariablesLoaded) {
          getTopErrorResources();
          getTopHeavyResources();
          getTopSlowResources();
        }
      }
    };

    const slowResourceColumn = [
      {
        name: "url",
        label: "Resource URL",
        field: (row) => row["url"],
        align: "left",
      },
      {
        name: "max_duration",
        label: "Duration (ms)",
        field: (row: any) => row["max_duration"],
        align: "left",
        sortable: true,
        style: { width: "56px !important" },
      },
    ];

    const errorResourceColumns = [
      {
        name: "url",
        label: "Resource URL",
        field: (row) => row["url"],
        align: "left",
      },
      {
        name: "error_count",
        label: "Error Count",
        field: (row: any) => row["error_count"],
        align: "left",
        sortable: true,
        style: { width: "56px" },
      },
    ];

    const heavyResourceColumn = [
      {
        name: "url",
        label: "Resource URL",
        field: (row) => row["url"],
        align: "left",
      },
      {
        name: "max_resource_size",
        label: "Size (kb)",
        field: (row: any) => row["max_resource_size"],
        align: "left",
        sortable: true,
        style: { width: "56px" },
      },
    ];

    onMounted(async () => {
      await loadDashboard();
      updateLayout();
    });

    onActivated(() => {
      updateLayout();
    });

    const updateLayout = async () => {
      isLoading.value.push(true);

      // Settimeout is used to make consitent loading experience across all tabs
      // As due to vue-grid-layout issue in summary and webvitals
      setTimeout(() => {
        isLoading.value.pop();
      }, 1000);
    };

    const getTopSlowResources = () => {
      topSlowResources.value = [];

      let whereClause = `where resource_duration>=0 and resource_method is not null`;
      const variablesString = getVariablesString();
      if (variablesString && variablesString.length) {
        whereClause += ` ${variablesString}`;
      }

      const req = {
        query: {
          sql: `SELECT avg(resource_duration/1000000) as max_duration, SPLIT_PART(resource_url, '?', 1) AS url FROM "_rumdata" ${whereClause} group by url order by max_duration desc`,
          start_time: props.selectedDate.startTime,
          end_time: props.selectedDate.endTime,
          from: 0,
          size: 10,
          sql_mode: "full",
        },
      };

      searchService
        .search({
          org_identifier: store.state.selectedOrganization.identifier,
          query: req,
          page_type: "logs",
        })
        .then((res) => {
          res.data.hits.slice(0, topCount).forEach((element: any) => {
            topSlowResources.value.push({
              url: new URL(element.url).pathname,
              max_duration: element.max_duration.toFixed(2),
            });
          });
        });
    };

    const getTopHeavyResources = () => {
      topHeavyResources.value = [];

      let whereClause = `WHERE resource_size>=0`;
      const variablesString = getVariablesString();
      if (variablesString && variablesString.length) {
        whereClause += ` ${variablesString}`;
      }

      const req = {
        query: {
          sql: `SELECT avg(resource_size/1024) as max_resource_size, SPLIT_PART(resource_url, '?', 1) AS url FROM "_rumdata" ${whereClause} group by url order by max_resource_size desc`,
          start_time: props.selectedDate.startTime,
          end_time: props.selectedDate.endTime,
          from: 0,
          size: topCount,
          sql_mode: "full",
        },
      };

      searchService
        .search({
          org_identifier: store.state.selectedOrganization.identifier,
          query: req,
          page_type: "logs",
        })
        .then((res) => {
          res.data.hits.forEach((element: any) => {
            topHeavyResources.value.push({
              ...element,
              url: new URL(element.url).pathname,
            });
          });
        });
    };

    const getTopErrorResources = () => {
      topErrorResources.value = [];

      let whereClause = `WHERE resource_status_code>400`;
      const variablesString = getVariablesString();
      if (variablesString && variablesString.length) {
        whereClause += ` ${variablesString}`;
      }

      const req = {
        query: {
          sql: `SELECT SPLIT_PART(resource_url, '?', 1) AS url, count(*) as error_count FROM "_rumdata" ${whereClause}  group by url order by error_count desc`,
          start_time: props.selectedDate.startTime,
          end_time: props.selectedDate.endTime,
          from: 0,
          size: topCount,
          sql_mode: "full",
        },
      };

      searchService
        .search({
          org_identifier: store.state.selectedOrganization.identifier,
          query: req,
          page_type: "logs",
        })
        .then((res) => {
          res.data.hits.forEach((element: any) => {
            topErrorResources.value.push({
              ...element,
              url: new URL(element.url).pathname,
            });
          });
        });
    };

    const getVariablesString = () => {
      let variablesString = "";
      variablesData.value?.values?.length &&
        variablesData.value.values.forEach((element: any) => {
          if (element.type === "query_values" && !!element.value) {
            variablesString += ` and ${element.name}='${element.value}'`;
          }
        });

      return variablesString;
    };

    const loadDashboard = async () => {
      currentDashboardData.data = apiDashboard;

      // if variables data is null, set it to empty list
      if (
        !(
          currentDashboardData.data?.variables &&
          currentDashboardData.data?.variables?.list.length
        )
      ) {
        variablesData.value.isVariablesLoading = false;
        variablesData.value.values = [];
      }
    };

    const addSettingsData = () => {
      showDashboardSettingsDialog.value = true;
    };

    watch(
      () => props.selectedDate,
      (newVal, oldValue) => {
        if (JSON.stringify(newVal) !== JSON.stringify(oldValue)) {
          getTopHeavyResources();
          getTopSlowResources();
        }
      }
    );

    return {
      currentDashboardData,
      t,
      store,
      refDateTime,
      refreshInterval,
      viewOnly,
      eventLog,
      variablesData,
      variablesDataUpdated,
      addSettingsData,
      showDashboardSettingsDialog,
      loadDashboard,
      topSlowResources,
      topHeavyResources,
      heavyResourceColumn,
      slowResourceColumn,
      errorResourceColumns,
      topErrorResources,
      apiDashboard,
      isLoading,
    };
  },
});
</script>

<style lang="scss" scoped>
.performance_title {
  font-size: 24px;
}
.q-table {
  &__top {
    border-bottom: 1px solid $border-color;
    justify-content: flex-end;
  }
}

.view-error-table {
  margin-top: 4px;
  border: 1px solid rgba(194, 194, 194, 0.4784313725) !important;
  border-radius: 4px;
  min-height: 200px;
}

.api-performance-dashboards {
  min-height: auto !important;
  max-height: calc(100vh - 196px);
  overflow-y: auto;
}
</style>

<style lang="scss">
.view-error-table {
  .q-table {
    td {
      padding: 6px 10px !important;
      height: auto !important;
    }
  }

  .q-table thead tr {
    th {
      padding: 6px 8px !important;
      height: auto !important;
    }
    padding: 6px 0px !important;
    height: auto !important;
  }
}
</style>
