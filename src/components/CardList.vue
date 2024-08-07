<template lang="pug">
span
  ul.results-list.card-list(ref="resultsList")
    template(v-for="card in normalizedCards" :key="card.id")
      li(@click.stop="selectCard(card)" :data-card-id="card.id" :class="{active: cardIsActive(card), hover: cardIsFocused(card)}")
        //- date
        span.badge.status.inline-badge
          img.icon.time(src="@/assets/time.svg")
          span {{ relativeDate(card) }}
        //- user
        UserLabelInline(v-if="userIsNotCurrentUser(card.user.id)" :user="card.user")
        //- name
        span.card-info(:class="{ badge: card.backgroundColor, 'is-dark': colorIsDark(card) }" :style="styles(card)")
          template(v-for="segment in card.nameSegments")
            img.card-image(v-if="segment.isImage" :src="segment.url")
            img.card-image(v-if="urlPreviewImage(card)" :src="urlPreviewImage(card)")
            NameSegment(:segment="segment" :search="search" :isStrikeThrough="isStrikeThrough(card)")
          //- remove
          button.small-button.remove-button.danger(v-if="cardsShowRemoveButton" @click.left.stop="removeCard(card)")
            img.icon(src="@/assets/remove.svg")
          //- loading
          Loader(:visible="card.isLoading")

</template>

<script>
import UserLabelInline from '@/components/UserLabelInline.vue'
import NameSegment from '@/components/NameSegment.vue'
import Loader from '@/components/Loader.vue'
import utils from '@/utils.js'
import cache from '@/cache.js'

import { mapState, mapGetters } from 'vuex'

export default {
  name: 'ComponentName',
  components: {
    UserLabelInline,
    NameSegment,
    Loader
  },
  props: {
    cards: Array,
    search: String,
    cardsShowRemoveButton: Boolean
  },
  created () {
    this.$store.subscribe((mutation, state) => {
      if (mutation.type === 'triggerRemoveCardFromCardList') {
        const card = mutation.payload
        this.removedCardIds.push(card.id)
      }
    })
  },
  data () {
    return {
      removedCardIds: []
    }
  },
  computed: {
    ...mapState([
      'cardDetailsIsVisibleForCardId',
      'previousResultItem',
      'currentUser'
    ]),
    ...mapGetters([
    ]),
    normalizedCards () {
      let cards = utils.clone(this.cards)
      cards = cards.filter(card => !this.removedCardIds.includes(card.id))
      return cards.map(card => {
        card = this.$store.getters['currentCards/nameSegments'](card)
        card.user = this.$store.getters['currentSpace/userById'](card.userId)
        if (!card.user) {
          card.user = {
            id: '',
            name: '',
            color: undefined
          }
        }
        return card
      })
    }
  },
  methods: {
    urlPreviewImage (card) {
      if (!card.urlPreviewIsVisible) { return }
      return card.urlPreviewImage
    },
    selectCard (card) {
      this.$emit('selectCard', card)
    },
    removeCard (card) {
      this.$emit('removeCard', card)
    },
    cardIsActive (card) {
      const isCardDetailsVisible = this.cardDetailsIsVisibleForCardId === card.id
      return isCardDetailsVisible || card.isLoading
    },
    cardIsFocused (card) {
      return this.previousResultItem.id === card.id
    },
    relativeDate (card) {
      return utils.shortRelativeTime(card.nameUpdatedAt || card.updatedAt)
    },
    userIsNotCurrentUser (userId) {
      return this.currentUser.id !== userId
    },
    isStrikeThrough (card) {
      return card.name.startsWith('[x]')
    },
    colorIsDark (card) {
      if (!card.backgroundColor) { return }
      return utils.colorIsDark(card.backgroundColor)
    },
    styles (card) {
      return {
        backgroundColor: card.backgroundColor
      }
    }
  }
}
</script>

<style lang="stylus">
.card-list
  li
    position relative
    display block !important
    .button-badge
      box-shadow none
      display initial
      margin-right 0
      pointer-events none
      &:hover,
      &:active
        box-shadow none
    img
      max-width 48px
      border-radius var(--small-entity-radius)
      vertical-align middle
  .time
    vertical-align 0
    height 11px
  .inline-badge
    display inline-block
  .remove-button
    position absolute
    top 7px
    right 4px
    .icon
      vertical-align 0
  .loader
    position absolute
    top 6px
    left 8px
  .card-info
    &.badge
      position initial
      &.is-dark
        color var(--primary-on-dark-background)
</style>
