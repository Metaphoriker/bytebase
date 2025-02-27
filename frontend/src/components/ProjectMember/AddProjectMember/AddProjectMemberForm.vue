<template>
  <div class="w-full flex flex-col justify-start items-start gap-y-4">
    <MembersBindingSelect
      v-model:value="state.memberList"
      :required="true"
      :include-all-users="true"
      :include-service-account="true"
    >
      <template #suffix>
        <NButton v-if="allowRemove" text @click="$emit('remove')">
          <template #icon>
            <heroicons:trash class="w-4 h-4" />
          </template>
        </NButton>
      </template>
    </MembersBindingSelect>

    <div class="w-full">
      <div class="flex items-center gap-x-1">
        <span>{{ $t("settings.members.assign-role") }}</span>
        <span class="text-red-600">*</span>
      </div>
      <ProjectRoleSelect v-model:role="state.role" class="mt-2" />
    </div>
    <div class="w-full">
      <span>{{ $t("common.reason") }}</span>
      <NInput
        v-model:value="state.reason"
        class="mt-2"
        type="textarea"
        rows="2"
        :placeholder="$t('project.members.assign-reason')"
      />
    </div>
    <div
      v-if="
        state.role === PresetRoleType.PROJECT_QUERIER ||
        state.role === PresetRoleType.PROJECT_EXPORTER
      "
      class="w-full"
    >
      <span class="block mb-2">{{ $t("common.databases") }}</span>
      <QuerierDatabaseResourceForm
        :project-name="project.name"
        :database-resources="state.databaseResources"
        @update:condition="state.databaseResourceCondition = $event"
        @update:database-resources="state.databaseResources = $event"
      />
    </div>
    <template v-if="state.role === PresetRoleType.PROJECT_EXPORTER">
      <div class="w-full flex flex-col justify-start items-start">
        <span class="mb-2">
          {{ $t("issue.grant-request.export-rows") }}
        </span>
        <MaxRowCountSelect v-model:value="state.maxRowCount" />
      </div>
    </template>

    <div class="w-full flex flex-col gap-y-2">
      <span>{{ $t("common.expiration") }}</span>
      <ExpirationSelector
        class="grid-cols-3 sm:grid-cols-4"
        :value="state.expireDays"
        @update="state.expireDays = $event"
      />
    </div>
  </div>
</template>

<script lang="ts" setup>
/* eslint-disable vue/no-mutating-props */
import dayjs from "dayjs";
import { head } from "lodash-es";
import { NInput, NButton } from "naive-ui";
import { computed, reactive, watch } from "vue";
import ExpirationSelector from "@/components/ExpirationSelector.vue";
import QuerierDatabaseResourceForm from "@/components/GrantRequestPanel/DatabaseResourceForm/index.vue";
import MaxRowCountSelect from "@/components/GrantRequestPanel/MaxRowCountSelect.vue";
import MembersBindingSelect from "@/components/Member/MembersBindingSelect.vue";
import { ProjectRoleSelect } from "@/components/v2/Select";
import { useSettingV1Store } from "@/store";
import type { ComposedProject, DatabaseResource } from "@/types";
import { PresetRoleType } from "@/types";
import { Expr } from "@/types/proto/google/type/expr";
import type { Binding } from "@/types/proto/v1/iam_policy";
import { displayRoleTitle, extractDatabaseResourceName } from "@/utils";

const props = defineProps<{
  project: ComposedProject;
  binding: Binding;
  allowRemove: boolean;
}>();

defineEmits<{
  (event: "remove"): void;
}>();

interface LocalState {
  memberList: string[];
  role?: string;
  reason: string;
  expireDays: number;
  // Querier and exporter options.
  databaseResourceCondition?: string;
  databaseResources?: DatabaseResource[];
  // Exporter options.
  maxRowCount: number;
  databaseId?: string;
}

