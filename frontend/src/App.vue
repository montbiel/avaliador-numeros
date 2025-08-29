<script setup>
import { ref, computed, nextTick } from 'vue'
import Papa from 'papaparse'
import axios from 'axios'
import { BaseButton, BaseInput, BaseProgress, Card } from './components/argon'

// Reactive data
const selectedFile = ref(null)
const accessToken = ref('')
const apiVersion = ref('v23.0')
const results = ref([])
const isAnalyzing = ref(false)
const isDragOver = ref(false)
const fileInput = ref(null)
const currentFilter = ref('all')
const currentProgress = ref(0)
const totalNumbers = ref(0)
const searchTerm = ref('')
const sortOrder = ref('none') // 'none', 'asc', 'desc'

// Computed properties
const summary = computed(() => {
  const summaryData = { bom: 0, medio: 0, ruim: 0, erros: 0 }
  
  results.value.forEach(result => {
    if (result.status === 'SUCESSO') {
      switch (result.qualidade_pt) {
        case 'BOM':
          summaryData.bom++
          break
        case 'MÉDIO':
          summaryData.medio++
          break
        case 'RUIM':
          summaryData.ruim++
          break
        default:
          summaryData.erros++
      }
    } else {
      summaryData.erros++
    }
  })
  
  return summaryData
})

const filteredResults = computed(() => {
  let filtered = results.value
  
  // Aplicar filtro por qualidade
  if (currentFilter.value !== 'all') {
    filtered = results.value.filter(result => {
      if (currentFilter.value === 'bom') {
        return result.status === 'SUCESSO' && result.qualidade_pt === 'BOM'
      } else if (currentFilter.value === 'medio') {
        return result.status === 'SUCESSO' && result.qualidade_pt === 'MÉDIO'
      } else if (currentFilter.value === 'ruim') {
        return result.status === 'SUCESSO' && result.qualidade_pt === 'RUIM'
      } else if (currentFilter.value === 'erros') {
        return result.status !== 'SUCESSO'
      }
      return true
    })
  }
  
  // Aplicar busca por nome
  if (searchTerm.value.trim()) {
    const searchLower = searchTerm.value.toLowerCase().trim()
    filtered = filtered.filter(result => 
      result.nome.toLowerCase().includes(searchLower) ||
      result.numero.includes(searchTerm.value.trim())
    )
  }
  
  // Aplicar ordenação por qualidade
  if (sortOrder.value !== 'none') {
    filtered = [...filtered].sort((a, b) => {
      const qualityOrder = { 'BOM': 3, 'MÉDIO': 2, 'RUIM': 1 }
      const aQuality = qualityOrder[a.qualidade_pt] || 0
      const bQuality = qualityOrder[b.qualidade_pt] || 0
      
      if (sortOrder.value === 'asc') {
        return aQuality - bQuality
      } else {
        return bQuality - aQuality
      }
    })
  }
  
  return filtered
})

const progressPercentage = computed(() => {
  if (totalNumbers.value === 0) return 0
  return Math.round((currentProgress.value / totalNumbers.value) * 100)
})

// Methods
const setFilter = (filter) => {
  currentFilter.value = filter
}

const getFilterDisplayName = () => {
  const filterNames = {
    'all': 'Geral',
    'bom': 'Bons',
    'medio': 'Médios',
    'ruim': 'Ruins',
    'erros': 'Com Erros'
  }
  return filterNames[currentFilter.value] || 'Geral'
}

const setSortOrder = (order) => {
  sortOrder.value = order
}

const getSortDisplayName = () => {
  const sortNames = {
    'none': 'Sem Ordenação',
    'asc': 'Menor Qualidade',
    'desc': 'Maior Qualidade'
  }
  return sortNames[sortOrder.value] || 'Sem Ordenação'
}

const clearSearch = () => {
  searchTerm.value = ''
}

