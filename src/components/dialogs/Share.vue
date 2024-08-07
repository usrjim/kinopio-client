<script setup>
import { reactive, computed, onMounted, defineProps, defineEmits, watch, ref, nextTick } from 'vue'
import { useStore } from 'vuex'

import PrivacyButton from '@/components/PrivacyButton.vue'
import Invite from '@/components/Invite.vue'
import RssFeeds from '@/components/dialogs/RssFeeds.vue'
import Embed from '@/components/dialogs/Embed.vue'
import UserList from '@/components/UserList.vue'
import utils from '@/utils.js'
import ImportExport from '@/components/dialogs/ImportExport.vue'
import AddToExplore from '@/components/AddToExplore.vue'
import AskToAddToExplore from '@/components/AskToAddToExplore.vue'
import ReadOnlySpaceInfoBadges from '@/components/ReadOnlySpaceInfoBadges.vue'
import consts from '@/consts.js'
const store = useStore()

const dialog = ref(null)

onMounted(() => {
  store.subscribe((mutation, state) => {
    if (mutation.type === 'updatePageSizes') {
      updateDialogHeight()
    }
  })
})

const props = defineProps({
  visible: Boolean
})

watch(() => props.visible, (value, prevValue) => {
  store.commit('clearNotificationsWithPosition')
  closeDialogs()
  if (value) {
    updateDialogHeight()
    store.commit('shouldExplicitlyHideFooter', true)
    store.dispatch('currentSpace/createSpacePreviewImage')
  } else {
    store.commit('shouldExplicitlyHideFooter', false)
  }
})

const state = reactive({
  urlIsCopied: false,
  privacyPickerIsVisible: false,
  dialogHeight: null,
  rssFeedsIsVisible: false,
  embedIsVisible: false,
  importExportIsVisible: false,
  isShareInPresentationMode: false,
  emailInvitesIsVisible: false
})

const isSecureAppContextIOS = computed(() => consts.isSecureAppContextIOS)
const currentUserIsSignedIn = computed(() => store.getters['currentUser/isSignedIn'])
const isSpaceMember = computed(() => store.getters['currentUser/isSpaceMember']())
const spaceIsRemote = computed(() => store.getters['currentSpace/isRemote'])
const spaceIsPublic = computed(() => store.state.currentSpace.privacy !== 'private')
const spaceIsPrivate = computed(() => store.state.currentSpace.privacy === 'private')

// add to explore

const exploreSectionIsVisible = computed(() => {
  const showInExplore = store.state.currentSpace.showInExplore
  const shouldShowAskToAddToExplore = !isSpaceMember.value && !showInExplore
  return spaceIsPublic.value && (isSpaceMember.value || shouldShowAskToAddToExplore)
})

// collaborators

const selectedUser = computed(() => {
  const userDetailsIsVisible = store.state.userDetailsIsVisible
  if (!userDetailsIsVisible) { return }
  return store.state.userDetailsUser
})
const spaceCollaborators = computed(() => store.state.currentSpace.collaborators)
const spaceHasCollaborators = computed(() => {
  const collaborators = store.state.currentSpace.collaborators
  return Boolean(collaborators.length)
})
const spaceOtherCardUsers = computed(() => {
  const currentUserId = store.state.currentUser.id
  const collaborators = store.state.currentSpace.collaborators
  let users = store.getters['currentCards/users']
  users = users.filter(user => Boolean(user))
  // remove currentUser
  users = users.filter(user => user.id !== currentUserId)
  // remove collaborators
  users = users.filter(user => {
    const isCollaborator = spaceCollaborators.value.find(collaborator => {
      return collaborator.id === user.id
    })
    return !isCollaborator
  })
  return users
})
const spaceHasOtherCardUsers = computed(() => Boolean(spaceOtherCardUsers.value.length))
const toggleUserDetails = (event, user) => {
  closeDialogs()
  showUserDetails(event, user)
}
const showUserDetails = (event, user) => {
  let element = event.target
  let options = { element, offsetX: 25, shouldIgnoreZoom: true }
  let position = utils.childDialogPositionFromParent(options)
  store.commit('userDetailsUser', user)
  store.commit('userDetailsPosition', position)
  store.commit('userDetailsIsVisible', true)
}
const removeCollaborator = async (user) => {
  store.dispatch('currentSpace/removeCollaboratorFromSpace', user)
  const isCurrentUser = store.state.currentUser.id === user.id
  if (isCurrentUser) {
    store.dispatch('closeAllDialogs')
  }
  closeDialogs()
}

