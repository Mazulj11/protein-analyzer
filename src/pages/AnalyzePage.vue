<template>
  <q-page>

    <div class="text-h4 q-pa-lg">Analiza Proteina</div>
    <div class="text-subtitle1 q-pl-lg q-mb-lg">Unesite FASTA sekvence ili učitajte fajl</div>

    <q-form @submit.prevent="submitForm" class="q-pa-md">
      <div>
        <div class="field">
          <label class="text-weight-bold">Kriterij:</label>
          <q-select v-model="criteri" :options="['ident','sim']" borderless />
        </div>
        <div class="field">
          <label class="text-weight-bold">Prag:</label>
          <q-input v-model.number="limit" type="number" borderless />
        </div>
        <div class="field">
          <label class="text-weight-bold">Unesi FASTA sekvence:</label>
          <q-input v-model="textInput" type="textarea" borderless autogrow />
        </div>
        <div class="field">
          <label class="text-weight-bold">Učitaj FASTA fajl:</label>
          <q-file v-model="file" borderless accept=".fasta,.fa,.txt" class="bg-white border-radius" />
        </div>
      </div>

      <div class="row justify-center">
        <q-btn
          v-if="!loading"
          label="Analiziraj"
          type="submit"
          class="full-width"
          color="primary"
          style="
            color: white;
            height: 56px;
            font-size: 16px;
            border-radius: 10px;
            box-shadow: rgba(0, 0, 0, 0.3) 7px 7px 0px, rgba(0, 0, 0, 0.5) 0px 5px 5px;
          "
        />
        <q-circular-progress
          v-else
          indeterminate
          size="75px"
          :thickness="0.6"
          color="primary"
          center-color="grey-8"
          class="q-ma-md"
        />
      </div>
    </q-form>

    <q-card
      v-if="result?.status === 'success'"
      class="q-ma-lg q-pa-md"
      style="border-radius: 10px; box-shadow: rgba(0, 0, 0, 0.3) 7px 7px 0px, rgba(0, 0, 0, 0.5) 0px 5px 5px;"
     >
      <q-card-section>
        <div class="row justify-between items-center text-h5 q-mb-lg">
          <label>Rezultat</label>
          <q-btn label="Export u CSV" dense icon="download" color="secondary" @click="exportCsv" style="font-size: 12px;" />
        </div>


        <q-list bordered separator class="q-mt-md">
          <q-item><q-item-section>Trajanje:</q-item-section><q-item-section side>{{ result.result.trajanje }}</q-item-section></q-item>
          <q-item><q-item-section>Broj proteina:</q-item-section><q-item-section side>{{ result.result.broj_proteina }}</q-item-section></q-item>
          <q-item><q-item-section>Dimenzija matrice:</q-item-section><q-item-section side>{{ result.result.dimenzija_matrice }}</q-item-section></q-item>
          <q-item><q-item-section>Broj jedinica u matrici:</q-item-section><q-item-section side>{{ result.result.num_ones_in_matrix }}</q-item-section></q-item>
          <q-item><q-item-section>Gustoća matrice:</q-item-section><q-item-section side>{{ result.result.matrix_density }}</q-item-section></q-item>
          <q-item><q-item-section>Prosječna duljina:</q-item-section><q-item-section side>{{ result.result.avg_length }}</q-item-section></q-item>
        </q-list>

        <div class="q-mt-md">
          <div class="text-subtitle2">Prvih 5 ID-jeva:</div>
          <q-chip v-for="id in result.result.first_ids" :key="id" color="primary" text-color="white" class="q-mr-sm q-mb-sm">
            {{ id }}
          </q-chip>
        </div>

        <div class="q-mt-md">
          <div class="text-subtitle2">Prve 3 sekvence:</div>
          <q-card flat bordered class="q-mt-xs q-pa-sm bg-grey-2" v-for="(seq, idx) in result.result.first_sequences" :key="idx">
            <code>{{ seq }}</code>
          </q-card>
        </div>

        <div class="q-mt-md">
          <div class="text-subtitle2">Prvih 5 ID-jeva u tablici:</div>
          <q-table
            :rows="result.result.first_ids.map((id, i) => ({ id, seq: result.result.first_sequences[i] || 'N/A' }))"
            :columns="tableColumns"
            row-key="id"
            flat dense bordered
            class="q-mt-sm"
          />
        </div>

        <div class="q-mt-md">
          <div class="text-subtitle2 q-mb-sm">Vizualizacija:</div>
          <canvas ref="chartCanvas" height="150"></canvas>
        </div>

      </q-card-section>
    </q-card>

  </q-page>