const triggerFileInput = async () => {
  console.log('Triggering file input...')
  
  // Método 1: Usar a referência Vue
  if (fileInput.value) {
    console.log('File input found via ref, clicking...')
    fileInput.value.click()
    return
  }
  
  // Método 2: Usar nextTick e tentar novamente
  await nextTick()
  if (fileInput.value) {
    console.log('File input found after nextTick, clicking...')
    fileInput.value.click()
    return
  }
  
  // Método 3: Fallback - buscar por ID
  const inputElement = document.getElementById('fileInput')
  if (inputElement) {
    console.log('Found input by ID, clicking...')
    inputElement.click()
    return
  }
  
  // Método 4: Fallback - buscar por querySelector
  const inputByQuery = document.querySelector('input[type="file"]')
  if (inputByQuery) {
    console.log('Found input by querySelector, clicking...')
    inputByQuery.click()
    return
  }
  
  console.log('No file input found by any method')
  alert('Erro: Não foi possível abrir o seletor de arquivos. Tente arrastar e soltar o arquivo.')
}

const handleDrop = (e) => {
  e.preventDefault()
  isDragOver.value = false
  
  const files = e.dataTransfer.files
  if (files.length > 0) {
    const file = files[0]
    if (file.type === 'text/csv' || file.name.endsWith('.csv')) {
      selectedFile.value = file
    } else {
      alert('Por favor, selecione apenas arquivos CSV')
    }
  }
}

const handleFileSelect = (e) => {
  const files = e.target.files
  if (files.length > 0) {
    const file = files[0]
    if (file.type === 'text/csv' || file.name.endsWith('.csv')) {
      selectedFile.value = file
    } else {
      alert('Por favor, selecione apenas arquivos CSV')
      e.target.value = '' // Limpa o input
    }
  }
}

const readCSVFile = (file) => {
  return new Promise((resolve, reject) => {
    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      complete: (results) => {
        if (results.errors.length > 0) {
          reject(new Error('Erro ao ler o arquivo CSV'))
        } else {
          resolve(results.data)
        }
      },
      error: (error) => {
        reject(error)
      }
    })
  })
}

const verifyNumberQuality = async (codTelefone) => {
  const url = `https://graph.facebook.com/${apiVersion.value}/${codTelefone}`
  
  const headers = {
    'Authorization': `Bearer ${accessToken.value}`,
    'Content-Type': 'application/json'
  }
  
  try {
    const response = await axios.get(url, { headers, timeout: 10000 })
    
    if (response.status === 200) {
      const data = response.data
      
      if (data.quality_rating) {
        const quality = data.quality_rating
        const statusMap = {
          'GREEN': 'BOM',
          'YELLOW': 'MÉDIO',
          'RED': 'RUIM'
        }
        
        const statusPt = statusMap[quality] || 'DESCONHECIDO'
        
        return {
          status: 'SUCESSO',
          qualidade: quality,
          qualidade_pt: statusPt,
          dados: data
        }
      } else {
        return {
          status: 'SEM_INFO_QUALIDADE',
          qualidade: 'N/A',
          qualidade_pt: 'SEM INFORMAÇÃO',
          dados: data
        }
      }
    } else if (response.status === 404) {
      return {
        status: 'NAO_ENCONTRADO',
        qualidade: 'N/A',
        qualidade_pt: 'NÚMERO NÃO ENCONTRADO',
        dados: null
      }
    } else if (response.status === 401) {
      return {
        status: 'ERRO_AUTENTICACAO',
        qualidade: 'N/A',
        qualidade_pt: 'ERRO DE AUTENTICAÇÃO - TOKEN INVÁLIDO',
        dados: null
      }
    } else {
      return {
        status: 'ERRO_API',
        qualidade: 'N/A',
        qualidade_pt: `ERRO ${response.status}`,
        dados: null
      }
    }
  } catch (error) {
    if (error.response) {
      if (error.response.status === 404) {
        return {
          status: 'NAO_ENCONTRADO',
          qualidade: 'N/A',
          qualidade_pt: 'NÚMERO NÃO ENCONTRADO',
          dados: null
        }
      } else if (error.response.status === 401) {
        return {
          status: 'ERRO_AUTENTICACAO',
          qualidade: 'N/A',
          qualidade_pt: 'ERRO DE AUTENTICAÇÃO - TOKEN INVÁLIDO',
          dados: null
        }
      }
    }
    
    return {
      status: 'ERRO_CONEXAO',
      qualidade: 'N/A',
      qualidade_pt: `ERRO DE CONEXÃO: ${error.message}`,
      dados: null
    }
  }
}

