# 🔍 Analisador de Qualidade dos Números de WhatsApp

Este projeto oferece duas versões para analisar a qualidade dos números de WhatsApp usando a API do Facebook Graph:

## 📱 Versões Disponíveis

### 1. **Aplicação Web (Vue.js)** - **RECOMENDADA**
Interface moderna e intuitiva para análise de números.

### 2. **Script Python**
Versão em linha de comando para análise automatizada.

---

## 🚀 Aplicação Web (Vue.js)

### Funcionalidades
- ✅ Interface moderna e responsiva
- ✅ Upload de arquivos CSV com drag & drop
- ✅ Análise em tempo real com barra de progresso
- ✅ Filtros visuais por qualidade (Bons, Médios, Ruins, Erros)
- ✅ Exportação de resultados em CSV
- ✅ Design otimizado para proporção 16:9
- ✅ Tratamento de erros com feedback visual

### Instalação e Uso

1. **Navegue para a pasta frontend:**
   ```bash
   cd frontend
   ```

2. **Instale as dependências:**
   ```bash
   npm install
   ```

3. **Execute a aplicação:**
   ```bash
   npm run dev
   ```

4. **Acesse no navegador:**
   ```
   http://localhost:5173
   ```

### Como Usar a Aplicação Web

1. **Configure o Access Token:**
   - Insira seu access token do Facebook na seção de configuração

2. **Faça Upload do Arquivo CSV:**
   - Arraste e solte o arquivo ou clique em "Selecionar Arquivo"
   - O arquivo deve ter as colunas: Nome, Número, Cod Telefone

3. **Analise os Números:**
   - Clique em "Analisar Números"
   - Acompanhe o progresso pela barra de progresso
   - Visualize os resultados nos filtros coloridos

4. **Exporte os Resultados:**
   - Use os botões de exportação para baixar os dados

---

## 🐍 Script Python

### Funcionalidades
- ✅ Lê números de um arquivo CSV
- ✅ Verifica a qualidade de cada número via API do Facebook com autenticação
- ✅ Mapeia status: GREEN = BOM, YELLOW = MÉDIO, RED = RUIM
- ✅ Exibe relatório detalhado com estatísticas
- ✅ Gera relatório XML para melhor visualização
- ✅ Gera relatório Excel (.xlsx) com múltiplas abas
- ✅ Interface amigável com emojis e formatação
- ✅ Tratamento de erros de autenticação e API

### Instalação

1. Instale as dependências:
   ```bash
   pip3 install -r requirements.txt
   ```

### Configuração

1. **Configure o Access Token:**
   - Edite o arquivo `config.env`
   - Substitua `seu_access_token_aqui` pelo seu token real do Facebook
   ```
   FACEBOOK_ACCESS_TOKEN=seu_token_real_aqui
   ```

2. **Como obter o Access Token:**
   - Acesse o [Facebook Developers](https://developers.facebook.com/)
   - Crie um app ou use um existente
   - Gere um token de acesso com permissões para WhatsApp Business API

### Uso

1. Certifique-se de que o arquivo `Números.csv` está no mesmo diretório
2. Configure o access token no arquivo `config.env`
3. Execute o programa:
   ```bash
   python3 analisador_qualidade.py
   ```

---

## 📁 Estrutura do CSV

O arquivo CSV deve ter as seguintes colunas:

- `Nome`: Nome/identificação do número
- `Número`: Número de telefone completo
- `Cod Telefone`: Código do telefone para a API

Exemplo:
```csv
Nome,Número,Cod Telefone
Sophia (IBFT Lanc 01),558131960642,731805193350393
João Silva,5511999999999,123456789012345
```

---

## 📊 Status de Qualidade

- 🟢 **BOM** (GREEN): Número de alta qualidade
- 🟡 **MÉDIO** (YELLOW): Número de qualidade média
- 🔴 **RUIM** (RED): Número de baixa qualidade

---

## 📤 Saída

### Aplicação Web
- Interface visual com filtros
- Exportação em CSV
- Barra de progresso em tempo real

### Script Python
- 📱 Verificação individual de cada número
- 📊 Resumo final com contadores
- 📋 Lista detalhada por qualidade
- ❌ Lista de erros (se houver)
- 📄 Relatório XML gerado automaticamente
- 📊 Relatório Excel gerado automaticamente

---

## 📄 Arquivos Gerados (Script Python)

### 📄 Relatório XML
- **relatorio_qualidade_YYYYMMDD_HHMMSS.xml**: Relatório completo em formato XML
- Contém estatísticas e detalhes de todos os números analisados

### 📊 Relatório Excel
- **relatorio_qualidade_YYYYMMDD_HHMMSS.xlsx**: Relatório completo em formato Excel
- **Múltiplas abas organizadas:**
  - **Dados Completos**: Todos os números com informações detalhadas
  - **Resumo**: Estatísticas com quantidades e percentuais
  - **Números BOM**: Lista dos números de alta qualidade
  - **Números MÉDIO**: Lista dos números de qualidade média
  - **Números RUIM**: Lista dos números de baixa qualidade
  - **Erros**: Lista de números com problemas (se houver)
  - **Informações**: Metadados do relatório (data/hora, taxas de sucesso, etc.)

---

## ⚠️ Tratamento de Erros

O programa trata os seguintes tipos de erro:

- **401**: Erro de autenticação (token inválido)
- **400**: Requisição inválida
- **404**: Número não encontrado
- **Erros de conexão**: Problemas de rede

---

## 📝 Observações

- O programa faz uma pausa de 1 segundo entre as verificações para não sobrecarregar a API
- Em caso de erro de conexão ou API indisponível, o programa continuará com os próximos números
- Todos os resultados são exibidos no final e salvos nos relatórios XML e Excel
- O relatório Excel é ideal para análise em planilhas e criação de gráficos
- Os relatórios incluem timestamp de geração e estatísticas completas

---

## 🔧 Exemplo de Curl

O programa usa a mesma estrutura do curl:

```bash
curl --location 'https://graph.facebook.com/v23.0/722948904241000' \
--header 'Authorization: Bearer seu_token_aqui'
```

---

## 🎯 Recomendação

**Use a aplicação web (Vue.js)** para uma experiência mais moderna e intuitiva. É a versão mais atualizada e oferece uma interface visual superior para análise de números.

---

## 📄 Licença

Este projeto é de uso livre para fins educacionais e comerciais.
