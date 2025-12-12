# Análise Comparativa de Degradação de Performance de Escrita e Tamanho de Índice: UUIDv4 (Aleatório) vs. UUIDv7 (Time-ordered) em SGBD Relacional

## 1. Identificação

### 1.1 Título do Experimento
Análise Comparativa de Degradação de Performance de Escrita e Tamanho de Índice: UUIDv4 (Aleatório) vs. UUIDv7 (Time-ordered) em SGBD Relacional

### 1.2 ID
`UUIDv4-vs-UUIDv7-EXP-MED`

### 1.3 Versão do Documento e Histórico de Revisão
- **v1.0 – 20/11/2025:** Criação inicial do planejamento.
- **v1.1 – 21/11/2025:** Ajuste no título para remover menção específica a microserviços.
- **v1.2 – 02/12/2025:** Ajuste no conteúdo para MySQL
### 1.4 Datas
- **Criação:** 20/11/2025
- **Última atualização:** 02/12/2025

### 1.5 Autores
- **Autor Principal:** João Victor Temponi Daltro de Castro (Eng. Software)
- **Contato:** jvictortemponi@gmail.com

### 1.6 Responsável Principal (PI)
- **PI:** João Victor Temponi Daltro de Castro

### 1.7 Projeto
Estudo sobre Padrões de Identidade e Otimização de Estruturas de Dados em Bancos Relacionais.

---

## 2. Contexto e Problema

### 2.1 Descrição
Identificadores aleatórios (UUIDv4) geram um padrão de acesso não sequencial em estruturas de índice B-Tree. Diferente de inserções ordenadas que preenchem páginas de memória de maneira sequencial, UUIDs aleatórios exigem carregamento frequente de páginas arbitrárias do disco para a memória e provocam divisões de páginas (*page splits*).

O objetivo é medir o custo oculto dessa aleatoriedade: o aumento desproporcional de I/O e o desperdício de armazenamento conforme a tabela cresce, comparado ao padrão UUIDv7 ordenado por tempo.

### 2.2 Contexto Organizacional e Técnico
- **Ambiente do Experimento:** Ambiente controlado e isolado para evitar interferência de outras aplicações.
- **SGBD:** MySQL, escolhido como representativo de bancos relacionais amplamente utilizados. Resultados com aplicabilidade também para SQL Server, PostgreSQL etc.
- **Cenário de Teste:** Simulação de carga focada em operações de escrita de alta ingestão, comuns em logs, IoT e sistemas financeiros.
- **Configuração:** Hardware com limitação conhecida de RAM, forçando operações de I/O em disco, onde as diferenças entre UUIDv4 e UUIDv7 se tornam mais evidentes.

### 2.3 Trabalhos e Evidências Prévias
- **Literatura Técnica:** A especificação IETF RFC 9562 argumenta que identificadores ordenados por tempo melhoram a localidade de referência em B-Trees.
- **Benchmarks de bancos de dados:** Estudos de índices clusterizados mostram que chaves aleatórias causam *page splits* excessivos, resultando em índices maiores e mais lentos quando comparados com chaves sequenciais (ex.: BIGINT).

### 2.4 Referencial Teórico e Empírico Essencial
- **B-Tree & Page Splitting:** Índices B-Tree crescem eficientemente quando novos valores são adicionados à direita da árvore (valores crescentes). Valores aleatórios exigem inserções no meio de nós já preenchidos, causando *splits*, aumentando custo de CPU e o tamanho físico do arquivo.
- **Localidade de Cache:** O desempenho do banco depende fortemente do *buffer pool*. Acesso aleatório reduz o *cache hit ratio*, forçando leituras/escritas lentas em disco.
- **K-Sortable IDs:** Conceito de identificadores aproximadamente ordenáveis como UUIDv7, que combinam a unicidade do UUID com o desempenho de chaves sequenciais.

---

## 3. Planejamento Experimental e Metodologia

### 3.1 Abordagem GQM (Goal-Question-Metric)

A tabela abaixo detalha o alinhamento entre os objetivos estratégicos do experimento, as questões de pesquisa formuladas e as métricas quantitativas que responderão a essas questões.