const analyzeNumbers = async (csvData) => {
  const resultsData = []
  const validNumbers = csvData.filter(row => {
    const codTelefone = row['Cod Telefone'] || row['cod_telefone'] || ''
    return codTelefone && codTelefone.trim() !== ''
  })
  
  totalNumbers.value = validNumbers.length
  currentProgress.value = 0
  
  console.log(`Total de números válidos para análise: ${validNumbers.length}`)
  
  for (let i = 0; i < validNumbers.length; i++) {
    const row = validNumbers[i]
    const nome = row['Nome'] || row['nome'] || ''
    const numero = row['Número'] || row['numero'] || ''
    const codTelefone = row['Cod Telefone'] || row['cod_telefone'] || ''
    
    console.log(`Analisando ${i + 1}/${validNumbers.length}: ${nome} (${codTelefone})`)
    
    try {
      const result = await verifyNumberQuality(codTelefone)
      resultsData.push({
        nome: nome.trim(),
        numero: numero.trim(),
        cod_telefone: codTelefone.trim(),
        ...result
      })
      
      currentProgress.value = i + 1
      
      // Pequena pausa para não sobrecarregar a API
      await new Promise(resolve => setTimeout(resolve, 1000))
    } catch (error) {
      resultsData.push({
        nome: nome.trim(),
        numero: numero.trim(),
        cod_telefone: codTelefone.trim(),
        status: 'ERRO_CONEXAO',
        qualidade: 'N/A',
        qualidade_pt: `Erro: ${error.message}`,
        dados: null
      })
      
      currentProgress.value = i + 1
    }
  }
  
  results.value = resultsData
  console.log('Análise concluída!')
}

const analyzeFile = async () => {
  if (!selectedFile.value) {
    alert('Por favor, selecione um arquivo CSV')
    return
  }
  
  if (!accessToken.value) {
    alert('Por favor, insira o access token do Facebook')
    return
  }
  
  isAnalyzing.value = true
  results.value = []
  currentFilter.value = 'all'
  currentProgress.value = 0
  totalNumbers.value = 0
  searchTerm.value = ''
  sortOrder.value = 'none'
  
  try {
    const csvData = await readCSVFile(selectedFile.value)
    await analyzeNumbers(csvData)
  } catch (error) {
    console.error('Erro na análise:', error)
    alert('Erro ao analisar o arquivo: ' + error.message)
  } finally {
    isAnalyzing.value = false
  }
}

const getStatusClass = (status) => {
  if (status === 'SUCESSO') return 'success'
  if (status === 'NAO_ENCONTRADO') return 'not-found'
  if (status === 'ERRO_AUTENTICACAO') return 'auth-error'
  return 'error'
}

const getQualityClass = (quality) => {
  if (quality === 'BOM') return 'quality-good'
  if (quality === 'MÉDIO') return 'quality-medium'
  if (quality === 'RUIM') return 'quality-bad'
  return 'quality-unknown'
}

const exportToCSV = () => {
  const headers = ['Nome', 'Número', 'Código Telefone', 'Status', 'Qualidade']
  const csvContent = [
    headers.join(','),
    ...filteredResults.value.map(result => [
      `"${result.nome}"`,
      `"${result.numero}"`,
      `"${result.cod_telefone}"`,
      `"${result.status}"`,
      `"${result.qualidade_pt}"`
    ].join(','))
  ].join('\n')
  
  const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.download = `analise_qualidade_${currentFilter.value}_${new Date().toISOString().split('T')[0]}.csv`
  link.click()
}

const exportToExcel = () => {
  // Para exportar Excel, precisaríamos de uma biblioteca adicional
  // Por enquanto, vamos usar CSV como alternativa
  alert('Funcionalidade de exportação Excel será implementada em breve!')
  exportToCSV()
}
</script>

