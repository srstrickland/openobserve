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

<template>
  <div class="q-pa-md row">
    <div class="col q-pr-xs">
      <div class="flex justify-start bordered q-mb-md">
        <div class="q-mr-md">
          <div class="flex items-center q-pt-xs">
            <span style="font-size: 14px"> Show </span>
            <div
              class="q-ml-xs"
              style="
                border: 1px solid #d7d7d7;
                width: fit-content;
                border-radius: 2px;
              "
            >
              <template
                v-for="visual in usersDisplayOptions"
                :key="visual.value"
              >
                <q-btn
                  :color="visual.value === usersDisplay ? 'primary' : ''"
                  :flat="visual.value === usersDisplay ? false : true"
                  dense
                  no-caps
                  size="11px"
                  class="q-px-md visual-selection-btn"
                  @click="updateUserTable(visual.value)"
                >
                  {{ visual.label }}</q-btn
                >
              </template>
            </div>
          </div>
        </div>
        <div class="o2-input q-mr-md" style="width: 400px">
          <q-input
            data-test="alert-list-search-input"
            v-model="userSearchKey"
            borderless
            filled
            dense
            class="q-ml-auto q-mb-xs no-border"
            placeholder="Search"
          >
            <template #prepend>
              <q-icon name="search" class="cursor-pointer" />
            </template>
          </q-input>
        </div>
      </div>
      <div class="q-ml-sm q-mb-sm text-bold">{{ rows.length }} Roles</div>
      <template v-if="rows.length">
        <app-table
          :rows="rows"
          :columns="columns"
          :dense="true"
          style="height: fit-content"
          :filter="{
            value: userSearchKey,
            method: filterRoles,
          }"
        >
          <template v-slot:select="slotProps">
            <q-checkbox
              size="xs"
              v-model="slotProps.column.row.isInGroup"
              class="filter-check-box cursor-pointer"
              @click="toggleUserSelection(slotProps.column.row)"
            />
          </template>
        </app-table>
      </template>
      <div v-if="!rows.length" class="q-mt-md text-bold q-pl-md">
        No users added
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { watch } from "vue";
import AppTable from "@/components/AppTable.vue";
import usePermissions from "@/composables/iam/usePermissions";
import { cloneDeep } from "lodash-es";
import type { Ref } from "vue";
import { ref } from "vue";
import { useI18n } from "vue-i18n";
import { getRoles } from "@/services/iam";
import { useStore } from "vuex";

// show selected users in the table
// Add is_selected to the user object

const props = defineProps({
  groupRoles: {
    type: Array,
    default: () => [],
  },
  activeTab: {
    type: String,
    default: "users",
  },
  addedRoles: {
    type: Set,
    default: () => new Set(),
  },
  removedRoles: {
    type: Set,
    default: () => new Set(),
  },
});

const emits = defineEmits(["add", "remove"]);

const users = ref([
  {
    role_name: "example@example.com",
  },
  {
    role_name: "root@example.com",
  },
]);

watch(
  () => props.groupRoles,
  () => {
    updateGroupUsers();
  },
  {
    deep: true,
  }
);

const { rolesState, groupsState } = usePermissions();

const rows: Ref<any[]> = ref([]);

const userSearchKey = ref("");

const usersDisplay = ref("selected");

const usersDisplayOptions = [
  {
    label: "All",
    value: "all",
  },
  {
    label: "Selected",
    value: "selected",
  },
];

const { t } = useI18n();

const hasFetchedOrgUsers = ref(false);

const store = useStore();

const groupUsersMap = ref(new Set());

const columns = [
  {
    name: "select",
    field: "",
    label: "",
    align: "left",
    sortable: true,
    slot: true,
    slotName: "select",
  },
  {
    name: "role_name",
    field: "role_name",
    label: t("iam.roleName"),
    align: "left",
    sortable: true,
  },
];

watch(
  () => props.groupRoles,
  () => {
    updateGroupUsers();
  },
  {
    deep: true,
  }
);

const updateUserTable = async (value: string) => {
  usersDisplay.value = value;

  if (!hasFetchedOrgUsers.value && value === "all") {
    await getchOrgUsers();
  }

  if (usersDisplay.value === "all") {
    rows.value = users.value;
  } else {
    rows.value = users.value.filter((user: any) => user.isInGroup);
  }
};

const updateGroupUsers = () => {
  users.value = props.groupRoles.map((role: any, index: number) => {
    groupUsersMap.value.add(role);
    return {
      role_name: role,
      "#": index + 1,
      isInGroup: true,
    };
  });

  updateUserTable(usersDisplay.value);
};

const getchOrgUsers = async () => {
  // fetch group users
  hasFetchedOrgUsers.value = true;
  return new Promise(async (resolve) => {
    const data: any = await getRoles(
      store.state.selectedOrganization.identifier
    );

    users.value = cloneDeep(data.data).map((role: any, index: number) => {
      return {
        role_name: role,
        "#": index + 1,
        isInGroup: groupUsersMap.value.has(role.role_name),
      };
    });
    resolve(true);
  });
};

const toggleUserSelection = (user: any) => {
  if (user.isInGroup && !groupUsersMap.value.has(user.role_name)) {
    props.addedRoles.add(user.role_name);
  } else if (!user.isInGroup && groupUsersMap.value.has(user.role_name)) {
    props.removedRoles.add(user.role_name);
  }

  if (!user.isInGroup && props.addedRoles.has(user.role_name)) {
    props.addedRoles.delete(user.role_name);
  }

  if (user.isInGroup && props.removedRoles.has(user.role_name)) {
    props.removedRoles.delete(user.role_name);
  }
};

const filterRoles = (rows: any[], term: string) => {
  var filtered = [];
  for (var i = 0; i < rows.length; i++) {
    var user = rows[i];
    if (user.role_name.toLowerCase().indexOf(term.toLowerCase()) > -1) {
      filtered.push(user);
    }
  }
  return filtered;
};
</script>

<style scoped></style>
