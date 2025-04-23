<script setup>
import { ref, onMounted, computed, nextTick } from 'vue'

const selectedPanel = ref('Alla')

     
                const selectedPanel = ref('Alla')
                const selectedArticle = ref(null);
                const articles = ref([]);
                const currentPage = ref(1);
                const articlesPerPage = 10;
                const renderStart = ref(0);

                const getReloadCount = () => Number(localStorage.getItem("reloadCount") || "0");
                const incrementReloadCount = () => localStorage.setItem("reloadCount", getReloadCount() + 1);
                const clearReloadCount = () => localStorage.removeItem("reloadCount");

                const saveRenderTime = (label, time) => {
                    const existing = JSON.parse(localStorage.getItem("renderTimes") || "[]");
                    existing.push({ label, time });
                    localStorage.setItem("renderTimes", JSON.stringify(existing));
                };

                const exportCSV = () => {
                    const data = JSON.parse(localStorage.getItem("renderTimes") || "[]");
                    const header = 'Mätning,Label,Tid (ms)\n';
                    const rows = data.map((entry, i) =>
                        `${i + 1},${entry.label},${entry.time.toFixed(2)}`
                    );
                    const csvContent = header + rows.join('\n');

                    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8' });
                    const link = document.createElement('a');
                    link.href = URL.createObjectURL(blob);
                    link.setAttribute('download', 'renderingstider.csv');
                    document.body.appendChild(link);
                    link.click();
                    document.body.removeChild(link);

                    localStorage.removeItem("renderTimes");
                    clearReloadCount();
                };


                const fetchJson = async () => {
                    try {
                        const res = await fetch("dataset.json");
                        const json = await res.json();
                        const data = json.data;
                        const allData = data

                            .map((row, index) => ({
                                id: index,
                                year: row[17], // år
                                ageGroup: row[15], // åldersgrupp
                                gender: row[16], // kön
                                panel: row[9], // drogtyp
                                estimate: row[21], // ungefära dödsantalet
                                description: `Under ${row[17]} was ${row[21]} deaths per 100 000 reported among ${row[15]?.toLowerCase()}.`
                            }));

                        console.log("Antal artiklar:", allData.length);
                        articles.value = allData;


                    } catch (err) {
                        console.error("Fel vid hämtning:", err);

                    }

                };

                const panels = computed(() => {
                    const all = articles.value.map(a => a.panel);
                    return ['Alla', ...new Set(all)];
                });

                const filteredArticles = computed(() => {
                    if (selectedPanel.value === 'Alla') return articles.value;
                    return articles.value.filter(a => a.panel === selectedPanel.value);
                });

                const paginatedArticles = computed(() => {
                    const start = (currentPage.value - 1) * articlesPerPage;
                    const end = start + articlesPerPage;
                    return filteredArticles.value.slice(start, end);
                });

                const totalPages = computed(() => {
                    return Math.ceil(filteredArticles.value.length / articlesPerPage);
                });

                const nextPage = async () => {
                    if (currentPage.value < totalPages.value) {
                        currentPage.value++;
                        await measureRender(`Page change to ${currentPage.value}`);

                    }
                };

                const prevPage = async () => {
                    if (currentPage.value > 1) {
                        currentPage.value--;
                        await measureRender(`Page change to ${currentPage.value}`);

                    }
                };

                onMounted(async () => {
                    const start = performance.now();
                    await fetchJson();        // hämta datan
                    await nextTick();         // vänta tills Vue ritat ut DOM

                    setTimeout(() => {
                        requestAnimationFrame(() => {
                            const end = performance.now();
                            const time = end - start;
                            const label = `Reload ${getReloadCount()}`;
                            console.log(`${label}: ${time.toFixed(2)} ms`);
                            saveRenderTime(label, time);

                            if (getReloadCount() < 50) {
                                incrementReloadCount();
                                setTimeout(() => location.reload(), 300);
                            } else {
                                exportCSV();
                                clearReloadCount();
                            }
                        });
                    }, 0);
                });


                async function autoTest(pages = 50) {
                    for (let i = 0; i < pages; i++) {
                        currentPage.value = (i % totalPages.value) + 1;
                        await measureRender(`Test page ${currentPage.value}`);
                    }
                }

                const onPanelChange = () => {
                    measureRender('Filter Applied');
                };

                async function measureRender(label = 'Render') {
                    renderStart.value = performance.now();
                    await nextTick(); // vänta tills DOM har uppdaterats
                    const renderEnd = performance.now();
                    const time = renderEnd - renderStart.value;

                    const existing = JSON.parse(localStorage.getItem("renderTimes") || "[]");
                    existing.push({ label, time });
                    localStorage.setItem("renderTimes", JSON.stringify(existing));

                    console.log(`${label}: ${time.toFixed(2)} ms`);
                }

                async function startPagingTest() {
                    let direction = 1; // 1 = next, -1 = prev

                    for (let i = 0; i < 50; i++) {
                        if (direction === 1 && currentPage.value < totalPages.value) {
                            currentPage.value++;
                        } else if (direction === -1 && currentPage.value > 1) {
                            currentPage.value--;
                        }

                        await measureRender(`Page change ${i + 1}`);

                        // Ändra riktning vid slutet
                        if (currentPage.value === totalPages.value) direction = -1;
                        if (currentPage.value === 1) direction = 1;
                    }

                    exportCSV(); // Exportera efter 50 byten
                }


                onMounted(fetchJson);

                return {
                    articles,
                    selectedArticle,
                    selectedPanel,
                    panels,
                    filteredArticles,
                    paginatedArticles,
                    currentPage,
                    totalPages,
                    nextPage,
                    prevPage,
                    onPanelChange,
                    autoTest,
                    exportCSV
                };

            
        
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