<template>
  <div id="app">
    <!-- Header Centralizado -->
    <header class="header">
      <div class="header-content">
        <h1>🔍 Analisador de Qualidade - WhatsApp</h1>
        <p>Verifique a qualidade dos números de telefone do WhatsApp</p>
      </div>
    </header>

    <main class="main-content">
      <!-- Upload Section - Largura Total -->
      <section class="full-width-section upload-section">
        <div class="section-container">
          <h2>📁 Upload do Arquivo CSV</h2>
          <div class="upload-area" 
               @drop="handleDrop" 
               @dragover.prevent 
               @dragenter.prevent
               @dragleave.prevent
               @dragenter="isDragOver = true"
               @dragleave="isDragOver = false"
               :class="{ 'drag-over': isDragOver }">
            <div class="upload-content">
              <div class="upload-icon">📄</div>
              <p>Arraste e solte seu arquivo CSV aqui</p>
              <p>ou</p>
              <input 
                type="file" 
                ref="fileInput" 
                id="fileInput"
                @change="handleFileSelect" 
                accept=".csv"
                class="file-input"
                style="display: none;"
              >
              <button 
                @click="triggerFileInput" 
                type="button"
                class="btn btn-primary btn-lg"
              >
                📁 Selecionar Arquivo
              </button>
              <p class="upload-hint">Clique no botão acima para selecionar um arquivo CSV</p>
            </div>
          </div>
          
          <div v-if="selectedFile" class="file-info">
            <p>📎 Arquivo selecionado: <strong>{{ selectedFile.name }}</strong></p>
            <p class="file-size">Tamanho: {{ (selectedFile.size / 1024).toFixed(2) }} KB</p>
            <BaseButton 
              @click="analyzeFile" 
              :disabled="isAnalyzing" 
              type="success"
              size="lg"
              icon="ni ni-chart-bar-32"
            >
              {{ isAnalyzing ? '🔍 Analisando...' : '🔍 Analisar Números' }}
            </BaseButton>
            
            <!-- Progress Bar -->
            <div v-if="isAnalyzing" class="progress-container">
              <BaseProgress 
                :value="progressPercentage"
                type="success"
                :striped="true"
                :animated="true"
                :height="12"
                :label="`${currentProgress} de ${totalNumbers} números analisados`"
              />
            </div>
          </div>
        </div>
      </section>

      <!-- Configuration Section - Largura Total -->
      <section class="full-width-section config-section">
        <div class="section-container">
          <h2>⚙️ Configuração</h2>
          <div class="config-form">
            <div class="form-group">
              <label for="accessToken">Access Token do Facebook</label>
              <input
                type="password"
                id="accessToken"
                v-model="accessToken"
                placeholder="Digite seu access token do Facebook"
                class="form-control"
                required
              />
            </div>
            <div class="form-group">
              <label for="apiVersion">Versão da API</label>
              <select
                id="apiVersion"
                v-model="apiVersion"
                class="form-control"
              >
                <option value="v23.0">v23.0</option>
                <option value="v24.0">v24.0</option>
                <option value="v25.0">v25.0</option>
                <option value="v26.0">v26.0</option>
                <option value="v27.0">v27.0</option>
                <option value="v28.0">v28.0</option>
              </select>
            </div>
          </div>
        </div>
      </section>

      <!-- Results Section - Largura Total -->
      <section v-if="results.length > 0" class="full-width-section results-section">
        <div class="section-container">
          <h2>📊 Resultados da Análise</h2>
          
          <!-- Summary Cards as Filters -->
          <div class="filter-cards">
            <Card 
              @click="setFilter('all')" 
              :class="{ active: currentFilter === 'all' }"
              type="secondary"
              hover
              shadow
              class="filter-card"
            >
              <div class="card-icon">📊</div>
              <div class="card-content">
                <h3>Todos</h3>
                <p class="card-number">{{ results.length }}</p>
              </div>
            </Card>
            
            <Card 
              @click="setFilter('bom')" 
              :class="{ active: currentFilter === 'bom' }"
              type="success"
              hover
              shadow
              class="filter-card"
            >
              <div class="card-icon">🟢</div>
              <div class="card-content">
                <h3>Bons</h3>
                <p class="card-number">{{ summary.bom }}</p>
              </div>
            </Card>
            
            <Card 
              @click="setFilter('medio')" 
              :class="{ active: currentFilter === 'medio' }"
              type="warning"
              hover
              shadow
              class="filter-card"
            >
              <div class="card-icon">🟡</div>
              <div class="card-content">
                <h3>Médios</h3>
                <p class="card-number">{{ summary.medio }}</p>
              </div>
            </Card>
            
            <Card 
              @click="setFilter('ruim')" 
              :class="{ active: currentFilter === 'ruim' }"
              type="danger"
              hover
              shadow
              class="filter-card"
            >
              <div class="card-icon">🔴</div>
              <div class="card-content">
                <h3>Ruins</h3>
                <p class="card-number">{{ summary.ruim }}</p>
              </div>
            </Card>
            
            <Card 
              @click="setFilter('erros')" 
              :class="{ active: currentFilter === 'erros' }"
              type="dark"
              hover
              shadow
              class="filter-card"
            >
              <div class="card-icon">❌</div>
              <div class="card-content">
                <h3>Erros</h3>
                <p class="card-number">{{ summary.erros }}</p>
              </div>
            </Card>
          </div>

          <!-- Search and Sort Controls -->
          <div class="controls-section">
            <div class="search-container">
              <label for="searchInput">Buscar</label>
              <div class="search-input-wrapper">
                <input
                  type="text"
                  id="searchInput"
                  v-model="searchTerm"
                  placeholder="Buscar por nome ou número..."
                  class="form-control search-input"
                />
                <button 
                  v-if="searchTerm" 
                  @click="clearSearch"
                  type="button"
                  class="search-clear-btn"
                  title="Limpar busca"
                >
                  ✕
                </button>
              </div>
            </div>
            
            <div class="sort-container">
              <label for="sortSelect">Ordenar por Qualidade</label>
              <select
                id="sortSelect"
                v-model="sortOrder"
                class="form-control"
              >
                <option value="none">Sem Ordenação</option>
                <option value="desc">Maior Qualidade</option>
                <option value="asc">Menor Qualidade</option>
              </select>
            </div>
          </div>

          <!-- Results Table -->
          <div class="results-table">
            <div class="table-header">
              <h3>Resultados {{ getFilterDisplayName() }}</h3>
              <p class="results-count">{{ filteredResults.length }} de {{ results.length }} números</p>
              <p v-if="searchTerm" class="search-info">🔍 Buscando por: "{{ searchTerm }}"</p>
              <p v-if="sortOrder !== 'none'" class="sort-info">📊 {{ getSortDisplayName() }}</p>
            </div>
            
            <table>
              <thead>
                <tr>
                  <th>Nome</th>
                  <th>Número</th>
                  <th>Código</th>
                  <th>Status</th>
                  <th>Qualidade</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="result in filteredResults" :key="result.cod_telefone" 
                    :class="getStatusClass(result.status)">
                  <td>{{ result.nome }}</td>
                  <td>{{ result.numero }}</td>
                  <td>{{ result.cod_telefone }}</td>
                  <td>
                    <span class="status-badge" :class="getStatusClass(result.status)">
                      {{ result.status }}
                    </span>
                  </td>
                  <td>
                    <span class="quality-badge" :class="getQualityClass(result.qualidade_pt)">
                      {{ result.qualidade_pt }}
                    </span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Export Buttons -->
          <div class="export-section">
            <BaseButton 
              @click="exportToCSV" 
              type="info"
              icon="ni ni-single-copy-04"
            >
              Exportar CSV
            </BaseButton>
            <BaseButton 
              @click="exportToExcel" 
              type="warning"
              icon="ni ni-chart-pie-35"
            >
              Exportar Excel
            </BaseButton>
          </div>
        </div>
      </section>

      
    </main>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  color: #333;
}

