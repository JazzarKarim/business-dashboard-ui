<script setup lang="ts">
import { CorpTypeCd, FilingTypes } from '@bcrs-shared-components/enums'
import { FilingSubTypeE } from '~/enums/filing-sub-type-e'
import type { DocumentI } from '~/interfaces/document-i'
import { BusinessStateE } from '~/enums/business-state-e'
import { fetchDocuments, saveBlob } from '~/utils/download-file'
import { AuthorizedActionsE, getContactInfo } from '#imports'
import { useBcrosLegalApi } from '~/composables/useBcrosLegalApi'
import { LDFlags } from '~/enums/ld-flags'

const t = useNuxtApp().$i18n.t
const {
  currentBusiness,
  comments,
  currentBusinessIdentifier,
  currentBusinessName,
  isFirm,
  businessConfig,
  currentBusinessAddresses
} = storeToRefs(useBcrosBusiness())
const { getStoredFlag } = useBcrosLaunchdarkly()
const { isAllowedToFile, isDisableNonBenCorps } = useBcrosBusiness()
const isCommentOpen = ref(false)
const isDissolutionDialogOpen = ref(false)
const { goToCreateUI, goToEditUI } = useBcrosNavigate()
const ui = useBcrosDashboardUi()
const filings = useBcrosFilings()

const isAllowedBusinessSummary = computed(() => {
  const supportedEntityTypes = getStoredFlag(LDFlags.SupportedBusinessSummaryEntities)?.split(' ')
  return !!currentBusinessIdentifier.value && supportedEntityTypes?.includes(currentBusiness?.value?.legalType)
})

const isCurrentlyEnabledBusinessSummary = computed(() => {
  const enabledEntityTypes = getStoredFlag(LDFlags.EnabledBusinessSummaryEntities)?.split(' ')
  return !!enabledEntityTypes?.includes(currentBusiness?.value?.legalType)
})

const businessSummaryTooltipText = computed(
  () => isAllowedBusinessSummary.value && isCurrentlyEnabledBusinessSummary.value
    ? t('tooltip.filing.button.businessSummary')
    : t('tooltip.filing.button.businessSummaryDisabled')
)

const isPendingDissolution = computed(() => {
  return false
  // todo: implement !!FUTURE not implemented in current dashboard
})

const isChangeBusinessInfoDisabled = computed(() => {
  if (!currentBusiness.value.goodStanding && !isAuthorized(AuthorizedActionsE.OVERRIDE_NIGS)) {
    return false
  }

  const isAllowed =
    // if it's coop
    (currentBusiness.value.legalType === CorpTypeCd.COOP &&
      !!getStoredFlag(LDFlags.SpecialResolutionUIEnabled) &&
      isAllowedToFile(FilingTypes.SPECIAL_RESOLUTION) &&
      isAuthorized(AuthorizedActionsE.SPECIAL_RESOLUTION_FILING)) ||
    // if it's firm
    (isFirm.value && isAllowedToFile(FilingTypes.CHANGE_OF_REGISTRATION) &&
    isAuthorized(AuthorizedActionsE.FIRM_CHANGE_FILING)) ||

    // otherwise
    (isAllowedToFile(FilingTypes.ALTERATION) && isAuthorized(AuthorizedActionsE.ALTERATION_FILING))

  return !isAllowed
})

const showCommentDialog = (show: boolean) => {
  isCommentOpen.value = show
}

const setShowDissolutionDialog = (show: boolean) => {
  showDissolutionText.value = true
  isDissolutionDialogOpen.value = show
}

const dialogTitle = computed<string>(() => {
  return showChangeNotInGoodStandingDialog.value
    ? t('title.dialog.notGoodStanding.notInGoodStanding')
    : businessConfig.value?.dissolutionConfirmation.modalTitle
})

const dissolutionDialogOptions = computed<DialogOptionsI>(() => {
  return {
    title: dialogTitle.value,
    text: '', // content slot is used
    hideClose: false,
    buttons: [] as DialogButtonI[], // button slot is used
    alertIcon: true
  }
})

const showDissolutionText = ref(true)
const showChangeNotInGoodStandingDialog = ref(false)
const setShowChangeNotInGoodStandingDialog = (show: boolean) => {
  showChangeNotInGoodStandingDialog.value = show
}

const closeNotGoodStandingDialog = () => {
  if (isDissolutionDialogOpen.value) {
    setShowDissolutionDialog(false)
  } else {
    setShowChangeNotInGoodStandingDialog(false)
  }
}

/**
 * If business is Not In Good Standing and user isn't staff, emits an event to display NIGS dialog.
 * Otherwise, navigates to Edit UI to create a Special Resolution or Change or Alteration filing.
 */
