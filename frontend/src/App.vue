<script setup>
import { ref, computed } from 'vue'
import Papa from 'papaparse'
import axios from 'axios'

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

const handleDrop = (e) => {
  e.preventDefault()
  isDragOver.value = false
  
  const files = e.dataTransfer.files
  if (files.length > 0) {
    selectedFile.value = files[0]
  }
}

const handleFileSelect = (e) => {
  const files = e.target.files
  if (files.length > 0) {
    selectedFile.value = files[0]
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
    return codTelefone
  })
  
  totalNumbers.value = validNumbers.length
  currentProgress.value = 0
  
  for (let i = 0; i < csvData.length; i++) {
    const row = csvData[i]
    const nome = row['Nome'] || row['nome'] || ''
    const numero = row['Número'] || row['numero'] || ''
    const codTelefone = row['Cod Telefone'] || row['cod_telefone'] || ''
    
    if (!codTelefone) continue
    
    console.log(`Analisando ${i + 1}/${csvData.length}: ${nome}`)
    
    try {
      const result = await verifyNumberQuality(codTelefone)
      resultsData.push({
        nome: nome.trim(),
        numero: numero.trim(),
        cod_telefone: codTelefone.trim(),
        ...result
      })
      
      currentProgress.value++
      
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
      
      currentProgress.value++
    }
  }
  
  results.value = resultsData
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
               :class="{ 'drag-over': isDragOver }">
            <div class="upload-content">
              <div class="upload-icon">📄</div>
              <p>Arraste e solte seu arquivo CSV aqui</p>
              <p>ou</p>
              <input 
                type="file" 
                ref="fileInput" 
                @change="handleFileSelect" 
                accept=".csv"
                class="file-input"
              >
              <button @click="$refs.fileInput.click()" class="upload-btn">
                Selecionar Arquivo
              </button>
            </div>
          </div>
          
          <div v-if="selectedFile" class="file-info">
            <p>📎 Arquivo selecionado: {{ selectedFile.name }}</p>
            <button @click="analyzeFile" :disabled="isAnalyzing" class="analyze-btn">
              {{ isAnalyzing ? '🔍 Analisando...' : '🔍 Analisar Números' }}
            </button>
            
            <!-- Progress Bar -->
            <div v-if="isAnalyzing" class="progress-container">
              <div class="progress-bar">
                <div class="progress-fill" :style="{ width: progressPercentage + '%' }"></div>
              </div>
              <p class="progress-text">{{ currentProgress }} de {{ totalNumbers }} números analisados</p>
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
              <label for="accessToken">Access Token do Facebook:</label>
              <input 
                type="password" 
                id="accessToken" 
                v-model="accessToken"
                placeholder="Digite seu access token do Facebook"
                class="form-input"
              >
            </div>
            <div class="form-group">
              <label for="apiVersion">Versão da API:</label>
              <select id="apiVersion" v-model="apiVersion" class="form-select">
                <option value="v23.0">v23.0</option>
                <option value="v24.0">v24.0</option>
                <option value="v25.0">v25.0</option>
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
            <button 
              @click="setFilter('all')" 
              :class="['filter-card', { active: currentFilter === 'all' }]"
              class="filter-card all">
              <div class="card-icon">📊</div>
              <div class="card-content">
                <h3>Todos</h3>
                <p class="card-number">{{ results.length }}</p>
              </div>
            </button>
            
            <button 
              @click="setFilter('bom')" 
              :class="['filter-card', { active: currentFilter === 'bom' }]"
              class="filter-card good">
              <div class="card-icon">🟢</div>
              <div class="card-content">
                <h3>Bons</h3>
                <p class="card-number">{{ summary.bom }}</p>
              </div>
            </button>
            
            <button 
              @click="setFilter('medio')" 
              :class="['filter-card', { active: currentFilter === 'medio' }]"
              class="filter-card medium">
              <div class="card-icon">🟡</div>
              <div class="card-content">
                <h3>Médios</h3>
                <p class="card-number">{{ summary.medio }}</p>
              </div>
            </button>
            
            <button 
              @click="setFilter('ruim')" 
              :class="['filter-card', { active: currentFilter === 'ruim' }]"
              class="filter-card bad">
              <div class="card-icon">🔴</div>
              <div class="card-content">
                <h3>Ruins</h3>
                <p class="card-number">{{ summary.ruim }}</p>
              </div>
            </button>
            
            <button 
              @click="setFilter('erros')" 
              :class="['filter-card', { active: currentFilter === 'erros' }]"
              class="filter-card error">
              <div class="card-icon">❌</div>
              <div class="card-content">
                <h3>Erros</h3>
                <p class="card-number">{{ summary.erros }}</p>
              </div>
            </button>
          </div>

          <!-- Search and Sort Controls -->
          <div class="controls-section">
            <div class="search-container">
              <div class="search-input-wrapper">
                <input 
                  type="text" 
                  v-model="searchTerm"
                  placeholder="Buscar por nome ou número..."
                  class="search-input"
                >
                <button 
                  v-if="searchTerm" 
                  @click="clearSearch" 
                  class="clear-search-btn"
                  title="Limpar busca"
                >
                  ✕
                </button>
              </div>
            </div>
            
            <div class="sort-container">
              <label for="sortSelect" class="sort-label">Ordenar por Qualidade:</label>
              <select 
                id="sortSelect" 
                v-model="sortOrder" 
                class="sort-select"
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
            <button @click="exportToCSV" class="export-btn">
              📄 Exportar CSV
            </button>
            <button @click="exportToExcel" class="export-btn">
              📊 Exportar Excel
            </button>
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

