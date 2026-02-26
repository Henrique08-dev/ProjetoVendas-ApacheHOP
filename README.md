# 📦 Projeto Data Warehouse de Vendas com Apache Hop

## 📋 Sobre o Projeto

Este projeto consiste na construção completa de um **Data Warehouse de Vendas** utilizando **Apache Hop** como ferramenta de orquestração e ETL. O objetivo foi resolver problemas de performance e estruturação de dados de um cliente que possuía grande volume de informações em um banco PostgreSQL, necessitando de uma solução escalável e estruturada para análises estratégicas.

**Período:** 2026 
**Tecnologias:** Apache Hop | PostgreSQL | SQL Server | DBeaver | Telegram Bot API

---

## 🎯 Objetivos do Projeto

- ✅ Criar um **Data Warehouse estruturado** para melhor performance analítica
- ✅ Implementar **cargas incrementais** para otimização de processos
- ✅ Desenvolver **sistema de monitoramento** com alertas em tempo real
- ✅ Garantir **rastreabilidade** através de logs detalhados
- ✅ Separar **ambientes DEV e PROD** para segurança
- ✅ Tratar **dados inconsistentes** e garantir integridade referencial

---

## 🏗️ Arquitetura do Projeto

### 🔹 **Fluxo de Dados**
```
PostgreSQL (Origem) 
    → Apache Hop (ETL) 
        → SQL Server (Stage) 
            → Apache Hop (Transformações) 
                → SQL Server (Data Warehouse)
```

### 🔹 **Estrutura de Bancos**

**🟢 PostgreSQL (Origem)**
- Tabelas: categorias, clientes, devoluções, produtos, subcategorias, territórios, vendas
- Volume: ~3,8M registros de vendas

**🟡 SQL Server (Stage)**
- Esquema: stage
- Tabelas temporárias: tb_categorias, tb_clientes, tb_devolucoes, tb_logs, tb_metas, tb_produtos, tb_subcategorias, tb_territorios, tb_vendas

**🔵 SQL Server (Data Warehouse)**
- Esquema: dbo
- Tabelas dimensionais: dCalendario, dCliente, dProduto, dSubcategoria, dTerritorio
- Tabelas fato: fDevolucoes, fMetas, fVendas

---

## 📊 Modelagem Dimensional (Star Schema)

### 🔹 **Tabelas Dimensão**

```sql
table dCliente {
  IdCliente int PK
  NomeCompleto varchar(100)
  DataNascimento date
  Idade int
  EstadoCivil varchar(100)
  Email varchar(50)
  Sexo varchar(50)
  NivelEducacao varchar(50)
  Ocupacao varchar(50)
  DataCadastro date
  DataCriacaoRegistro datetime
}

table dTerritorio {
  IdTerritorio int PK
  Regiao varchar(50)
  Pais varchar(50)
  GrupoTerritorio varchar(50)
  DataCriacaoRegistro datetime
}

table dProduto {
  SkProduto int PK
  IdProduto int
  Sku varchar(50)
  Produto varchar(50)
  Modelo varchar(50)
  DescricaoProduto varchar(256)
  Cor varchar(50)
  Tamanho varchar(50)
  Estilo varchar(50)
  CustoUnitario float
  PrecoUnitario float
  DataInicio datetime
  DataFim datetime
  Versao int
  DataCriacaoRegistro datetime
}

table dSubcategoria {
  IdSubcategoria int PK
  Subcategoria varchar(50)
  IdCategoria int
  Categoria varchar(50)
  DataCriacaoRegistro datetime
}

table dCalendario {
  Data date PK
  Ano int
  Mes int
  MesNome varchar(9)
  Trimestre int
  TrimestreNome varchar(18)
  Semestre int
  SemestreNome varchar(17)
  DataCriacaoRegistro datetime
}
```

### 🔹 **Tabelas Fato**