const getInitialState = (): LocalState => {
  const defaultState: LocalState = {
    memberList: props.binding.members,
    reason: "",
    // Default to never expire.
    expireDays: 0,
    maxRowCount: 1000,
  };

  if (maximumRoleExpiration.value) {
    defaultState.expireDays = maximumRoleExpiration.value;
  }
  return defaultState;
};

const maximumRoleExpiration = computed(() => {
  const seconds =
    settingV1Store.workspaceProfileSetting?.maximumRoleExpiration?.seconds?.toNumber();
  if (!seconds) {
    return undefined;
  }
  return Math.floor(seconds / (60 * 60 * 24));
});

const settingV1Store = useSettingV1Store();
const state = reactive<LocalState>(getInitialState());

watch(
  () => state.role,
  () => {
    state.databaseResourceCondition = undefined;
    state.databaseResources = undefined;
  },
  {
    immediate: true,
  }
);

watch(
  () => state,
  () => {
    const conditionName = generateConditionTitle();
    props.binding.members = state.memberList;
    if (state.role) {
      props.binding.role = state.role;
    }
    const expression: string[] = [];
    if (state.expireDays > 0) {
      const now = dayjs();
      const expiresAt = now.add(state.expireDays, "days");
      expression.push(`request.time < timestamp("${expiresAt.toISOString()}")`);
    }
    if (state.role === PresetRoleType.PROJECT_QUERIER) {
      if (state.databaseResourceCondition) {
        expression.push(state.databaseResourceCondition);
      }
    }
    if (state.role === PresetRoleType.PROJECT_EXPORTER) {
      if (state.databaseResourceCondition) {
        expression.push(state.databaseResourceCondition);
      }
      if (state.maxRowCount) {
        expression.push(`request.row_limit <= ${state.maxRowCount}`);
      }
    }
    props.binding.condition = Expr.create({
      title: conditionName,
      description: state.reason,
      expression: expression.length > 0 ? expression.join(" && ") : undefined,
    });
  },
  {
    deep: true,
  }
);

const generateConditionTitle = () => {
  if (!state.role) {
    return "";
  }

  const title = [displayRoleTitle(state.role)];
  if (
    state.role === PresetRoleType.PROJECT_QUERIER ||
    state.role === PresetRoleType.PROJECT_EXPORTER
  ) {
    let conditionSuffix = "";
    if (!state.databaseResources || state.databaseResources.length === 0) {
      conditionSuffix = `All databases`;
    } else if (state.databaseResources.length <= 3) {
      const databaseResourceNames = state.databaseResources.map((ds) =>
        getDatabaseResourceName(ds)
      );
      conditionSuffix = `${databaseResourceNames.join(", ")}`;
    } else {
      const firstDatabaseResourceName = getDatabaseResourceName(
        head(state.databaseResources)!
      );
      conditionSuffix = `${firstDatabaseResourceName} and ${
        state.databaseResources.length - 1
      } more`;
    }
    title.push(conditionSuffix);
  }
  if (state.expireDays > 0) {
    const now = dayjs();
    const expiresAt = now.add(state.expireDays, "days");
    title.push(`${now.format("L")}-${expiresAt.format("L")}`);
  }

  return title.join(" ");
};

const getDatabaseResourceName = (databaseResource: DatabaseResource) => {
  const { databaseName } = extractDatabaseResourceName(
    databaseResource.databaseName
  );
  if (databaseResource.table) {
    if (databaseResource.schema) {
      return `${databaseName}.${databaseResource.schema}.${databaseResource.table}`;
    } else {
      return `${databaseName}.${databaseResource.table}`;
    }
  } else if (databaseResource.schema) {
    return `${databaseName}.${databaseResource.schema}`;
  } else {
    return databaseName;
  }
};

defineExpose({
  allowConfirm: computed(() => {
    if (state.memberList.length <= 0) {
      return false;
    }
    if ((!state.expireDays && state.expireDays !== 0) || state.expireDays < 0) {
      return false;
    }
    // TODO: use parsed expression to check if the expression is valid.
    return true;
  }),
});
</script>