<style scoped>
body {
            font-family: sans-serif;
            margin: 0;
            background: #f4f4f4;
        }

        header {
            background: black;
            color: white;
            text-align: center;
            padding: 1rem;
        }

        .container {
            width: 30%;
            height: 60%;
            margin: 40px auto;
        }

        .card {
            background: white;
            border-radius: 5px;
            box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
            padding: 1rem;
            margin-bottom: 1rem;
            cursor: pointer;
            transition: 0.2s;
        }

        .card:hover {
            transform: scale(1.2);
        }

        .sidebar {
            position: fixed;
            top: 0;
            width: 100px;
            height: 100%;
            background: black;
            display: flex;
            color: white;
            justify-content: center;
            text-align: center;
            align-items: center;
        }

        .sidebar.left {
            left: 0;
        }

        .sidebar.right {
            right: 0;
        }

        .modal {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.7);
            display: flex;
            justify-content: center;
            align-items: center;
        }


        .modal-content {
            background: white;
            padding: 2rem;
            border-radius: 8px;
            max-width: 500px;
            width: 90%;
            max-height: fit-content;
            position: relative;
        }

        .close-btn {
            position: absolute;
            top: 10px;
            right: 15px;
            font-size: 1.5rem;
            cursor: pointer;
        }

        select {
            padding: 0.5rem 1rem;
            font-size: 1rem;
            margin: 1rem auto;
            display: block;
            border: 1px solid #ccc;
            border-radius: 5px;
            background: white;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
            cursor: pointer;
            transition: 0.2s;
        }

        select:hover {
            border-color: #666;
        }


        #app {
            background-color: white;
            display: flex;
            justify-content: center;
        }

        .pagination {
            display: flex;
            justify-content: center;
            gap: 1rem;
            margin-top: 2rem;
        }

        .pagination button {
            padding: 0.5rem 1rem;
            background-color: black;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }

        .pagination button:disabled {
            background-color: #aaa;
            cursor: not-allowed;
        }

        .exportCSV {
            text-align: center;
            margin-top: 1rem;
            z-index: 999;
            display: flex;
            justify-content: center;
        }
</style>