.file-input {
  display: none;
}

.upload-btn {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  padding: 15px 30px;
  border-radius: 10px;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: 600;
  transition: all 0.3s ease;
}

.analyze-btn {
  background: linear-gradient(135deg, #28a745 0%, #20c997 100%);
  color: white;
  border: none;
  padding: 15px 30px;
  border-radius: 10px;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: 600;
  transition: all 0.3s ease;
}

.upload-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.4);
}

.analyze-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(40, 167, 69, 0.4);
}

.analyze-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
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

/* Progress Bar */
.progress-container {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 10px;
  border: 1px solid #e9ecef;
}

.progress-bar {
  width: 100%;
  height: 12px;
  background: #e9ecef;
  border-radius: 6px;
  overflow: hidden;
  margin-bottom: 0.75rem;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #28a745, #20c997);
  border-radius: 6px;
  transition: width 0.3s ease;
  position: relative;
}

.progress-fill::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.3), transparent);
  animation: shimmer 2s infinite;
}

@keyframes shimmer {
  0% { transform: translateX(-100%); }
  100% { transform: translateX(100%); }
}

.progress-text {
  text-align: center;
  font-size: 0.9rem;
  color: #6c757d;
  font-weight: 500;
  margin: 0;
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
}

.form-input,
.form-select {
  padding: 15px;
  border: 2px solid #ddd;
  border-radius: 10px;
  font-size: 1rem;
  transition: border-color 0.3s ease;
}

.form-input:focus,
.form-select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
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
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1.5rem;
  border-radius: 12px;
  color: white;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  border: none;
  cursor: pointer;
  transition: all 0.3s ease;
  min-height: 100px;
}

.filter-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.2);
}

.filter-card.active {
  transform: translateY(-3px);
  box-shadow: 0 12px 35px rgba(0, 0, 0, 0.3);
  border: 3px solid white;
}

.filter-card.all {
  background: linear-gradient(135deg, #6c757d, #495057);
}

.filter-card.good {
  background: linear-gradient(135deg, #4CAF50, #45a049);
}

.filter-card.medium {
  background: linear-gradient(135deg, #FF9800, #F57C00);
}

.filter-card.bad {
  background: linear-gradient(135deg, #F44336, #D32F2F);
}

.filter-card.error {
  background: linear-gradient(135deg, #9E9E9E, #757575);
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

.search-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-input {
  width: 100%;
  padding: 12px 40px 12px 16px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.3s ease;
  background: white;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.clear-search-btn {
  position: absolute;
  right: 8px;
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 50%;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 12px;
  transition: background-color 0.3s ease;
}

.clear-search-btn:hover {
  background: #c82333;
}

.sort-container {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.sort-label {
  font-weight: 600;
  color: #2c3e50;
  font-size: 0.9rem;
  white-space: nowrap;
}

.sort-select {
  padding: 10px 12px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 0.9rem;
  background: white;
  cursor: pointer;
  transition: border-color 0.3s ease;
}

.sort-select:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
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

.export-btn {
  background: linear-gradient(135deg, #28a745, #20c997);
  color: white;
  border: none;
  padding: 15px 30px;
  border-radius: 10px;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: 600;
  transition: all 0.3s ease;
}

.export-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(40, 167, 69, 0.4);
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