const promptChangeBusinessInfo = () => {
  if (!currentBusiness.value.goodStanding && !isAuthorized(AuthorizedActionsE.OVERRIDE_NIGS)) {
    // show not good standing popup
    showDissolutionText.value = false
    setShowChangeNotInGoodStandingDialog(true)
    return
  }

  if (!currentBusiness.value.goodStanding && !isAuthorized(AuthorizedActionsE.OVERRIDE_NIGS)) {
    alert('change company info')
    // this.emitNotInGoodStanding(NigsMessage.CHANGE_COMPANY_INFO)
  } else if (currentBusiness.value.legalType === CorpTypeCd.COOP) {
    goToEditUI(`/${currentBusiness.value.identifier}/special-resolution`)
  } else if (isFirm.value) {
    goToEditUI(`/${currentBusiness.value.identifier}/change`)
  } else {
    goToEditUI(`/${currentBusiness.value.identifier}/alteration`)
  }
}

/** Request and Download Business Summary Document. */
const downloadBusinessSummary = async (): Promise<void> => {
  ui.fetchingData = true
  const businessId = currentBusiness.value.identifier
  const { apiURL } = useBcrosLegalApi().getConfig()
  const summaryDocument: DocumentI = {
    title: 'Summary',
    filename: `${businessId} Summary - ${todayIsoDateString()}.pdf`,
    link: `${apiURL}/businesses/${businessId}/documents/summary`
  }
  try {
    const blob = await fetchDocuments(summaryDocument.link)
    if (blob) {
      saveBlob(blob, summaryDocument.filename)
    }
  } catch (error) {
    console.error('Failed to download business summary.', error)
    ui.showDownloadingErrorDialog = true
  }
  ui.fetchingData = false
}

/** Creates a draft filing and navigates to the Create UI to file a company dissolution filing. */
const dissolveBusiness = async (): Promise<void> => {
  const payload = {
    custodialOffice: currentBusinessAddresses.value?.registeredOffice,
    dissolutionType: FilingSubTypeE.DISSOLUTION_VOLUNTARY
  }

  // SP and Partnership use business office instead of registered office
  if (currentBusiness.value.legalType === CorpTypeCd.SOLE_PROP ||
      currentBusiness.value.legalType === CorpTypeCd.PARTNERSHIP) {
    payload.custodialOffice = currentBusinessAddresses.value?.businessOffice
  }

  const response = await filings.createFiling(
    currentBusiness.value,
    FilingTypes.DISSOLUTION,
    payload,
    true
  )

  await new Promise<void>((resolve, reject) => {
    if (response.error?.value) {
      console.error('Filing error', response.error.value)
      reject(new Error('Failed to create filing'))
    } else {
      const filingId = +response.header?.filingId
      if (isNaN(filingId)) {
        console.error('Filing error no filingId')
        reject(new Error('Failed to create filing'))
      }
      if (isFirm.value) {
        goToCreateUI('/define-dissolution', { id: currentBusiness.value.identifier })
      } else {
        goToCreateUI('/dissolution-define-dissolution', { id: currentBusiness.value.identifier })
      }
      resolve()
    }
  })
}

const contacts = getContactInfo('registries')
</script>