```sql
table fVendas {
  DataPedido date [ref: > dCalendario.Data]
  DataEstoque date
  NumeroPedido varchar(7) PK
  Sk_Produto int [ref: > dProduto.SkProduto]
  Id_Subcategoria int [ref: > dSubcategoria.IdSubcategoria]
  Id_Cliente int [ref: > dCliente.IdCliente]
  Id_Territorio int [ref: > dTerritorio.IdTerritorio]
  ItemLinhaPedido int PK
  Quantidade int
  DataCriacaoRegistro datetime
}

table fDevolucoes {
  IdDevolucao PK
  DataDevolucao date [ref: > dCalendario.Data]
  Id_Territorio int [ref: > dTerritorio.IdTerritorio]
  Sk_Produto int [ref: > dProduto.SkProduto]
  QuantidadeDevolucoes int
  DataCriacaoRegistro datetime
}

table fMetas {
  DataMeta date PK [ref: > dCalendario.Data]
  Valor float
  DataCriacaoRegistro datetime
}
```

---

## 🔧 Pipelines Desenvolvidas no Apache Hop

### 📁 **Workflow Stage** (8 pipelines)
- `stage_categories.hpl` - Carga de categorias
- `stage_vendas.hpl` - Carga de vendas
- `stage_territorios.hpl` - Carga de territórios
- `stage_subcategories.hpl` - Carga de subcategorias
- `stage_produtos.hpl` - Carga de produtos
- `stage_metas.hpl` - Carga de metas (Excel → Stage)
- `stage_clientes.hpl` - Carga de clientes
- `stage_devolucoes.hpl` - Carga de devoluções

### 📁 **Workflow Dimensões** (5 pipelines)
- `dCalendario.hpl` - Geração da dimensão tempo
- `dTerritorio.hpl` - Carga dimensão território
- `dSubcategorias.hpl` - Carga dimensão subcategoria (com merge join)
- `dProdutos.hpl` - Carga dimensão produto (SCD Tipo 2)
- `dCliente.hpl` - Carga dimensão cliente (deduplicação)

### 📁 **Workflow Fatos** (3 pipelines)
- `fVendas.hpl` - Carga fato vendas
- `fMetas.hpl` - Carga fato metas
- `fDevolucoes.hpl` - Carga fato devoluções

### 📁 **Workflow Principal**
```
Start 
    → API Telegram Inicio 
    → dataincremental.hpl 
    → workflowstage.hwf 
    → workflowdimensoes.hwf 
    → workflowfatos.hwf 
    → API Telegram Fim 
    → Mail-Success 
    → Success
```

**Tratamento de Erros:**
```
Mail-Erro Stage → Abort workflow
Mail-Erro Dimensoes → Abort workflow
Mail-Erro Fatos → Abort workflow
API Telegram Erro
```

---

## 🚀 Principais Desafios Técnicos

### 1. **Tratamento de Dados Duplicados (Clientes)**
- Identificação de IDs de cliente duplicados
- Utilização de `Sort Rows` + `Unique Rows` para manter apenas o cadastro mais recente
- Concatenação de nome com `Concat` para criar `NomeCompleto`

### 2. **SCD Tipo 2 (Produtos)**
- Implementação de histórico de alterações com `Dimension Lookup/Update`
- Rastreamento de mudanças por `IdProduto`
- Controle de versão com `DataInicio`, `DataFim` e `Versao`

### 3. **Merge entre Categorias e Subcategorias**
- Dois `Table Input` (categoria e subcategoria)
- `Sort Rows` em ambas para ordenação
- `Merge Join` via `IdCategoria`

### 4. **Geração da Dimensão Calendário**
- `Generate Rows` a partir da primeira data de venda (2015-02-02)
- `Add Sequence` para 10.000 linhas
- `Calculator` para extrair ano, mês e data
- `Mapper Values` para criar nomes de meses, trimestres e semestres

### 5. **Carga Incremental**
- Pipeline `dataincremental.hpl` com SQL:
```sql
SELECT DATEADD(DAY, -30, MAX(DataPedido)) 
FROM dw.[dbo].fVendas
```
- Carrega apenas registros dos últimos 30 dias para performance

### 6. **Validação de Integridade Referencial**
- Identificação de `IdTerritorio = 10` sem correspondente na dimensão
- Documentação para tratamento futuro

---

## 📱 Sistema de Monitoramento

### 🔹 **Alertas via Telegram**
- Bot criado com **BotFather**
- Grupo dedicado para notificações
- Mensagens para:
  - ✅ Início do processo
  - ✅ Execução com sucesso
  - ❌ Erros específicos

**Exemplo de notificações:**
```
Data Warehouse Iniciando Processo 18:14
Data Warehouse Executado com Sucesso 18:14
```

