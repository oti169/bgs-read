<template>
  <q-page class="fit q-pa-sm">
    <div v-if="stillLoading" class="q-pa-lg">
      <q-linear-progress indeterminate></q-linear-progress>
    </div>
    <div v-else>
      <div v-if="missingYear" class="q-pa-lg">
        Sorry, this app doesn't know about BGS reading information for this year.
      </div>
      <q-list v-else>
        <q-item>
          <q-item-section>
            <q-item-label overline>Date</q-item-label>
            <q-item-label>{{ dayInfo.day }}, {{ thisYear }}</q-item-label>
          </q-item-section>
        </q-item>
        <q-item v-for="rb in ['psalm', 'ntr']" :key="`rb-${rb}`">
          <q-item-section>
            <q-item-label overline>{{ readingBits[rb] }}</q-item-label>
            <q-item-label>{{ dayInfo[rb] }}</q-item-label>
            <q-item-label><a
              v-for="tr in translations" :key="`rb-${rb}-${tr}`"
              :href="bgLink(dayInfo[rb], tr)"
              class="external-link q-pr-sm"
              rel="noopener noreferrer"
            >{{ tr }}</a></q-item-label>
          </q-item-section>
        </q-item>
        <q-item v-for="ib in ['intro', 'tbp', 'ex1', 'ex2', 'tmm', 'morg']" :key="ib">
          <q-item-section>
            <q-item-label overline class="item-heading">{{ itemBits[ib] }}</q-item-label>
            <q-item-label>
              <span v-if="dayInfo[ib]" v-html="dayInfo[ib]"></span>
              <span v-else class="text-grey"><em>N/A</em></span>
            </q-item-label>
          </q-item-section>
        </q-item>
      </q-list>
      <div class="row q-col-gutter-sm">
        <div class="col-4">
          <q-btn id="btn-prev-day" outline square class="col-4 full-width bgs-nav" color="primary" @click="changeDay(-1)">
            <div class="q-pt-xs">
              <q-icon class="fix-icon-align" name="mdi-skip-previous" size="xs"></q-icon>{{ prevDay }}
            </div>
          </q-btn>
        </div>
        <div class="col-4">
          <q-btn id="btn-this-day" outline square class="col-4 full-width bgs-nav" color="primary" @click="loadToday()">
            <div class="q-pt-xs">Today</div>
          </q-btn>
        </div>
        <div class="col-4">
          <q-btn id="btn-next-day" outline square class="col-4 full-width bgs-nav" color="primary" @click="changeDay(1)">
            <div class="q-pt-xs">
              {{ nextDay }}<q-icon class="fix-icon-align" name="mdi-skip-next" size="xs"></q-icon>
            </div>
          </q-btn>
        </div>
      </div>
    </div>
    <div class="item-heading q-pt-md">Links</div>
    <div class="q-pt-sm">
      <a class="external-link" href="slack://channel?team=TQC64RB70&id=C069MRSEK2T">Slack: BGS Alumni / 2024-bgs-new-testament-leaders</a>
    </div>
    <div class="q-pt-sm">
      <a class="external-link" href="https://www.becomegoodsoil.com">Become Good Soil</a>
    </div>
    <div class="q-pt-sm">
      <a class="external-link" href="https://becomegoodsoil.com/becoming-a-king/">Becoming a King</a>
    </div>
    <div class="q-pt-sm">
      <a class="external-link" :href="spreadsheetLocns[thisYear]">Source sheet</a>
    </div>
  </q-page>
</template>

<script setup>
import { computed, inject, onMounted, ref, watch } from 'vue'
import axios from 'axios'
import { DateTime } from 'luxon'

const spreadsheetLocns = {
  2023: 'https://docs.google.com/spreadsheets/d/1Yo1Wj3SbbN_xIZibhsCwy2bvG77Syj_hNWZH6QbmEVw/htmlview',
  2024: 'https://docs.google.com/spreadsheets/d/1zs3x5trgorscFHxuy0P9Fmq8ZvUwJcC-hQkRYBa3EK8/htmlview'
}
const stillLoading = ref(true)
const missingYear = ref(false)
const sheetData = ref([])
const calendarDate = inject('calendarDate')