<template>
  <div class="flex flex-wrap gap-x-3 gap-y-1 items-center max-w-bcros">
    <!-- Dissolution Confirmation Dialog -->
    <BcrosDialog
      attach="#businessDetails"
      name="confirmDissolution"
      :display="isDissolutionDialogOpen || showChangeNotInGoodStandingDialog"
      :options="dissolutionDialogOptions"
      @close="closeNotGoodStandingDialog"
    >
      <template #content>
        <div v-if="showChangeNotInGoodStandingDialog">
          <p>
            {{ showDissolutionText
              ? $t('text.dialog.notGoodStanding.notGoodStanding1')
              : $t('text.dialog.notGoodStanding.changeNotGoodStanding1')
            }}
          </p>
          <p class="my-4">
            {{ showDissolutionText
              ? $t('text.dialog.notGoodStanding.notGoodStanding2')
              : $t('text.dialog.notGoodStanding.changeNotGoodStanding2')
            }}
          </p>
          <BcrosContactInfo :contacts="contacts" />
        </div>
        <div v-else>
          You are about to {{ businessConfig?.dissolutionConfirmation.dissolutionType }}
          <strong>{{ currentBusinessName || 'this company' }}</strong>;
          once this process is completed and the required documents are filed,
          the {{ businessConfig?.dissolutionConfirmation.entityTitle }} will be
          struck from the register and dissolved, ceasing to be
          {{ businessConfig?.dissolutionConfirmation.subTitle }} under the
          {{ businessConfig?.dissolutionConfirmation.act }} Act.
        </div>
      </template>
      <template #buttons>
        <div v-if="showChangeNotInGoodStandingDialog" class="flex justify-center gap-5">
          <UButton
            variant="outline"
            class="px-10 py-2"
            @click="closeNotGoodStandingDialog"
          >
            {{ $t('button.general.ok') }}
          </UButton>
        </div>
        <div v-else class="flex justify-center gap-5">
          <UButton
            variant="outline"
            class="px-10 py-2"
            @click="closeNotGoodStandingDialog"
          >
            {{ $t('button.general.cancel') }}
          </UButton>
          <UButton
            class="px-10 py-2"
            data-cy="dissolution-button"
            @click="dissolveBusiness"
          >
            {{ businessConfig?.dissolutionConfirmation.confirmButtonText }}
            <UIcon name="i-mdi-chevron-right" class="text-xl" />
          </UButton>
        </div>
      </template>
    </BcrosDialog>

    <!-- Staff Comments -->
    <div v-if="isAuthorized(AuthorizedActionsE.STAFF_COMMENTS) && currentBusiness && !isDisableNonBenCorps()">
      <UModal v-model="isCommentOpen" :ui="{base: 'absolute left-10 top-5 bottom-5'}">
        <BcrosComment :comments="comments" :business="currentBusiness.identifier" @close="showCommentDialog(false)" />
      </UModal>
      <UButton
        id="comment-button"
        small
        text
        variant="ghost"
        class="w-full text-nowrap"
        data-cy="button.comment"
        @click="showCommentDialog(true)"
      >
        <template #leading>
          <UIcon name="i-mdi-message-text-outline" size="small" />
        </template>
        <span class="font-13 ml-1 text-nowrap">{{ $t('label.comments.comment', (comments?.length || 0)) }}</span>
      </UButton>
    </div>

    <!-- COLIN link button -->
    <div
      v-if="!!currentBusinessIdentifier && isDisableNonBenCorps()"
    >
      <BcrosTooltip
        :text="$t('tooltip.filing.button.colinLink')"
        :popper="{
          placement: 'top',
          arrow: true
        }"
      >
        <UButton
          variant="ghost"
          leading-icon="i-mdi-file-document-edit-outline"
          class="w-full text-nowrap"
          data-cy="button.colinLink"
          @click="navigateTo('https://www.corporateonline.gov.bc.ca/', { external: true})"
        >
          <span class="font-13 ml-1">{{ $t('button.tombstone.colinLink') }}</span>
        </UButton>
      </BcrosTooltip>
    </div>

    <!-- View and Change Business Information -->
    <div
      v-if="!isDisableNonBenCorps() &&
        !!currentBusinessIdentifier &&
        currentBusiness.state !== BusinessStateE.HISTORICAL"
    >
      <UButton
        id="business-information-button"
        small
        text
        color="primary"
        variant="ghost"
        :disabled="isChangeBusinessInfoDisabled"
        class="w-full text-nowrap"
        leading-icon="i-mdi-file-document-edit-outline"
        data-cy="button.viewAndChangeBusinessInfo"
        @click="promptChangeBusinessInfo()"
      >
        <span class="font-13 ml-1">{{ $t('button.tombstone.viewAndChangeBusinessInfo') }}</span>
      </UButton>

      <BcrosTooltip
        v-if="isPendingDissolution"
        :text="$t('tooltip.filing.button.isPendingDissolution')"
        :popper="{
          placement: 'top',
          arrow: true
        }"
      >
        <UIcon
          class="pr-2 text-orange-500 text-xl"
          name="i-mdi-alert"
        />
      </BcrosTooltip>
    </div>

    <!-- Download Business Summary -->
    <div v-if="!isDisableNonBenCorps() && isAllowedBusinessSummary">
      <BcrosTooltip
        :text="businessSummaryTooltipText"
        :popper="{
          placement: 'top',
          arrow: true
        }"
      >
        <UButton
          id="download-summary-button"
          small
          text
          :disabled="!isCurrentlyEnabledBusinessSummary"
          variant="ghost"
          class="w-full text-nowrap"
          data-cy="button.downloadSummary"
          @click="downloadBusinessSummary"
        >
          <template #leading>
            <img
              src="@/assets/images/business_summary_icon.svg"
              alt=""
              class="pa-1"
            >
          </template>
          <span class="font-13 ml-1 text-nowrap">{{ $t('button.tombstone.businessSummary') }}</span>
        </UButton>
      </BcrosTooltip>
    </div>

    <div class="mb-2 mt-2">
      <BcrosBusinessDetailsLinkActions
        v-if="!!currentBusinessIdentifier && !isDisableNonBenCorps()"
        @dissolve="setShowDissolutionDialog(true)"
      />
    </div>
  </div>
</template>