| Objetivo (Goal) | Questão (Question) | Métrica Associada (Metric) |
| :--- | :--- | :--- |
| **G1. Analisar a degradação de performance de escrita (Write Latency)**<br>Avaliar o quanto a aleatoriedade do UUIDv4 impacta o tempo de resposta conforme o volume de dados aumenta. | **Q1.1** Qual é a latência média de inserção para cada tipo de UUID? | • M01. Average Insert Latency<br>• M02. Throughput (TPS) |
| | **Q1.2** Como a latência se comporta nos piores casos (cauda longa)? | • M03. P99 Latency<br>• M01. Average Insert Latency |
| | **Q1.3** Qual a taxa máxima de transações sustentada antes da saturação? | • M02. Throughput (TPS)<br>• M06. CPU Usage |
| **G2. Avaliar a eficiência de armazenamento em disco**<br>Determinar o impacto da fragmentação de índice no tamanho físico dos arquivos de dados e índices. | **Q2.1** Qual o tamanho final do índice da chave primária (Clustered Index)? | • M04. Primary Key Index Size<br>• M09. Total Table Size |
| | **Q2.2** Qual a proporção de espaço desperdiçado devido à fragmentação? | • M08. Data Free Space (Fragmentation)<br>• M04. Primary Key Index Size |
| | **Q2.3** Como o tamanho total da tabela escala em relação ao número de linhas? | • M09. Total Table Size<br>• M10. Rows Count |
| **G3. Mensurar o impacto nos recursos de I/O e Buffer Pool**<br>Verificar se a falta de localidade do UUIDv4 força mais leituras/escritas em disco e polui o cache. | **Q3.1** Qual a frequência de operações físicas de leitura no disco? | • M05. Disk Read Operations (IOPS)<br>• M07. Buffer Pool Hit Rate |
| | **Q3.2** O Buffer Pool está sendo utilizado eficientemente? | • M07. Buffer Pool Hit Rate<br>• M11. Page Eviction Rate |
| | **Q3.3** Qual o impacto na fila de espera do disco (IO Wait)? | • M12. Disk IO Wait Time<br>• M05. Disk Read Operations (IOPS) |
| **G4. Investigar o esforço estrutural do SGBD (Page Splits)**<br>Quantificar o trabalho extra realizado pelo motor do banco para manter a árvore B-Tree balanceada. | **Q4.1** Com que frequência ocorrem divisões de página (page splits)? | • M13. InnoDB Page Splits<br>• M02. Throughput (TPS) |
| | **Q4.2** Ocorre contenção excessiva em latches/locks de índice? | • M14. Index Latch Wait Time<br>• M06. CPU Usage |
| | **Q4.3** Qual a eficiência do preenchimento das páginas de dados? | • M15. Page Fill Factor<br>• M13. InnoDB Page Splits |

### 3.2 Definição das Métricas

Abaixo, a lista consolidada e detalhada de todas as métricas citadas no GQM.

| ID | Nome da Métrica | Descrição | Unidade |
| :--- | :--- | :--- | :--- |
| **M01** | Average Insert Latency | Tempo médio para completar uma operação de INSERT no banco. | Milissegundos (ms) |
| **M02** | Throughput (TPS) | Número de transações de escrita concluídas por segundo. | Transações/seg |
| **M03** | P99 Latency | Tempo de resposta abaixo do qual estão 99% das requisições mais rápidas (indica estabilidade). | Milissegundos (ms) |
| **M04** | Primary Key Index Size | Espaço em disco ocupado exclusivamente pela estrutura da B-Tree da chave primária. | Megabytes (MB) |
| **M05** | Disk Read Operations | Quantidade de operações físicas de leitura no disco (quando o dado não está na RAM). | IOPS (Ops/seg) |
| **M06** | CPU Usage | Porcentagem de utilização do processador pelo processo do SGBD. | Porcentagem (%) |
| **M07** | Buffer Pool Hit Rate | Razão entre leituras encontradas na memória (RAM) versus total de leituras solicitadas. | Porcentagem (%) |
| **M08** | Data Free Space | Espaço alocado nas páginas do banco de dados que não contém dados reais (fragmentação). | Megabytes (MB) |
| **M09** | Total Table Size | Soma do tamanho dos dados da tabela + tamanho dos índices. | Megabytes (MB) |
| **M10** | Rows Count | Contagem total de registros inseridos com sucesso. | Unidade (linhas) |
| **M11** | Page Eviction Rate | Taxa na qual páginas são removidas da memória para dar lugar a novas páginas. | Páginas/seg |
| **M12** | Disk IO Wait Time | Tempo que a CPU passa ociosa aguardando resposta do disco. | Milissegundos (ms) |
| **M13** | InnoDB Page Splits | Contador de vezes que uma página da B-Tree teve que ser dividida para acomodar novos dados. | Contagem Absoluta |
| **M14** | Index Latch Wait Time | Tempo de espera por bloqueios internos na estrutura do índice (concorrência). | Milissegundos (ms) |
| **M15** | Page Fill Factor | Percentual médio de ocupação de dados dentro das páginas alocadas. | Porcentagem (%) |

---

## 4. Variáveis e Planejamento Fatorial

### 4.1 Variáveis do Experimento