</template>

<script setup>
import { ref, watch, nextTick } from 'vue'
import { Notify } from 'quasar'
import { Chart, BarController, BarElement, CategoryScale, LinearScale, Tooltip, Legend } from 'chart.js'

Chart.register(BarController, BarElement, CategoryScale, LinearScale, Tooltip, Legend)

const criteri = ref('ident')
const limit = ref(20)
const textInput = ref('')
const file = ref(null)
const loading = ref(false)
const result = ref(null)
const chartCanvas = ref(null)
let chartInstance = null

const tableColumns = [
  { name: 'id', label: 'ID', field: 'id', align: 'left' },
  { name: 'seq', label: 'Sekvenca', field: 'seq', align: 'left' }
]

const submitForm = async () => {
  loading.value = true
  result.value = null

  try {
    const formData = new FormData()
    formData.append('criteri', criteri.value)
    formData.append('limit', limit.value)
    if (textInput.value) formData.append('text_input', textInput.value)
    if (file.value) formData.append('file', file.value)

    const response = await fetch('https://backend-bioinformatika.onrender.com/analyze', {
      method: 'POST',
      body: formData
    })
    const data = await response.json()
    result.value = data
  } catch (error) {
    Notify.create({ type: 'negative', message: 'Greška: ' + error })
  } finally {
    loading.value = false
  }
}

watch(result, async (newVal) => {
  if (newVal?.status === 'success') {
    await nextTick()
    if (chartCanvas.value) {
      drawChart(newVal.result)
    }
  }
})

const drawChart = (data) => {
  if (chartInstance) chartInstance.destroy()

  chartInstance = new Chart(chartCanvas.value, {
    type: 'bar',
    data: {
      labels: ['Broj proteina', 'Broj jedinica', 'Gustoća matrice', 'Prosj. duljina'],
      datasets: [{
        label: 'Statistika',
        backgroundColor: ['#1E88E5', '#43A047', '#FB8C00', '#8E24AA'],
        data: [
          data.broj_proteina,
          data.num_ones_in_matrix,
          data.matrix_density,
          data.avg_length
        ]
      }]
    },
    options: {
      responsive: true,
      plugins: {
        legend: { display: false }
      }
    }
  })
}

const exportCsv = () => {
  if (!result.value?.result) return

  const rows = [
    ['Trajanje', result.value.result.trajanje],
    ['Broj proteina', result.value.result.broj_proteina],
    ['Dimenzija matrice', result.value.result.dimenzija_matrice],
    ['Broj jedinica u matrici', result.value.result.num_ones_in_matrix],
    ['Gustoća matrice', result.value.result.matrix_density],
    ['Prosj. duljina', result.value.result.avg_length],
    ['Prvih 5 ID-jeva', result.value.result.first_ids.join(';')],
    ['Prve 3 sekvence', result.value.result.first_sequences.join(';')]
  ]

  const csvContent = rows.map(r => r.join(',')).join('\\n')
  const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.setAttribute('href', url)
  link.setAttribute('download', 'protein_analysis.csv')
  link.click()
}
</script>

<style scoped>
.q-field {
  background-color: white;
  border-radius: 15px;
  padding: 0 15px;
  font-size: 14px;
  box-shadow: rgba(0, 0, 0, 0.3) 7px 7px 0px, rgba(0, 0, 0, 0.5) 0px 5px 5px;
}

.field {
  display: grid;
  grid-template-columns: 180px auto;
  align-items: center;
  margin-bottom: 32px;
}
</style>
