<template>
  <q-btn-dropdown class="q-mx-xs" :icon="Icon" :color="Color" :label="Label" :disable="Disable || AvailableActions.length < 1">
    <q-list class="text-grey-7">
      <q-item v-for="action in AvailableActions" :key="action.label" clickable @click="action.fn">
        <q-item-section avatar>
          <q-icon size="sm" :name="action.icon" />
        </q-item-section>
        <q-item-section>{{ action.label }}</q-item-section>
      </q-item>
    </q-list>
  </q-btn-dropdown>
</template>

<script>
export default {
  name: 'components-common-btndropdown',

  props: {
    Icon: String,
    Disable: Boolean,
    Color: String,
    Label: String,
    AvailableActions: Array,
  }
}
</script>