// copy url

const spaceUrl = computed(() => {
  let url = store.getters['currentSpace/url']
  url = new URL(url)
  if (state.isShareInPresentationMode) {
    url.searchParams.set('present', true)
  }
  return url.href
})
const copySpaceUrl = async (event) => {
  store.commit('clearNotificationsWithPosition')
  const position = utils.cursorPositionInPage(event)
  try {
    await navigator.clipboard.writeText(spaceUrl.value)
    store.commit('addNotificationWithPosition', { message: 'Copied', position, type: 'success', layer: 'app', icon: 'checkmark' })
  } catch (error) {
    console.warn('🚑 copyText', error)
    store.commit('addNotificationWithPosition', { message: 'Copy Error', position, type: 'danger', layer: 'app', icon: 'cancel' })
  }
}
const webShareIsSupported = computed(() => navigator.share)
const webShare = () => {
  const spaceName = store.state.currentSpace.name
  const data = {
    title: 'Kinopio Space',
    text: spaceName,
    url: spaceUrl.value
  }
  navigator.share(data)
}

// dialog

const updateDialogHeight = () => {
  if (!props.visible) { return }
  nextTick(() => {
    let element = dialog.value
    state.dialogHeight = utils.elementHeight(element)
  })
}
const dialogIsVisible = computed(() => {
  return state.privacyPickerIsVisible || state.rssFeedsIsVisible || state.embedIsVisible || state.importExportIsVisible || state.emailInvitesIsVisible
})
const closeDialogs = () => {
  state.privacyPickerIsVisible = false
  state.rssFeedsIsVisible = false
  state.embedIsVisible = false
  state.importExportIsVisible = false
  state.emailInvitesIsVisible = false
  store.commit('userDetailsIsVisible', false)
  store.commit('triggerCloseChildDialogs')
}

// toggles

const triggerSignUpOrInIsVisible = () => {
  store.dispatch('closeAllDialogs')
  store.commit('triggerSignUpOrInIsVisible')
}
const togglePrivacyPickerIsVisible = () => {
  const isVisible = state.privacyPickerIsVisible
  closeDialogs()
  state.privacyPickerIsVisible = !isVisible
}
const toggleRssFeedsIsVisible = () => {
  const isVisible = state.rssFeedsIsVisible
  closeDialogs()
  state.rssFeedsIsVisible = !isVisible
}
const toggleEmbedIsVisible = () => {
  const isVisible = state.embedIsVisible
  closeDialogs()
  state.embedIsVisible = !isVisible
}
const toggleImportExportIsVisible = () => {
  const isVisible = state.importExportIsVisible
  closeDialogs()
  state.importExportIsVisible = !isVisible
}
const toggleIsShareInPresentationMode = () => {
  closeDialogs()
  state.isShareInPresentationMode = !state.isShareInPresentationMode
}
const emailInvitesIsVisible = (value) => {
  state.emailInvitesIsVisible = value
}
</script>