#app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* Header Centralizado */
.header {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  padding: 2rem 0;
  text-align: center;
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.1);
  width: 100%;
  position: relative;
  z-index: 10;
}

.header-content {
  max-width: 100%;
  margin: 0 auto;
  padding: 0 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.header h1 {
  color: #2c3e50;
  margin-bottom: 0.75rem;
  font-size: 2.8rem;
  text-align: center;
  font-weight: 700;
  line-height: 1.2;
  width: 100%;
  max-width: 900px;
}

.header p {
  color: #7f8c8d;
  font-size: 1.2rem;
  text-align: center;
  max-width: 700px;
  line-height: 1.5;
}

/* Main Content */
.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2rem;
  padding: 2rem 0;
}

/* Seções de Largura Total */
.full-width-section {
  width: 100%;
  background: white;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.section-container {
  max-width: 100%;
  margin: 0 auto;
  padding: 2.5rem 4rem;
}

section h2 {
  color: #2c3e50;
  margin-bottom: 2rem;
  font-size: 2rem;
  text-align: center;
  font-weight: 600;
}

/* Upload Section */
.upload-area {
  border: 3px dashed #ddd;
  border-radius: 15px;
  padding: 3rem;
  text-align: center;
  transition: all 0.3s ease;
  cursor: pointer;
  margin-bottom: 1.5rem;
}

.upload-area:hover,
.upload-area.drag-over {
  border-color: #667eea;
  background: #f8f9ff;
  transform: scale(1.02);
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.2);
}

