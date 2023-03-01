<script setup>
import MultipleConnectionsPicker from '@/components/dialogs/MultipleConnectionsPicker.vue'
import ConnectionDecorators from '@/components/ConnectionDecorators.vue'

import uniq from 'lodash-es/uniq'
import uniqBy from 'lodash-es/uniqBy'
import { reactive, computed, onMounted, defineProps, defineEmits, watch } from 'vue'
import { useStore } from 'vuex'
const store = useStore()

const props = defineProps({
  visible: Boolean,
  connections: Array,
  canEditAll: Object
})
const emit = defineEmits(['closeDialogs'])
const state = reactive({
  multipleConnectionsPickerVisible: false
})

const toggleMultipleConnectionsPickerVisible = () => {
  const isVisible = state.multipleConnectionsPickerVisible
  closeDialogs()
  state.multipleConnectionsPickerVisible = !isVisible
}

// connection types

const connectionTypes = computed(() => {
  let types = uniq(store.state.multipleConnectionsSelectedIds)
  types = types.map(id => {
    const connection = store.getters['currentConnections/byId'](id)
    if (!connection) { return }
    return store.getters['currentConnections/typeByTypeId'](connection.connectionTypeId)
  })
  types = types.filter(type => Boolean(type))
  types = uniqBy(types, 'id')
  types = uniqBy(types, 'color')
  return types
})
const editableConnectionTypes = computed(() => {
  return uniq(props.connections.map(connection => {
    return store.getters['currentConnections/typeByTypeId'](connection.connectionTypeId)
  }))
})

// utils

const closeDialogs = () => {
  state.multipleConnectionsPickerVisible = false
  emit('closeDialogs')
}

</script>

<template lang="pug">
section.subsection.connection-actions(v-if="visible")
  //- Edit Connections
  .row.edit-connection-types
    //- Type Color
    .button-wrap
      button.change-color(:disabled="!canEditAll.connections" @click.left.stop="toggleMultipleConnectionsPickerVisible" :class="{active: state.multipleConnectionsPickerVisible}")
        //- img.icon.connection-path(src="@/assets/connection-path.svg")
        .segmented-colors.icon
          template(v-for="type in connectionTypes")
            .current-color(:style="{ background: type.color }")
        span Type
      MultipleConnectionsPicker(:visible="state.multipleConnectionsPickerVisible" :selectedConnections="connections" :selectedConnectionTypes="editableConnectionTypes")
    //- Arrows or Label
    ConnectionDecorators(:connections="connections")
</template>

<style lang="stylus">
dialog section.connection-actions
  padding 4px
  padding-bottom 0
  background-color transparent
  border 1px solid var(--primary-border)
  padding 4px
  padding-bottom 0
  .button-wrap
    margin-left 0
    margin-right 4px
    vertical-align middle
    margin-bottom 10px
</style>