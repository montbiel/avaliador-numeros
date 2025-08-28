# 🔍 Analisador de Qualidade - WhatsApp

Uma aplicação Vue.js para verificar a qualidade dos números de telefone do WhatsApp usando a API do Facebook Graph.

## 🚀 Funcionalidades

- **Upload de arquivos CSV**: Arraste e solte ou selecione arquivos CSV
- **Análise em tempo real**: Verifica a qualidade de cada número usando a API do Facebook
- **Interface moderna**: Design responsivo e intuitivo
- **Relatórios visuais**: Cards de resumo com estatísticas
- **Exportação**: Exporta resultados em formato CSV
- **Tratamento de erros**: Gerencia diferentes tipos de erros da API

## 📋 Pré-requisitos

- Node.js (versão 16 ou superior)
- Access Token do Facebook Graph API
- Arquivo CSV com os números para análise

## 🛠️ Instalação

1. Clone ou baixe o projeto
2. Navegue até a pasta do frontend:
   ```bash
   cd frontend
   ```

3. Instale as dependências:
   ```bash
   npm install
   ```

4. Execute a aplicação:
   ```bash
   npm run dev
   ```

5. Abra o navegador em `http://localhost:5173`

## 📁 Formato do Arquivo CSV

O arquivo CSV deve conter as seguintes colunas:

```csv
Nome,Número,Cod Telefone
João Silva,5511999999999,123456789012345
Maria Santos,5511888888888,987654321098765
```

### Colunas obrigatórias:
- **Nome**: Nome do contato (opcional)
- **Número**: Número de telefone (opcional)
- **Cod Telefone**: Código do telefone no WhatsApp Business API (obrigatório)

## 🔑 Configuração do Access Token

1. Acesse o [Facebook Developers](https://developers.facebook.com/)
2. Crie um novo app ou use um existente
3. Configure as permissões necessárias para a WhatsApp Business API
4. Gere um Access Token com as permissões adequadas
5. Cole o token no campo "Access Token do Facebook" na aplicação

## 🎯 Como Usar

1. **Configure o Access Token**:
   - Insira seu access token do Facebook na seção de configuração

2. **Selecione a versão da API**:
   - Escolha a versão da API do Facebook Graph (recomendado: v23.0)

3. **Faça upload do arquivo CSV**:
   - Arraste e solte o arquivo na área indicada
   - Ou clique em "Selecionar Arquivo" para escolher manualmente

4. **Inicie a análise**:
   - Clique em "Analisar Números"
   - A aplicação processará cada número individualmente
   - Aguarde a conclusão da análise

5. **Visualize os resultados**:
   - Veja o resumo nos cards coloridos
   - Analise a tabela detalhada com todos os resultados
   - Exporte os dados se necessário

## 📊 Interpretação dos Resultados

### Status de Qualidade:
- 🟢 **BOM**: Número com boa qualidade
- 🟡 **MÉDIO**: Número com qualidade intermediária
- 🔴 **RUIM**: Número com baixa qualidade

### Status de Processamento:
- ✅ **SUCESSO**: Análise concluída com sucesso
- ❌ **NAO_ENCONTRADO**: Número não encontrado na API
- 🔐 **ERRO_AUTENTICACAO**: Token inválido ou sem permissões
- 🌐 **ERRO_CONEXAO**: Problemas de conectividade

## 📤 Exportação

A aplicação permite exportar os resultados em:
- **CSV**: Formato padrão para análise em planilhas
- **Excel**: Funcionalidade em desenvolvimento

## ⚠️ Limitações e Considerações

- **Rate Limiting**: A API do Facebook tem limites de requisições
- **Token Expiração**: Access tokens podem expirar
- **Permissões**: Certifique-se de ter as permissões corretas na API
- **Tamanho do arquivo**: Arquivos muito grandes podem demorar para processar

## 🛠️ Tecnologias Utilizadas

- **Vue.js 3**: Framework frontend
- **Vite**: Build tool
- **Axios**: Cliente HTTP
- **PapaParse**: Parser de CSV
- **CSS3**: Estilização moderna

## 📝 Estrutura do Projeto

```
frontend/
├── src/
│   ├── App.vue          # Componente principal
│   ├── main.js          # Ponto de entrada
│   └── assets/          # Recursos estáticos
├── public/              # Arquivos públicos
├── package.json         # Dependências
└── README.md           # Este arquivo
```

## 🔧 Desenvolvimento

Para contribuir com o projeto:

1. Faça um fork do repositório
2. Crie uma branch para sua feature
3. Implemente as mudanças
4. Teste a aplicação
5. Faça um pull request

## 📞 Suporte

Para dúvidas ou problemas:
- Verifique a documentação da API do Facebook
- Consulte os logs do console do navegador
- Verifique se o access token está válido

## 📄 Licença

Este projeto é de uso livre para fins educacionais e comerciais.