.upload-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.25rem;
}

.upload-icon {
  font-size: 3.5rem;
}

.upload-content p {
  font-size: 1.2rem;
  color: #666;
}

.upload-hint {
  font-size: 0.9rem !important;
  color: #6c757d !important;
  margin-top: 0.5rem;
}

.file-input {
  display: none;
}



.file-info {
  text-align: center;
  padding: 1.5rem;
  background: #f8f9fa;
  border-radius: 10px;
}

.file-info p {
  font-size: 1.1rem;
  margin-bottom: 1rem;
}

.file-size {
  color: #6c757d;
  font-size: 0.9rem !important;
  margin-bottom: 1.5rem !important;
}

/* Progress Bar */
.progress-container {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 10px;
  border: 1px solid #e9ecef;
}

/* Configuration Section */
.config-form {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 2rem;
  max-width: 800px;
  margin: 0 auto;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.form-group label {
  font-weight: 600;
  color: #2c3e50;
  font-size: 1.1rem;
  margin-bottom: 0.5rem;
  display: block;
}

.form-control {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  font-size: 1rem;
  transition: all 0.3s ease;
  background: white;
  color: #495057;
}

.form-control:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.form-control::placeholder {
  color: #6c757d;
}

select.form-control {
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 20 20'%3e%3cpath stroke='%236b7280' stroke-linecap='round' stroke-linejoin='round' stroke-width='1.5' d='m6 8 4 4 4-4'/%3e%3c/svg%3e");
  background-position: right 12px center;
  background-repeat: no-repeat;
  background-size: 16px;
  padding-right: 40px;
  appearance: none;
}



/* Filter Cards */
.filter-cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 1rem;
  margin-bottom: 3rem;
  max-width: 1000px;
  margin-left: auto;
  margin-right: auto;
}

.filter-card {
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  min-height: 100px;
}

.filter-card:hover {
  transform: translateY(-3px);
}

.filter-card.active {
  transform: scale(1.05);
}

.card-icon {
  font-size: 2rem;
  margin-bottom: 0.75rem;
}

.card-content h3 {
  font-size: 1rem;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

.card-number {
  font-size: 1.5rem;
  font-weight: bold;
}

/* Search and Sort Controls */
.controls-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  gap: 2rem;
  padding: 1.5rem;
  background: #f8f9fa;
  border-radius: 12px;
  border: 1px solid #e9ecef;
}

.search-container {
  flex: 1;
  max-width: 400px;
}

.sort-container {
  min-width: 250px;
}

/* Results Table */
.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  padding: 0 1rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.table-header h3 {
  color: #2c3e50;
  font-size: 1.4rem;
  margin: 0;
}

.results-count {
  color: #7f8c8d;
  font-size: 1rem;
  margin: 0;
}

.search-info {
  color: #28a745;
  font-size: 0.9rem;
  margin: 0;
  font-weight: 500;
}

.sort-info {
  color: #667eea;
  font-size: 0.9rem;
  margin: 0;
  font-weight: 500;
}

.results-table {
  overflow-x: auto;
  margin-bottom: 3rem;
}

table {
  width: 100%;
  border-collapse: collapse;
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
}

