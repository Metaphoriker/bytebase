<template>
  <Drawer :show="true" @close="onDismiss">
    <DrawerContent :title="$t('settings.sensitive-data.grant-access')">
      <div class="divide-block-border space-y-8 h-full">
        <SensitiveColumnTable
          :row-clickable="false"
          :show-operation="false"
          :row-selectable="false"
          :column-list="columnList"
          :checked-column-index-list="[]"
        />

        <div class="w-full">
          <p class="mb-2">{{ $t("settings.sensitive-data.action.self") }}</p>
          <div class="flex space-x-4">
            <NCheckbox
              v-for="action in ACTIONS"
              :key="action"
              :checked="state.supportActions.has(action)"
              @update:checked="toggleAction(action, $event)"
            >
              {{
                $t(
                  `settings.sensitive-data.action.${maskingExceptionPolicy_MaskingException_ActionToJSON(
                    action
                  ).toLowerCase()}`
                )
              }}
            </NCheckbox>
          </div>
        </div>

        <div class="w-full">
          <p class="mb-2">
            {{ $t("settings.sensitive-data.masking-level.self") }}
          </p>
          <MaskingLevelRadioGroup
            :level="state.maskingLevel"
            :level-list="MASKING_LEVELS"
            @update:level="state.maskingLevel = $event"
          />
        </div>

        <div class="w-full">
          <p class="mb-2">{{ $t("common.expiration") }}</p>
          <NDatePicker
            v-model:value="state.expirationTimestamp"
            style="width: 100%"
            type="datetime"
            :is-date-disabled="(date: number) => date < Date.now()"
            clearable
          />
          <span v-if="!state.expirationTimestamp" class="textinfolabel">{{
            $t("settings.sensitive-data.never-expires")
          }}</span>
        </div>

        <MembersBindingSelect
          v-model:value="state.memberList"
          :required="true"
          :project-name="projectName"
          :include-all-users="false"
          :include-service-account="false"
        />
      </div>

      <template #footer>
        <div class="w-full flex justify-between items-center">
          <div class="w-full flex justify-end items-center gap-x-3">
            <NButton @click.prevent="onDismiss">
              {{ $t("common.cancel") }}
            </NButton>
            <NButton
              :disabled="submitDisabled || state.processing"
              type="primary"
              @click.prevent="onSubmit"
            >
              {{ $t("common.confirm") }}
            </NButton>
          </div>
        </div>
      </template>
    </DrawerContent>
  </Drawer>
</template>

<script lang="ts" setup>
import { groupBy } from "lodash-es";
import { NButton, NCheckbox, NDatePicker } from "naive-ui";
import { computed, reactive } from "vue";
import { useI18n } from "vue-i18n";
import MembersBindingSelect from "@/components/Member/MembersBindingSelect.vue";
import { Drawer, DrawerContent } from "@/components/v2";
import { usePolicyV1Store, pushNotification } from "@/store";
import type { ComposedProject } from "@/types";
import { Expr } from "@/types/proto/google/type/expr";
import { MaskingLevel } from "@/types/proto/v1/common";
import type {
  Policy,
  MaskingExceptionPolicy_MaskingException,
} from "@/types/proto/v1/org_policy_service";
import {
  PolicyType,
  PolicyResourceType,
  MaskingExceptionPolicy_MaskingException_Action,
  maskingExceptionPolicy_MaskingException_ActionToJSON,
} from "@/types/proto/v1/org_policy_service";
import MaskingLevelRadioGroup from "./components/MaskingLevelRadioGroup.vue";
import SensitiveColumnTable from "./components/SensitiveColumnTable.vue";
import type { SensitiveColumn } from "./types";
import { getExpressionsForSensitiveColumn } from "./utils";

const props = defineProps<{
  columnList: SensitiveColumn[];
  projectName: string;
}>();

const emit = defineEmits(["dismiss"]);

interface LocalState {
  memberList: string[];
  expirationTimestamp?: number;
  maskingLevel: MaskingLevel;
  processing: boolean;
  supportActions: Set<MaskingExceptionPolicy_MaskingException_Action>;
}

const ACTIONS = [
  MaskingExceptionPolicy_MaskingException_Action.EXPORT,
  MaskingExceptionPolicy_MaskingException_Action.QUERY,
];
const MASKING_LEVELS = [MaskingLevel.PARTIAL, MaskingLevel.NONE];

const state = reactive<LocalState>({
  memberList: [],
  maskingLevel: MaskingLevel.PARTIAL,
  processing: false,
  supportActions: new Set(ACTIONS),
});

const policyStore = usePolicyV1Store();
const { t } = useI18n();

const resetState = () => {
  state.expirationTimestamp = undefined;
  state.maskingLevel = MaskingLevel.PARTIAL;
  state.supportActions = new Set(ACTIONS);
  state.memberList = [];
};

const onDismiss = () => {
  emit("dismiss");
  resetState();
};

const submitDisabled = computed(() => {
  if (state.memberList.length === 0) {
    return true;
  }
  if (state.supportActions.size === 0) {
    return true;
  }
  return false;
});

const onSubmit = async () => {
  state.processing = true;

  const groupByProject = groupBy(
    props.columnList,
    (item) => item.database.project
  );
  try {
    for (const [project, columnList] of Object.entries(groupByProject)) {
      if (columnList.length === 0) {
        continue;
      }
      const pendingUpdate = await getPendingUpdatePolicy(
        columnList[0].database.projectEntity,
        columnList
      );
      await policyStore.upsertPolicy({
        parentPath: project,
        policy: pendingUpdate,
        updateMask: ["payload"],
      });
    }
    pushNotification({
      module: "bytebase",
      style: "SUCCESS",
      title: t("common.created"),
    });
    onDismiss();
  } finally {
    state.processing = false;
  }
};

const getPendingUpdatePolicy = async (
  project: ComposedProject,
  columnList: SensitiveColumn[]
): Promise<Partial<Policy>> => {
  const maskingExceptions: MaskingExceptionPolicy_MaskingException[] = [];

  for (const column of columnList) {
    const expressions = getExpressionsForSensitiveColumn(column);
    if (state.expirationTimestamp) {
      expressions.push(
        `request.time < timestamp("${new Date(
          state.expirationTimestamp
        ).toISOString()}")`
      );
    }

    for (const action of state.supportActions.values()) {
      for (const member of state.memberList) {
        maskingExceptions.push({
          member,
          action,
          maskingLevel: state.maskingLevel,
          condition: Expr.fromPartial({
            expression: expressions.join(" && "),
          }),
        });
      }
    }
  }

  const policy = await policyStore.getOrFetchPolicyByParentAndType({
    parentPath: project.name,
    policyType: PolicyType.MASKING_EXCEPTION,
  });
  const existed = policy?.maskingExceptionPolicy?.maskingExceptions ?? [];
  return {
    name: policy?.name,
    type: PolicyType.MASKING_EXCEPTION,
    resourceType: PolicyResourceType.PROJECT,
    maskingExceptionPolicy: {
      maskingExceptions: [...existed, ...maskingExceptions],
    },
  };
};

const toggleAction = (
  action: MaskingExceptionPolicy_MaskingException_Action,
  check: boolean
) => {
  if (check) {
    state.supportActions.add(action);
  } else {
    state.supportActions.delete(action);
  }
};
</script>
