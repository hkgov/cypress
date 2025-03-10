<template>
  <div
    data-cy="debug-spec-col"
    class="flex flex-col grid gap-24px self-stretch"
  >
    <div
      data-cy="debug-spec-item"
      class="rounded flex flex-col border-gray-100 border-t-1px border-x-1px w-full overflow-hidden items-start box-border"
    >
      <div
        data-cy="debug-spec-header"
        class="rounded-t flex flex-row bg-gray-50 border-b-1px border-b-gray-100 w-full py-12px items-center"
      >
        <div
          data-cy="spec-contents"
          class="flex w-full grid px-18px gap-y-8px items-center"
        >
          <div class="flex-grow flex w-full gap-x-2 truncate items-center">
            <IconDocumentText
              stroke-color="gray-500"
              fill-color="gray-100"
              size="16"
            />
            <div
              data-cy="spec-path"
              class="flex-grow text-base non-italic truncate"
            >
              <span
                class="font-normal text-gray-600"
              >
                {{ specData.path }}
              </span>
              <span
                class="inline-flex items-center"
              >
                <SpecNameDisplay
                  :spec-file-name="specData.fileName"
                  :spec-file-extension="specData.fileExtension"
                />
              </span>
            </div>
          </div>
          <ul
            data-cy="spec-header-metadata"
            class="flex flex-wrap font-normal text-sm text-gray-700 gap-x-3 items-center whitespace-nowrap children:flex children:items-center"
          >
            <li
              :data-cy="'debugHeader-results'"
            >
              <ResultCounts
                :total-failed="specData.testsFailed"
                :total-passed="specData.testsPassed"
                :total-pending="specData.testsPending"
                :order="['FAILED', 'PASSED', 'PENDING']"
              />
            </li>
            <div class="-mt-6px text-lg text-gray-400">
              .
            </div>
            <StatsMetaData
              v-if="(Object.keys(specData.groups).length === 1)"
              :order="['DURATION','OS','BROWSER','TESTING']"
              :spec-duration="specData.specDuration!"
              :testing="specData.testingType"
              :groups="Object.values(specData.groups)"
            />
            <StatsMetaData
              v-else-if="(Object.keys(specData.groups).length > 1)"
              :order="['DURATION', 'GROUPS', 'G_OS', 'G_BROWSERS', 'TESTING']"
              :spec-duration="specData.specDuration!"
              :testing="specData.testingType"
              :groups="Object.values(specData.groups)"
            />
          </ul>
        </div>
        <div
          class="mr-16px"
        >
          <Tooltip
            placement="bottom"
            color="dark"
            :is-interactive="!!(runAllFailuresState.cta)"
            :disabled="!runAllFailuresState.disabled"
          >
            <Button
              data-cy="run-failures"
              variant="white"
              class="gap-x-10px inline-flex whitespace-nowrap justify-center items-center isolate"
              :disabled="runAllFailuresState.disabled"
              :to="{ path: '/specs/runner', query: { file: posixify(specData.fullPath), mode: 'debug' } }"
            >
              <template #prefix>
                <IconActionRefresh
                  data-cy="icon-refresh"
                  stroke-color="indigo-500"
                />
              </template>
              <!-- Wrapping this with a default template to avoid an unneeded space -->
              <template #default>
                {{ t('debugPage.runFailures.btn') }}
              </template>
            </Button>
            <template
              v-if="runAllFailuresState.disabled"
              #popper
            >
              <div
                class="flex flex-col text-sm max-w-350px items-center"
                data-cy="run-all-failures-tooltip"
              >
                <span class="text-center">{{ runAllFailuresState.message }}</span>
                <Button
                  v-if="runAllFailuresState.cta"
                  variant="text"
                  class="rounded-md font-medium bg-gray-800 my-12px"
                  @click="runAllFailuresState.cta?.action"
                >
                  {{ runAllFailuresState.cta.message }}
                </Button>
              </div>
            </template>
          </Tooltip>
        </div>
      </div>
      <div
        v-for="thumbprint in Object.keys(specData.failedTests)"
        :key="`test-${thumbprint}`"
        :data-cy="`test-group`"
        class="flex flex-col flex-start border-b-gray-100 border-b-1px w-full pr-16px pl-16px justify-center"
        :class="Object.keys(specData.groups).length > 1 ? 'pb-16px': 'hover:bg-gray-50'"
      >
        <DebugFailedTest
          v-if="specData.failedTests[thumbprint].length >= 1"
          :failed-tests-result="specData.failedTests[thumbprint]"
          :groups="groupsPerTest[thumbprint]"
          :expandable="Object.keys(specData.groups).length > 1"
        />
      </div>
    </div>
  </div>