const readingBits = {
  psalm: 'Psalms/Proverbs',
  ntr: 'NT reading'
}
const itemBits = {
  intro: 'Eugene Petersen',
  tbp: 'Bible Project',
  ex1: 'Excavation 1',
  ex2: 'Excavation 2',
  tmm: 'Tim Mackie Matthew',
  morg: 'Morgan notes'
}
const translations = ['NLT', 'NIV', 'ESV', 'MSG', 'NKJV', 'KJV', 'AMP', 'NASB']

const dayInfo = computed(() => {
  const day = DateTime.fromFormat(calendarDate.value, 'yyyy/MM/dd').toFormat('MMM d')
  let matchingInfo = sheetData.value.filter(sd => sd.day === day)
  if (matchingInfo.length === 0) {
    const altDay = DateTime.fromFormat(calendarDate.value, 'yyyy/MM/dd').toFormat('MMMM d')
    matchingInfo = sheetData.value.filter(sd => sd.day === altDay)
  }
  return matchingInfo.length > 0 ? matchingInfo[0] : {}
})

const prevDay = computed(() => {
  return DateTime.fromFormat(calendarDate.value, 'yyyy/MM/dd').minus({ days: 1 }).toFormat('MMM d')
})

const nextDay = computed(() => {
  return DateTime.fromFormat(calendarDate.value, 'yyyy/MM/dd').plus({ days: 1 }).toFormat('MMM d')
})

const thisYear = computed(() => {
  return DateTime.fromFormat(calendarDate.value, 'yyyy/MM/dd').year
})

watch(thisYear, val => {
  loadSheetData(thisYear.value)
})

onMounted(async () => {
  await loadSheetData(thisYear.value)
})

function bgLink (bibleRef, translation) {
  return `https://www.biblegateway.com/passage/?search=${bibleRef}&version=${translation}`
}

async function loadSheetData (year) {
  if (!spreadsheetLocns[year]) {
    missingYear.value = true
    return
  }
  missingYear.value = false
  stillLoading.value = true
  const htmlContent = await axios.get(spreadsheetLocns[year])
  const parser = new DOMParser()
  const doc = parser.parseFromString(htmlContent.data, 'text/html')
  const table = doc.querySelector('table.waffle')
  const rows = table.querySelectorAll('tr')
  sheetData.value.length = 0
  for (let idx = 5; idx < rows.length; idx += 1) {
    const cells = rows[idx].querySelectorAll('td')
    sheetData.value.push({
      day: cells[0].textContent.trim(), // Date
      psalm: cells[2].textContent.trim(), // Psalm
      ntr: cells[3].textContent.trim(), // NT reading
      intro: cells[4].innerHTML.trim(), // Eugene Petersen
      tbp: cells[5].innerHTML, // Bible Project
      ex1: cells[6].innerHTML, // Excavation 1
      ex2: cells[7].innerHTML, // Excavation 2
      tmm: cells[8].innerHTML, // Tim Mackie Matthew
      morg: cells[9].innerHTML // Morgan notes
    })
  }
  stillLoading.value = false
}

function changeDay (days) {
  calendarDate.value = DateTime.fromFormat(calendarDate.value, 'yyyy/MM/dd').plus({ days }).toFormat('yyyy/MM/dd')
}

function loadToday () {
  calendarDate.value = DateTime.now().toFormat('yyyy/MM/dd')
}
</script>

<style>
  #q-app > div > div.q-drawer-container > aside > div > div > div.q-date__header > div:nth-child(1) > div {
    display: none
  }
  a,
  .external-link,
  div.q-item__label > span > a {
    color: rgb(121, 90, 60);
    text-decoration: none;
    cursor: pointer;
    font-size: 18px !important;
  }
  .external-link:hover,
  div.q-item__label > span > a:hover {
    text-decoration: underline;
  }
  .item-heading {
    text-transform: uppercase;
    font-weight: bold;
  }
  .fix-icon-align {
    margin-top: -4px;
  }
</style>
