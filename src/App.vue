<template>
  <Transition name="loader">
    <div v-if="isLoading" class="loader-wrapper">
      <p class="loader-text">Chargement...</p>
    </div>
  </Transition>
  <Transition name="tableau" mode="out-in" appear>
  <div v-if="!isLoading" class="divTableau">
    <table border="3">
      <thead>
        <tr>
          <th :aria-sort="getAriaSort('ligne')" scope="col" @click="() => trierPar('ligne')">Ligne</th>
          <th :aria-sort="getAriaSort('pilote1')" scope="col" @click="() => trierPar('pilote1')">Pilote 1</th>
          <th :aria-sort="getAriaSort('pilote2')" scope="col" @click="() => trierPar('pilote2')">Pilote 2</th>
          <th :aria-sort="getAriaSort('debutPoste')" scope="col" @click="() => trierPar('debutPoste')">Début du poste</th>
          <th>Temps d'activité</th>
          <th>Quantité</th>
          <th>Numéro OF</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(value, index) in data" :key="index">
          <td>{{ value.ligne }}</td>
          <td>{{ value.pilote1 }}</td>
          <td>{{ value.pilote2 }}</td>
          <td>{{ formatDate(value.debutPoste) }}</td>
          <td>{{ value.tempsProd }}</td>
          <td>{{ value.quantite }}</td>
          <td>{{ value.numeroOF }}</td>
        </tr>
      </tbody>
    </table>
  </div>
  </Transition>
</template>

<script setup>
import { ref, onMounted} from 'vue'
import dayjs from 'dayjs'
import 'dayjs/locale/fr'
dayjs.locale("fr")

const data = ref([])
const lignes = ref([])
const isLoading = ref(true)
const sortKey = ref(null)
const sortAsc = ref(true)

function getAriaSort(cle) {
  if (sortKey.value !== cle) return 'none'
  return sortAsc.value ? 'ascending' : 'descending'
}

onMounted(() => {
  fetchAllData()

  setInterval(() => {
    refreshDynamicTempsProdLigne()
  }, 1000) //ms

  setInterval(() => {
    refreshDynamicData()
  }, 10000) //ms
})

async function fetchAllData() {
  for (let i = 1; i <= 6; i++) {
    const [posteData, tempsProd, quantite, numeroOF] = await Promise.all([
      fetch(`/api/WS_MES_PILOTE/Infos/PosteEnCours?ligne=${i}`).then(res => res.json()),
      fetch(`/api/WS_MES_PILOTE/Infos/tempsProdLigne?ligne=${i}`).then(res => res.text()),
      fetch(`/api/WS_MES_PILOTE/Infos/quantiteLigne?ligne=${i}`).then(res => res.text()),
      fetch(`/api/WS_MES_PILOTE/Infos/numeroOfEnCoursLigne?ligne=${i}`).then(res => res.text())
    ])

    if (posteData.length > 0) {
      lignes.value.push(...posteData[0].ligne)

      data.value.push({
        ...posteData[0],
        tempsProd,
        quantite,
        numeroOF
      })
    }
  }

  console.log(lignes.value)

  isLoading.value = false
}

async function refreshDynamicData() {
  const lignesActives = []

  for (let i = 1; i <= 6; i++) {
    const posteData = await fetch(`/api/WS_MES_PILOTE/Infos/PosteEnCours?ligne=${i}`).then(res => res.json())

    if (posteData.length > 0) {
      const [quantite, numeroOF] = await Promise.all([
        fetch(`/api/WS_MES_PILOTE/Infos/quantiteLigne?ligne=${i}`).then(res => res.text()),
        fetch(`/api/WS_MES_PILOTE/Infos/numeroOfEnCoursLigne?ligne=${i}`).then(res => res.text())
      ])

      const index = data.value.findIndex(d => d.ligne === posteData[0].ligne)

      const newEntry = {
        ...posteData[0],
        quantite,
        numeroOf: numeroOF
      }

      if (index !== -1) {
        // update
        data.value[index] = { ...data.value[index], ...newEntry }
      } else {
        // insert
        data.value.push(newEntry)
      }

      lignesActives.push(posteData[0].ligne)
    }
  }

  // Supprimer les lignes qui ne sont plus actives
  // data.value = data.value.filter(d => lignesActives.includes(d.ligne))
  // lignes.value = lignesActives
}

async function refreshDynamicTempsProdLigne() {
  for (let i = 0; i < data.value.length; i++) {
    const ligne = data.value[i].ligne
    const tempsProd = await fetch(`/api/WS_MES_PILOTE/Infos/tempsProdLigne?ligne=${ligne}`).then(res => res.text())

    data.value[i].tempsProd = tempsProd
  }
}

function trierPar(cle) {
  if (sortKey.value === cle) {
    sortAsc.value = !sortAsc.value
  } else {
    sortKey.value = cle
    sortAsc.value = true
  }

  data.value.sort((a, b) => {
    if (a[cle] < b[cle]) return sortAsc.value ? -1 : 1
    if (a[cle] > b[cle]) return sortAsc.value ? 1 : -1
    return 0
  })
}

function formatDate(dateString) {
  if (!dateString) return ''
  return dayjs(dateString).format('DD/MM/YYYY HH:mm')
}
</script>

<style>
#app{
  display: flex; /*écrase le display: grid de app*/
  place-items: center;
}

body::-webkit-scrollbar {
  display: none;
}

.relative {
  position: relative;
}
</style>