th, td {
  padding: 15px;
  text-align: left;
  border-bottom: 1px solid #eee;
}

th {
  background: #f8f9fa;
  font-weight: 600;
  color: #2c3e50;
  font-size: 1rem;
}

.status-badge,
.quality-badge {
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 0.85rem;
  font-weight: 600;
}

.status-badge.success {
  background: #d4edda;
  color: #155724;
}

.status-badge.not-found {
  background: #fff3cd;
  color: #856404;
}

.status-badge.auth-error {
  background: #f8d7da;
  color: #721c24;
}

.status-badge.error {
  background: #f8d7da;
  color: #721c24;
}

.quality-badge.quality-good {
  background: #d4edda;
  color: #155724;
}

.quality-badge.quality-medium {
  background: #fff3cd;
  color: #856404;
}

.quality-badge.quality-bad {
  background: #f8d7da;
  color: #721c24;
}

.quality-badge.quality-unknown {
  background: #e2e3e5;
  color: #383d41;
}

/* Export Section */
.export-section {
  display: flex;
  gap: 1.5rem;
  justify-content: center;
}

/* Search Input Styles */
.search-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-input {
  padding-right: 40px;
}

.search-clear-btn {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  color: #6c757d;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 14px;
  transition: all 0.2s ease;
}

.search-clear-btn:hover {
  background: #e9ecef;
  color: #495057;
}

/* Custom Button Styles */
.btn {
  display: inline-block;
  font-weight: 400;
  text-align: center;
  vertical-align: middle;
  user-select: none;
  border: 1px solid transparent;
  padding: 0.375rem 0.75rem;
  font-size: 1rem;
  line-height: 1.5;
  border-radius: 0.25rem;
  transition: color 0.15s ease-in-out, background-color 0.15s ease-in-out, border-color 0.15s ease-in-out, box-shadow 0.15s ease-in-out;
}

.btn-primary {
  color: #fff;
  background-color: #667eea;
  border-color: #667eea;
}

.btn-primary:hover {
  color: #fff;
  background-color: #5a6fd8;
  border-color: #5a6fd8;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
}

.btn-lg {
  padding: 0.75rem 1.5rem;
  font-size: 1.25rem;
  border-radius: 0.5rem;
}



/* Responsividade */
@media (max-width: 1200px) {
  .section-container {
    padding: 2rem 3rem;
  }
  
  .header h1 {
    font-size: 2.5rem;
  }
  
  .header p {
    font-size: 1.1rem;
  }
}

@media (max-width: 768px) {
  .header {
    padding: 2rem 0;
  }
  
  .header h1 {
    font-size: 2.5rem;
    padding: 0 1rem;
  }
  
  .header p {
    font-size: 1.2rem;
    padding: 0 1rem;
  }
  
  .section-container {
    padding: 2rem 1.5rem;
  }
  
  .upload-area {
    padding: 3rem 2rem;
  }
  
  .config-form {
    grid-template-columns: 1fr;
    gap: 1.5rem;
  }
  
  .filter-cards {
    grid-template-columns: repeat(3, 1fr);
  }
  
  .export-section {
    flex-direction: column;
  }
  
  .controls-section {
    flex-direction: column;
    gap: 1rem;
  }
  
  .search-container {
    max-width: 100%;
  }
  
  .sort-container {
    width: 100%;
    justify-content: space-between;
  }
  
  .table-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.75rem;
  }
}

@media (max-width: 480px) {
  .header h1 {
    font-size: 2rem;
  }
  
  .header p {
    font-size: 1.1rem;
  }
  
  .section-container {
    padding: 1.5rem 1rem;
  }
  
  .upload-area {
    padding: 2rem 1rem;
  }
  
  .filter-cards {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .upload-icon {
    font-size: 3rem;
  }
  
  .upload-content p {
    font-size: 1rem;
  }
}

@media (min-width: 1600px) {
  .section-container {
    padding: 3rem 6rem;
  }
  
  .header h1 {
    font-size: 3.2rem;
  }
  
  .header p {
    font-size: 1.4rem;
  }
  
  .filter-cards {
    grid-template-columns: repeat(5, 1fr);
  }
}
</style>
