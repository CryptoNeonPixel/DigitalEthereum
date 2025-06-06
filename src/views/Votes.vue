<template>
  <div :class="className" class="w-100">
    <v-card flat color="transparent">
      <v-text-field
        v-model="search"
        :append-inner-icon="mdiMagnify"
        :label="t('votes.search')"
        :class="`${className}__search`"
        single-line
        hide-details
        variant="underlined"
        class="a-input"
        color="primary"
      />
      <div :class="`${className}__info`" v-html="t('votes.stake_info')" />
      <delegates-table
        :page="pagination.page"
        :per-page="pagination.rowsPerPage"
        :search-query="search"
        :waiting-for-confirmation="waitingForConfirmation"
      />
      <v-row :class="`${className}__review`" align="center" justify="space-between" no-gutters>
        <pagination-component v-if="showPagination" v-model="pagination.page" :pages="pages" />

        <v-btn
          :disabled="reviewButtonDisabled"
          class="a-btn-primary ma-2"
          @click="showConfirmationDialog"
        >
          {{ t('votes.summary_title') }}
        </v-btn>
      </v-row>
    </v-card>

    <v-dialog v-model="dialog" width="500">
      <v-card>
        <v-card-title :class="`${className}__dialog-title`">
          {{ t('votes.summary_title') }}
        </v-card-title>

        <v-divider :class="`${className}__divider`" />

        <v-row no-gutters class="pa-4">
          <div :class="`${className}__dialog-summary`">
            {{ t('votes.upvotes') }}: <strong>{{ numOfUpvotes }}</strong
            >,&nbsp; {{ t('votes.downvotes') }}: <strong>{{ numOfDownvotes }}</strong
            >,&nbsp; {{ t('votes.total_new_votes') }}:
            <strong>{{ numOfUpvotes + numOfDownvotes }} / {{ VOTE_REQUEST_LIMIT }}</strong
            >,&nbsp; {{ t('votes.total_votes') }}:
            <strong>{{ totalVotes }} / {{ delegates.length }}</strong>
          </div>
          <!-- eslint-disable vue/no-v-html -- Safe internal content -->
          <div :class="`${className}__dialog-info`" v-html="t('votes.summary_info')" />
          <!-- eslint-enable vue/no-v-html -->
        </v-row>

        <v-card-actions>
          <v-spacer />

          <v-btn variant="text" class="a-btn-regular" @click="dialog = false">
            {{ t('transfer.confirm_cancel') }}
          </v-btn>

          <v-btn variant="text" class="a-btn-regular" @click="sendVotes">
            {{ t('votes.vote_button_text') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script lang="ts" setup>
import PaginationComponent from '@/components/Pagination.vue'
import DelegatesTable from '@/components/DelegatesTable/DelegatesTable.vue'
import { computed, onMounted, ref, reactive, watch } from 'vue'
import { useStore } from 'vuex'
import { useI18n } from 'vue-i18n'
import { mdiMagnify } from '@mdi/js'
import { DelegateDto } from '@/lib/schema/client'

const VOTE_REQUEST_LIMIT = 30

const store = useStore()
const { t } = useI18n()
const search = ref('')
const pagination = reactive({
  rowsPerPage: 50,
  sortBy: 'rank',
  page: 1
})
const waitingForConfirmation = ref(false)
const dialog = ref(false)
const className = 'delegates-view'

interface ExtendedDto extends DelegateDto {
  upvoted?: unknown
  voted?: unknown
  downvoted?: unknown
  _voted?: unknown
}

const delegates = computed(() => {
  const delegates = store.state.delegates.delegates || {}

  return Object.values(delegates) as ExtendedDto[]
})
const numOfUpvotes = computed(() => {
  return delegates.value.filter((delegate) => delegate.upvoted && !delegate.voted).length
})
const numOfDownvotes = computed(() => {
  return delegates.value.filter((delegate) => delegate.downvoted && delegate.voted).length
})
const totalVotes = computed(() => {
  return delegates.value.filter((delegate) => delegate._voted).length
})
const pages = computed(() => {
  if (delegates.value.length <= 0) {
    return 0
  }

  return Math.ceil(delegates.value.length / pagination.rowsPerPage)
})
watch(search, (newValue) => {
  if (newValue.length > 0) {
    pagination.page = 1
  }
})

const showPagination = computed(() => search.value.length === 0)
const reviewButtonDisabled = computed(() => {
  return numOfUpvotes.value + numOfDownvotes.value === 0
})

onMounted(() => {
  store.dispatch('delegates/getDelegates', {
    address: store.state.address
  })
})

const validateVotes = () => {
  if (numOfUpvotes.value + numOfDownvotes.value > VOTE_REQUEST_LIMIT) {
    store.dispatch('snackbar/show', {
      message: t('votes.vote_request_limit', { limit: VOTE_REQUEST_LIMIT })
    })

    return false
  }
  if (store.state.balance < 50) {
    store.dispatch('snackbar/show', {
      message: t('votes.no_money')
    })

    return false
  }

  return true
}
const sendVotes = () => {
  if (!validateVotes()) {
    return
  }

  const upVotes = delegates.value
    .filter((delegate) => delegate.upvoted && !delegate.voted)
    .map((delegate) => `+${delegate.publicKey}`)
  const downVotes = delegates.value
    .filter((delegate) => delegate.downvoted && delegate.voted)
    .map((delegate) => `-${delegate.publicKey}`)
  const allVotes = [...upVotes, ...downVotes]

  if (allVotes.length <= 0) {
    store.dispatch('snackbar/show', {
      message: t('votes.select_delegates')
    })

    return
  }

  waitingForConfirmation.value = true
  dialog.value = false

  store.dispatch('delegates/voteForDelegates', {
    votes: [...upVotes, ...downVotes],
    address: store.state.address
  })

  store.dispatch('snackbar/show', {
    message: t('votes.sent')
  })
}
const showConfirmationDialog = () => {
  dialog.value = true
}
</script>

<style lang="scss" scoped>
@use 'sass:map';
@use '@/assets/styles/settings/_colors.scss';
@use '@/assets/styles/themes/adamant/_mixins.scss';
@use 'vuetify/settings';

.delegates-view {
  position: relative;

  &__body {
    font-size: 14px;
    font-weight: 300;
    padding: 0 16px !important;
  }
  &__dialog-title {
    @include mixins.a-text-header();
  }
  &__dialog-summary {
    @include mixins.a-text-regular-enlarged();
  }
  &__dialog-info {
    @include mixins.a-text-regular-enlarged();
    margin-top: 16px;
    :deep(a) {
      text-decoration-line: none;
      &:hover {
        text-decoration-line: underline;
      }
    }
  }
  &__info {
    @include mixins.a-text-explanation-enlarged();
    padding: 20px 16px !important;
    :deep(a) {
      text-decoration-line: none;
      &:hover {
        text-decoration-line: underline;
      }
    }
  }
  &__review {
    padding-top: 15px !important;
    padding-bottom: 15px !important;
  }
  &__search {
    :deep(.v-field) {
      padding-left: 16px;
      padding-right: 16px;
    }
  }
}

/** Themes **/
.v-theme--light {
  .delegates-view {
    &__body {
      color: map.get(colors.$adm-colors, 'regular');
    }
    &__dialog-title {
      color: map.get(colors.$adm-colors, 'regular');
    }
    &__dialog-summary {
      color: map.get(colors.$adm-colors, 'regular');
    }
    &__dialog-info {
      color: map.get(colors.$adm-colors, 'regular');
    }
    &__divider {
      border-color: map.get(colors.$adm-colors, 'regular');
    }
    :deep(.v-table) tbody tr:not(:last-child) {
      border-bottom:
        1px solid,
        map.get(colors.$adm-colors, 'secondary2');
    }
  }
}
.v-theme--dark {
  .delegates-view {
  }
}
</style>