### 🔹 **Alertas via E-mail**
- `Mail-Success`: Workflow concluído com sucesso
- `Mail-Erro Stage`: Falha na etapa de stage
- `Mail-Erro Dimensoes`: Falha nas dimensões
- `Mail-Erro Fatos`: Falha nas tabelas fato

### 🔹 **Logs de Execução**
- Pipeline dedicada com `Workflow Logging`
- Armazenamento na tabela `tb_logs` (stage)
- Rastreabilidade completa de todas as execuções

---

## 🌐 Ambientes: DEV e PROD

### 🔹 **Configuração**
- **DEV:** SQL Server porta padrão
- **PROD:** SQL Server porta diferente
- Conexões configuradas no Apache Hop com credenciais específicas

### 🔹 **Estratégia**
- Desenvolvimento validado em DEV
- Promoção para PROD após testes
- Mesma lógica, diferentes conexões

---

## ⚙️ Orquestração e Agendamento

### 🔹 **Hop Server**
- Configuração para execução remota
- Processos rodando independentemente da máquina local

### 🔹 **Agendador de Tarefas**
- Execução automática do workflow principal
- Periodicidade definida conforme necessidade do cliente

---

## 📂 Estrutura do Repositório

```
├── /pipelines
│   ├── /stage
│   │   ├── stage_categories.hpl
│   │   ├── stage_vendas.hpl
│   │   └── ...
│   ├── /dimensoes
│   │   ├── dCalendario.hpl
│   │   ├── dCliente.hpl
│   │   └── ...
│   ├── /fatos
│   │   ├── fVendas.hpl
│   │   ├── fMetas.hpl
│   │   └── fDevolucoes.hpl
│   └── /workflows
│       ├── workflowstage.hwf
│       ├── workflowdimensoes.hwf
│       ├── workflowfatos.hwf
│       └── workflowprincipal.hwf
└── README.md
```

---

## 🚀 Como Executar o Projeto

### Pré-requisitos
- Apache Hop instalado
- PostgreSQL (origem)
- SQL Server (stage e DW)
- DBeaver (opcional, para visualização)

### Passo a Passo

1. **Clone o repositório**
```bash
git clone https://github.com/seu-usuario/projeto-hop-vendas.git
```

2. **Configure os bancos de dados**
- Execute scripts em `/scripts/create_tables_stage.sql` no stage
- Execute scripts em `/scripts/create_tables_dw.sql` no DW
- Carregue dados de origem no PostgreSQL

3. **Configure as conexões no Apache Hop**
- PostgreSQL (origem)
- SQL Server Stage (DEV/PROD)
- SQL Server DW (DEV/PROD)

4. **Execute o workflow principal**
- Abra o Apache Hop
- Importe o projeto
- Execute `workflowprincipal.hwf`

5. **Monitore as execuções**
- Acompanhe logs na tabela `tb_logs`
- Verifique notificações no Telegram
- Confirme e-mails de sucesso/erro

---

## 📊 Resultados Alcançados

✅ **Performance:** Carga incremental otimizando tempo de execução  
✅ **Integridade:** Tratamento de duplicatas e validação de chaves estrangeiras  
✅ **Monitoramento:** Alertas em tempo real via Telegram + e-mail  
✅ **Rastreabilidade:** Logs completos de todas as execuções  
✅ **Escalabilidade:** Arquitetura preparada para crescimento  
✅ **Segurança:** Ambientes DEV e PROD isolados  

**Volumes processados:**
- 📦 **3,8M** registros de vendas
- 👥 **2,2M** clientes
- 📊 **5M** registros na fVendas (DW)
- 🔍 **1,8M** registros de logs

---

## 💡 Principais Aprendizados

- **Apache Hop** é uma ferramenta poderosa e flexível para orquestração ETL
- **Modelagem dimensional** bem planejada evita retrabalho
- **Monitoramento proativo** é essencial para ambientes de produção
- **Carga incremental** melhora drasticamente a performance
- **Tratamento de dados inconsistentes** deve ser prioridade na fase de stage

---

## 📱 Contato

**Henrique Albuquerque Araújo**  
📧 he.fla3@gmail.com  
🔗 [LinkedIn](https://www.linkedin.com/in/henrique-dev)  
🐙 [GitHub](https://github.com/Henrique08-dev)

---

**📊 Transformando dados em decisões estratégicas!**