</template>
<script lang="ts" setup>

import { computed, unref } from 'vue'
import { IconActionRefresh, IconDocumentText } from '@cypress-design/vue-icon'
import type { SpecDataAggregate, CloudRunInstance } from '@packages/data-context/src/gen/graphcache-config.gen'
import DebugFailedTest from './DebugFailedTest.vue'
import StatsMetaData from './StatsMetadata.vue'
import ResultCounts from '@packages/frontend-shared/src/components/ResultCounts.vue'
import Button from '@packages/frontend-shared/src/components/Button.vue'
import Tooltip from '@packages/frontend-shared/src/components/Tooltip.vue'
import SpecNameDisplay from '../specs/SpecNameDisplay.vue'
import { useI18n } from '@cy/i18n'
import { useDurationFormat } from '../composables/useDurationFormat'
import { posixify } from '../paths'
import type { StatsMetadata_GroupsFragment, TestingTypeEnum } from '../generated/graphql'

export interface Spec {
  id: string
  path: string
  fileName: string
  fileExtension: string
  testsPassed: SpecDataAggregate | null
  testsFailed: SpecDataAggregate | null
  testsPending: SpecDataAggregate | null
  specDuration: SpecDataAggregate | null
  fullPath: string
}

export interface TestResults {
  readonly id: string
  readonly titleParts: ReadonlyArray<string>
  readonly instance: CloudRunInstance | null
}

const { t } = useI18n()

const props = defineProps<{
  spec: Spec
  testResults: {[thumbprint: string]: TestResults[]}
  groups: {[groupId: string]: StatsMetadata_GroupsFragment }
  testingType: TestingTypeEnum
  foundLocally: boolean
  matchesCurrentTestingType: boolean
}>()

const emits = defineEmits<{
  (event: 'switchTestingType', value: TestingTypeEnum): void
}>()

// helper function for formatting the number of passed, failed, and pending tests

const durationFormatter = (val: number | null | undefined) => {
  return unref(useDurationFormat(val ?? 0))
}

// helper function for formatting the number of passed, failed, and pending tests
const debugResultsCalc = (value: SpecDataAggregate | null, formatter?: (val: number | null | undefined) => string) => {
  if (!value) {
    return ''
  }

  const formattedMin = formatter ? formatter(value.min) : value.min
  const formattedMax = formatter ? formatter(value.max) : value.max

  if (formattedMin === formattedMax) {
    return formattedMin
  }

  return `${formattedMin}-${formattedMax}`
}

const specData = computed(() => {
  return {
    path: props.spec.path,
    fileName: props.spec.fileName,
    fileExtension: props.spec.fileExtension,
    failedTests: props.testResults,
    testsPassed: debugResultsCalc(props.spec.testsPassed),
    testsFailed: debugResultsCalc(props.spec.testsFailed),
    testsPending: debugResultsCalc(props.spec.testsPending),
    specDuration: debugResultsCalc(props.spec.specDuration, durationFormatter),
    groups: props.groups,
    testingType: props.testingType,
    fullPath: props.spec.fullPath,
  }
})

/**
 * Helper function that maps each test's thumbprint to all the groups in it
 */
const groupsPerTest = computed(() => {
  return Object.keys(props.testResults).reduce<Record<string, StatsMetadata_GroupsFragment[]>>((acc, currThumbprint) => {
    acc[currThumbprint] = props.testResults[currThumbprint].map((test) => props.groups[test.instance?.groupId || ''])

    return acc
  }, {})
})

const runAllFailuresState = computed(() => {
  if (!props.matchesCurrentTestingType) {
    return {
      disabled: true,
      message: t('debugPage.runFailures.switchTestingTypeInfo', { n: Object.keys(props.testResults).length, testingType: props.testingType }),
      cta: {
        message: t('debugPage.runFailures.switchTestingTypeAction', { testingType: props.testingType }),
        action: () => emits('switchTestingType', props.testingType),
      },
    }
  }

  if (!props.foundLocally) {
    return {
      disabled: true,
      message: t('debugPage.runFailures.notFoundLocally'),
    }
  }

  return { disabled: false }
})

</script>
