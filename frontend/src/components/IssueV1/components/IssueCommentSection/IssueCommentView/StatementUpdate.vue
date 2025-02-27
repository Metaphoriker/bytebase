<template>
  <NButton text type="primary" v-bind="$attrs" @click="showPanel = true">
    {{ $t("common.view-details") }}
  </NButton>

  <BBModal
    v-if="showPanel"
    :title="$t('common.detail')"
    header-class="!border-0"
    container-class="!pt-0"
    @close="showPanel = false"
  >
    <div
      style="width: calc(100vw - 9rem); height: calc(100vh - 10rem)"
      class="relative"
    >
      <DiffEditor
        v-if="!isPreparingOldStatement && !isPreparingNewStatement"
        class="w-full h-full border rounded-md overflow-clip"
        :original="oldStatement"
        :modified="newStatement"
        :readonly="true"
      />
      <div
        v-else
        class="absolute inset-0 flex flex-col items-center justify-center"
      >
        <BBSpin />
      </div>
    </div>
  </BBModal>
</template>

<script setup lang="ts">
import { asyncComputed } from "@vueuse/core";
import { NButton } from "naive-ui";
import { ref } from "vue";
import { BBModal, BBSpin } from "@/bbkit";
import { DiffEditor } from "@/components/MonacoEditor";
import { useSheetV1Store } from "@/store";
import { UNKNOWN_ID } from "@/types";
import { extractSheetUID, getSheetStatement } from "@/utils";

const props = defineProps<{
  // Format: projects/{project}/sheets/{sheet}
  oldSheet: string;
  // Format: projects/{project}/sheets/{sheet}
  newSheet: string;
}>();

const showPanel = ref(false);

const prepareStatement = async (sheetName: string) => {
  if (extractSheetUID(sheetName) === String(UNKNOWN_ID)) return "";
  // Use any (basic or full) view of sheets here to save data size
  const sheet = await useSheetV1Store().getOrFetchSheetByName(sheetName);
  if (!sheet) return "";
  return getSheetStatement(sheet);
};

const isPreparingOldStatement = ref(false);
const isPreparingNewStatement = ref(false);

const oldStatement = asyncComputed(
  () => {
    return prepareStatement(props.oldSheet);
  },
  "",
  {
    evaluating: isPreparingOldStatement,
    lazy: true,
  }
);

const newStatement = asyncComputed(
  () => {
    return prepareStatement(props.newSheet);
  },
  "",
  {
    evaluating: isPreparingNewStatement,
    lazy: true,
  }
);
</script>