<template lang="pug">
dialog.share.wide(v-if="props.visible" :open="props.visible" @click.left.stop="closeDialogs" ref="dialog" :style="{'max-height': state.dialogHeight + 'px'}" :class="{overflow: !dialogIsVisible}")
  section
    .row.title-row
      p Share
      .row
        .button-wrap(v-if="spaceIsRemote")
          button.small-button(@click.left.stop="toggleRssFeedsIsVisible" :class="{ active: state.rssFeedsIsVisible }" title="RSS Feeds")
            span RSS
          RssFeeds(:visible="state.rssFeedsIsVisible")

  section(v-if="spaceIsRemote")
    ReadOnlySpaceInfoBadges
    PrivacyButton(:privacyPickerIsVisible="state.privacyPickerIsVisible" :showDescription="true" @togglePrivacyPickerIsVisible="togglePrivacyPickerIsVisible" @closeDialogs="closeDialogs")

    //- Copy URL
    section.subsection(:class="{'share-url-subsection-member': isSpaceMember}")
      .row
        .segmented-buttons
          button(@click.left="copySpaceUrl")
            img.icon.copy(src="@/assets/copy.svg")
            .badge.badge-in-button.danger.private-copy-badge(v-if="spaceIsPrivate")
              img.icon.lock(src="@/assets/lock.svg")
            span Copy URL
          button(v-if="webShareIsSupported" @click="webShare")
            img.icon.share(src="@/assets/share.svg")

        label.label.small-button.extra-options-button.inline-button(title="Share in Presentation Mode" @mouseup.left="toggleIsShareInPresentationMode" @touchend.prevent="toggleIsShareInPresentationMode" :class="{active: state.isShareInPresentationMode}")
          input(type="checkbox" :value="state.isShareInPresentationMode")
          img.icon(src="@/assets/presentation.svg")
      //- Explore
      template(v-if="exploreSectionIsVisible")
        .row
          p Share with the Community
        .row
          AddToExplore
          AskToAddToExplore

  //- Invite
  Invite(v-if="isSpaceMember && currentUserIsSignedIn" @closeDialogs="closeDialogs" @emailInvitesIsVisible="emailInvitesIsVisible")
  //- Collaborators
  section.results-section.collaborators(v-if="spaceHasCollaborators || spaceHasOtherCardUsers")
    //- collaborators
    template(v-if="spaceHasCollaborators")
      UserList(:users="spaceCollaborators" :selectedUser="selectedUser" @selectUser="toggleUserDetails" :showRemoveUser="isSpaceMember" @removeUser="removeCollaborator" :isClickable="true")
    //- card users
    template(v-if="spaceHasOtherCardUsers")
      UserList(:users="spaceOtherCardUsers" :selectedUser="selectedUser" @selectUser="toggleUserDetails" :isClickable="true")

  section(v-if="!spaceIsRemote")
    p
      span To share or invite collaborators,
      span.badge.info you need to Sign Up or In
      span for your spaces to be synced and accessible anywhere.
    button(@click.left="triggerSignUpOrInIsVisible") Sign Up or In

  //- Import, Export, Embed
  section.import-export-section
    .row
      .segmented-buttons(@click.stop)
        button(@click.left.stop="toggleImportExportIsVisible" :class="{ active: state.importExportIsVisible }")
          span Import or Export
          ImportExport(:visible="state.importExportIsVisible")
      //- Embed
      .button-wrap
        button(@click.left.stop="toggleEmbedIsVisible" :class="{ active: state.embedIsVisible }")
          span Embed
        Embed(:visible="state.embedIsVisible")

</template>

<style lang="stylus">
dialog.share
  top calc(100% - 8px)
  left initial
  right 8px
  max-height calc(100vh - 25px)
  &.overflow
    overflow auto
  .badge
    display inline-block
    &.danger
      background-color var(--danger-background)
  .privacy-button + .textarea
    margin-top 10px
  .description
    margin-top 3px
  dialog.privacy-picker,
  dialog.dialog-wrap
    left initial
    right 8px
  dialog.user-details
    left initial
    right calc(100% - 20px)
  dialog.import-export
    top calc(100% - 8px)
    left initial
    right 8px

  .collaborators
    max-height calc(100vh - 200px)
  .share-private
    margin-top 10px
  .privacy-button + input
    margin-top 10px
  .privacy-button
    > button
      padding-top 8px
  .user
    vertical-align -3px
  .button-tip-badge
    top -12px
    pointer-events none
  p + .subsection
    margin-top 10px
  .share-url-subsection-member
    margin-top 0
    border-top-left-radius 0
    border-top-right-radius 0

  @media(max-height 670px)
    dialog.import-export,
    dialog.embed
      top -50px

  @media(max-height 500px)
    dialog.import-export,
    dialog.embed
      top -200px

  .segmented-buttons
    z-index 1
  .extra-options-button
    margin-left auto
    margin-top 0
    margin-bottom -2px
    width auto
    cursor pointer
    height 20px
    width initial
    &.label
      padding-top 2px
      input
        background-color transparent
        pointer-events none

  .privacy-button
    button
      border-bottom-left-radius 0
      border-bottom-right-radius 0

  .private-copy-badge
    margin-left 6px
    vertical-align 1px
  .title-row
    p + .row
      margin-top 0

  .import-export-section
    border-top 1px solid var(--primary-border)
</style>