| Tipo de Variável | Variável | Descrição | Como será tratada |
| :--- | :--- | :--- | :--- |
| **Independente** | Tipo de UUID | O algoritmo utilizado para gerar a chave primária (UUIDv4 ou UUIDv7). | Variável principal de manipulação. |
| **Independente** | Volume de Dados | A quantidade total de registros na tabela durante a medição. | Incrementada progressivamente (Carga). |
| **Dependente** | Performance (Tempo/TPS) | O comportamento de resposta do sistema. | Medida através das métricas M01, M02, M03. |
| **Dependente** | Tamanho Armazenamento | O espaço físico ocupado. | Medida através das métricas M04, M08, M09. |
| **Controlada** | Hardware (CPU/RAM/Disco) | Especificações da máquina onde roda o SGBD. | Fixado durante todo o experimento. RAM limitada para forçar uso de disco. |
| **Controlada** | Configuração do SGBD | Parâmetros do MySQL (ex: `innodb_buffer_pool_size`). | Fixado. Buffer Pool configurado para ser menor que o Dataset final. |
| **Controlada** | Cliente de Carga | O script/software que gera os inserts. | Mesma versão e mesma rede para todos os testes. |

### 4.2 Fatores e Tratamentos

O experimento seguirá um desenho fatorial $2 \times 3$, combinando dois tipos de chaves com três escalas de volume de dados para observar o ponto de degradação.

| Fator | Descrição | Níveis / Tratamentos |
| :--- | :--- | :--- |
| **A: Algoritmo de ID** | Tipo de geração da chave primária. | **A1:** UUIDv4 (Aleatório - Controle)<br>**A2:** UUIDv7 (Ordenado por Tempo) |
| **B: Escala de Carga** | Volume de registros inseridos para teste de estresse. | **B1:** 1 Milhão de linhas (Carga Baixa)<br>**B2:** 10 Milhões de linhas (Carga Média)<br>**B3:** 50 Milhões de linhas (Carga Alta/Saturação) |

**Combinações de Tratamentos (Runs):**
* **T1:** UUIDv4 com 1M linhas.
* **T2:** UUIDv4 com 10M linhas.
* **T3:** UUIDv4 com 50M linhas.
* **T4:** UUIDv7 com 1M linhas.
* **T5:** UUIDv7 com 10M linhas.
* **T6:** UUIDv7 com 50M linhas.

---

## 5. Fluxograma de Operacionalização

O diagrama abaixo ilustra o passo a passo da execução do experimento, desde a preparação do ambiente até a coleta final dos dados.

```mermaid
flowchart TD
    %% Nós de Início e Fim
    Start((Início))
    End((Fim))

    %% Preparação
    subgraph Preparacao [Fase 1: Preparação do Ambiente]
        ConfigHW[Configurar Hardware Limitado<br>RAM < Dataset Final]
        ConfigDB[Configurar SGBD MySQL<br>Tunagem de Buffer Pool Fixa]
        ResetDB[Resetar Instância do Banco<br>DROP DATABASE / FLUSH]
    end

    %% Definição de Cenário
    subgraph Execucao [Fase 2: Execução dos Tratamentos]
        SelectFactor{Selecionar Fator A:<br>Tipo de UUID}
        CreateSchema[Criar Tabela com PK<br>UUIDv4 ou UUIDv7]
        
        subgraph Carga [Loop de Ingestão]
            GenLoad[Gerar Lote de Inserts<br>Script Python/Go]
            ExecInsert[Executar Inserts no DB]
            Monitor[Monitorar em Tempo Real]
        end
        
        CheckVol{Atingiu Volume B?<br>1M / 10M / 50M}
    end

    %% Coleta
    subgraph Coleta [Fase 3: Coleta de Dados]
        SnapshotMetrics[Coletar Métricas Cumulativas<br>Latência, CPU, IOPS]
        AnalyzeStruct[Analisar Estrutura Física<br>Tamanho Índice, Page Splits]
        LogResults[Registrar Resultados no Dataset]
    end

    %% Fluxo de Conexão
    Start --> Preparacao
    ConfigHW --> ConfigDB
    ConfigDB --> SelectFactor
    
    SelectFactor -- A1: UUIDv4 --> ResetDB
    SelectFactor -- A2: UUIDv7 --> ResetDB
    
    ResetDB --> CreateSchema
    CreateSchema --> GenLoad
    GenLoad --> ExecInsert
    ExecInsert --> Monitor
    Monitor --> CheckVol
    
    CheckVol -- Não --> GenLoad
    CheckVol -- Sim (Ponto de Medição) --> SnapshotMetrics
    SnapshotMetrics --> AnalyzeStruct
    AnalyzeStruct --> LogResults
    
    LogResults --> CheckFinal{Todos Níveis<br>Testados?}
    CheckFinal -- Não (Próx. Nível B) --> GenLoad
    CheckFinal -- Sim (Trocar UUID) --> SelectFactor
    
    %% Finalização
    SelectFactor -- Ambos Testados --> Compare[Comparação Estatística<br>v4 vs v7]
    Compare --> End

    %% Estilos e Anotações
    classDef metrics fill:#f9f,stroke:#333,stroke-width:2px;
    class SnapshotMetrics,AnalyzeStruct,Monitor metrics;
    
    classDef vars fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    class ConfigHW,ConfigDB,SelectFactor,CheckVol vars;
