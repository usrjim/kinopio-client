<template lang="pug">
.select-all-below(v-if="isVisible" :style="{ top: positionY + 'px' }")
  .badge.label-badge(:style="{ 'background-color': userColor }" @mousedown="handleMouseDown")
    img.icon(src="@/assets/brush-y.svg")
    .pointer(:style="{ 'background-color': userColor }" :class="{ wide: isSelecting }")
</template>

<script>
import utils from '@/utils.js'

import debounce from 'lodash-es/debounce'

export default {
  name: 'SelectAllBelow',
  mounted () {
    window.addEventListener('mousemove', this.handleMouseMove)
    window.addEventListener('mouseup', this.handleMouseUp)
  },
  beforeUnmount () {
    window.removeEventListener('mousemove', this.handleMouseMove)
    window.removeEventListener('mouseup', this.handleMouseUp)
  },
  data () {
    return {
      isVisible: false,
      isSelecting: false,
      positionY: 250
    }
  },
  computed: {
    userColor () { return this.$store.state.currentUser.color },
    canEditSpace () { return this.$store.getters['currentUser/canEditSpace']() }
  },
  methods: {
    handleMouseDown (event) {
      this.isSelecting = true
      this.selectAllBelow(event)
    },
    handleMouseMove (event) {
      if (!this.canEditSpace) { return }
      const edgeThreshold = 45
      const header = document.querySelector('header').getBoundingClientRect().height + 10
      const footer = document.querySelector('.footer-wrap').getBoundingClientRect().height + 10
      const position = utils.cursorPositionInViewport(event)
      const viewport = utils.visualViewport()
      const isInThreshold = position.x <= edgeThreshold
      const isBetweenControls = utils.isBetween({
        value: position.y,
        min: header,
        max: viewport.height - footer
      })
      const isInPosition = isInThreshold && isBetweenControls
      if (isInPosition || this.isSelecting) {
        this.positionY = position.y
        this.isVisible = true
      } else {
        this.isVisible = false
      }
      if (this.isSelecting) {
        this.debouncedSelectAllBelow(event)
      }
    },
    handleMouseUp (event) {
      if (!this.isSelecting) { return }
      this.isSelecting = false
      this.selectAllBelow(event)
      this.isVisible = false
      setTimeout(() => {
        this.$store.commit('preventMultipleSelectedActionsIsVisible', false)
      }, 100)
    },
    debouncedSelectAllBelow: debounce(function (event) {
      this.selectAllBelow(event)
    }, 10, { leading: true }),
    selectAllBelow (event) {
      let position = utils.cursorPositionInPage(event)
      this.$store.commit('preventMultipleSelectedActionsIsVisible', true)
      this.$store.commit('triggerSelectAllCardsBelowCursor', position)
    }
  }
}
</script>

<style lang="stylus">
.select-all-below
  position fixed
  left 0
  top 250px
  pointer-events all
  cursor pointer
  .badge
    border-radius 0
    border-top-right-radius 6px
    border-bottom-right-radius 6px
    padding 0
    margin 0
    position relative
    box-shadow 0
    margin-bottom 8px
    &:hover
      box-shadow var(--button-hover-shadow)
    &:active
      box-shadow var(--button-active-inset-shadow)

  img
    padding 3px
    margin-left 6px
    margin-right 8px

  .pointer
    position absolute
    background-color var(--primary)
    height 1px
    width 12px
    left 30px
    top 10px
    &.wide
      width 100vw
</style>