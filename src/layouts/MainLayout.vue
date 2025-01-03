<template>
  <q-layout view="hHh lpR fFf">

    <q-drawer show-if-above v-model="leftDrawerOpen" side="left" bordered>
      <q-date square today-btn v-model="calendarDate" class="full-width"></q-date>
    </q-drawer>

    <q-page-container>
      <router-view />
    </q-page-container>

  </q-layout>
</template>

<script setup>
import { provide, ref } from 'vue'
import { DateTime } from 'luxon'

const calendarDate = ref(DateTime.now().toFormat('yyyy/MM/dd'))
const leftDrawerOpen = ref(false)
provide('calendarDate', calendarDate)
provide('leftDrawerOpen', leftDrawerOpen)
</script>
