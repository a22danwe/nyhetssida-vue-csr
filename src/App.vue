<script setup>
import { ref, onMounted, computed, nextTick } from 'vue'

const selectedPanel = ref('Alla')
const selectedArticle = ref(null)
const articles = ref([])
const currentPage = ref(1)
const articlesPerPage = 10
const renderStart = ref(0)

const selectArticle = (item) => {
  selectedArticle.value = item
}


const getReloadCount = () => Number(localStorage.getItem("reloadCount") || "0")
const incrementReloadCount = () => localStorage.setItem("reloadCount", getReloadCount() + 1)
const clearReloadCount = () => localStorage.removeItem("reloadCount")

const saveRenderTime = (label, time) => {
  const existing = JSON.parse(localStorage.getItem("renderTimes") || "[]")
  existing.push({ label, time })
  localStorage.setItem("renderTimes", JSON.stringify(existing))
}

const exportCSV = () => {
  const data = JSON.parse(localStorage.getItem("renderTimes") || "[]")
  const header = 'Mätning,Label,Tid (ms)\n'
  const rows = data.map((entry, i) =>
    `${i + 1},${entry.label},${entry.time.toFixed(2)}`
  )
  const csvContent = header + rows.join('\n')

  const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8' })
  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.setAttribute('download', 'renderingstider.csv')
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)

  localStorage.removeItem("renderTimes")
  clearReloadCount()
}

const fetchJson = async () => {
  try {
    const res = await fetch("dataset.json")
    const json = await res.json()
    const data = json.data
    const allData = data.map((row, index) => ({
      id: index,
      year: row[17],
      ageGroup: row[15],
      gender: row[16],
      panel: row[9],
      estimate: row[21],
      state: row[5],
      race: row[14],
      description: `Under ${row[17]} was ${row[21]} deaths per 100 000 reported among ${row[15]?.toLowerCase()}.`
    }))
    articles.value = allData
  } catch (err) {
    console.error("Fel vid hämtning:", err)
  }
}

const panels = computed(() => {
  const all = articles.value.map(a => a.panel)
  return ['Alla', ...new Set(all)]
})

const filteredArticles = computed(() => {
  if (selectedPanel.value === 'Alla') return articles.value
  return articles.value.filter(a => a.panel === selectedPanel.value)
})

const paginatedArticles = computed(() => {
  const start = (currentPage.value - 1) * articlesPerPage
  const end = start + articlesPerPage
  return filteredArticles.value.slice(start, end)
})

const totalPages = computed(() => {
  return Math.ceil(filteredArticles.value.length / articlesPerPage)
})

const nextPage = async () => {
  if (currentPage.value < totalPages.value) {
    currentPage.value++
    await measureRender(`Page change to ${currentPage.value}`)
  }
}

const prevPage = async () => {
  if (currentPage.value > 1) {
    currentPage.value--
    await measureRender(`Page change to ${currentPage.value}`)
  }
}

const onPanelChange = () => {
  measureRender('Filter Applied')
}

async function measureRender(label = 'Render') {
  renderStart.value = performance.now()
  await nextTick()
  const renderEnd = performance.now()
  const time = renderEnd - renderStart.value

  const existing = JSON.parse(localStorage.getItem("renderTimes") || "[]")
  existing.push({ label, time })
  localStorage.setItem("renderTimes", JSON.stringify(existing))

  console.log(`${label}: ${time.toFixed(2)} ms`)
}

onMounted(async () => {
  const start = performance.now()
  await fetchJson()
  await nextTick()

  setTimeout(() => {
    requestAnimationFrame(() => {
      const end = performance.now()
      const time = end - start
      const label = `Reload ${getReloadCount()}`
      console.log(`${label}: ${time.toFixed(2)} ms`)
      saveRenderTime(label, time)

      if (getReloadCount() < 200) {
        incrementReloadCount()
        setTimeout(() => location.reload(), 300)
      } else {
        exportCSV()
        clearReloadCount()
      }
    })
  }, 0)
})

</script>


<template>
    <div>
    <header>
      <h1>Nyheter: Drug Overdose Deaths In The US</h1>
    </header>

    <div class="container">
      <select v-model="selectedPanel">
        <option value="Alla">Alla</option>
        <option v-for="panel in panels" :key="panel">{{ panel }}</option>
      </select>

      <div class="card" v-for="item in paginatedArticles" :key="item.id" @click="selectArticle(item)">
        <h3>{{ item.year }} – {{ item.ageGroup }} – {{ item.gender }}</h3>
        <p><strong>Drug type:</strong> {{ item.panel }}</p>
      </div>

      <div class="pagination">
        <button @click="prevPage" :disabled="currentPage === 1">Föregående</button>
        <span>Sida {{ currentPage }} av {{ totalPages }}</span>
        <button @click="nextPage" :disabled="currentPage === totalPages">Nästa</button>
      </div>

      <div class="exportCSV">
        <button @click="exportCSV">Exportera som CSV</button>
      </div>
    </div>

    <div class="modal" v-if="selectedArticle" @click.self="selectedArticle = null">
      <div class="modal-content">
        <span class="close-btn" @click="selectedArticle = null">&times;</span>
        <h2>{{ selectedArticle.year }} – {{ selectedArticle.ageGroup }} – {{ selectedArticle.gender }}</h2>
        <p><strong>Drug type:</strong> {{ selectedArticle.panel }}</p>
        <p><strong>State:</strong> {{ selectedArticle.state }}</p>
        <p><strong>Race:</strong> {{ selectedArticle.race }}</p>
        <p><strong>Deaths:</strong> {{ selectedArticle.estimate }} per 100 000</p>
        <p>{{ selectedArticle.description }}</p>
      </div>
    </div>

    <div class="sidebar left"></div>
    <div class="sidebar right"></div>
  </div>
</template>


