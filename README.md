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
- **v2.0 – 12/12/2025:** Ajuste versão final
### 1.4 Datas
- **Criação:** 20/11/2025
- **Última atualização:** 12/12/2025

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
## 6. Riscos de alto nível, premissas e critérios de sucesso

### 6.1 Riscos de alto nível (negócio, técnicos, etc.)
- **Risco de Hardware (Saturação de Disco):** O volume de dados gerado pelo UUIDv4 (devido à fragmentação) pode exceder o espaço disponível no ambiente de teste, interrompendo a execução.
- **Risco de Ruído no Ambiente (Noisy Neighbor):** Processos do Sistema Operacional (atualizações, indexação de arquivos, antivírus) podem consumir I/O concorrente, distorcendo as métricas de latência.
- **Risco de Estabilidade do Cliente:** O script gerador de carga (Python) pode sofrer *Out-Of-Memory* ou gargalos de CPU antes do banco de dados, tornando-se o fator limitante.

### 6.2 Critérios de sucesso globais (go / no-go)
- **Sucesso Técnico:** O experimento será considerado válido se for possível concluir a ingestão de 50 milhões de registros para ambos os algoritmos (v4 e v7) sem erros fatais ou corrupção de dados no SGBD.
- **Sucesso Analítico:** Obtenção de dados estatisticamente significantes (p < 0.05 ou intervalos de confiança claros) que permitam diferenciar o desempenho dos dois algoritmos.

### 6.3 Critérios de parada antecipada (pré-execução)
- Detecção de falhas na unicidade dos UUIDs gerados pela biblioteca.
- Falta de espaço livre em disco inferior a 150GB antes do início dos testes.
- Incapacidade de configurar o limite de memória do container Docker (cgroups não funcionando).

---

## 7. Modelo conceitual e hipóteses

### 7.1 Modelo conceitual do experimento
O modelo baseia-se na mecânica das árvores B+ (B-Trees) utilizadas pelo InnoDB.
- **Cenário A (UUIDv7 - Time-ordered):** A inserção ocorre de forma quase sequencial (monotonicamente crescente). O SGBD preenche as páginas de dados da direita para a esquerda até o *fill factor* ideal (~93%), minimizando operações de I/O aleatórias e maximizando o uso do cache.
- **Cenário B (UUIDv4 - Aleatório):** A inserção ocorre em posições aleatórias da árvore. Isso obriga o SGBD a carregar páginas antigas do disco para a memória (Random Read) e realizar divisões de página (*Page Splits*) frequentes, resultando em páginas com ocupação parcial (~50-70%) e arquivos fisicamente maiores.

### 7.2 Hipóteses formais (H0, H1)
- **H0 (Nula):** $\mu_{v7} = \mu_{v4}$ e $Size_{v7} = Size_{v4}$. (Não há diferença significativa na latência média de inserção ou no tamanho final do índice entre os algoritmos).
- **H1 (Desempenho):** $\mu_{v7} < \mu_{v4}$. (A latência média de inserção do UUIDv7 é estatisticamente menor que a do UUIDv4, especialmente sob carga alta).
- **H2 (Armazenamento):** $Size_{v7} < Size_{v4}$. (O tamanho físico ocupado pelo índice UUIDv7 é menor devido à ausência de fragmentação interna excessiva).

### 7.3 Nível de significância e considerações de poder
- **Nível de Significância ($\alpha$):** 0,05 (5%).
- **Poder Estatístico:** Dado o tamanho da amostra (50 milhões de registros), o poder estatístico é extremamente alto. A análise focará na **magnitude do efeito** (diferença prática percentual) e não apenas na rejeição da hipótese nula.

---

## 8. Variáveis, fatores, tratamentos e objetos de estudo

### 8.1 Objetos de estudo
Instâncias de tabelas relacionais no motor InnoDB (MySQL 8.x) configuradas com `innodb_file_per_table=1`.

### 8.2 Sujeitos / participantes (visão geral)
Não aplicável (experimento computacional *in silico*). O "sujeito" é o Sistema Gerenciador de Banco de Dados.

### 8.3 Variáveis independentes (fatores) e seus níveis
- **Fator A: Algoritmo de Chave Primária.**
    - Nível A1: UUIDv4 (Aleatório).
    - Nível A2: UUIDv7 (Ordenado por tempo).
- **Fator B: Volume de Carga (Cardinalidade).**
    - Nível B1: 1.000.000 registros (Carga Inicial).
    - Nível B2: 10.000.000 registros (Carga Média).
    - Nível B3: 50.000.000 registros (Estresse/Saturação).

### 8.4 Tratamentos (condições experimentais)
Combinação dos fatores resultando em 6 tratamentos únicos (Runs T1 a T6), conforme detalhado na seção 4.2.

### 8.5 Variáveis dependentes (respostas)
- Latência de Inserção (Tempo de Resposta).
- Throughput (Transações por Segundo).
- Tamanho Físico do Índice (MB/GB).
- Contagem de *Page Splits*.

### 8.6 Variáveis de controle / bloqueio
- Hardware (CPU, RAM limitada a 1GB, Disco SSD).
- Configuração do MySQL (Buffer Pool, Log File Size fixos).
- Cliente de Carga (Mesmo script, mesma versão do Python).

### 8.7 Possíveis variáveis de confusão conhecidas
- *Cold Start* do banco de dados (mitigado por *warm-up*).
- Garbage Collection do Python (mitigado por medição interna no banco via `performance_schema`).

---

## 9. Desenho experimental

### 9.1 Tipo de desenho (completamente randomizado, blocos, fatorial, etc.)
**Fatorial Completo $2 \times 3$.** Este desenho permite analisar o efeito principal de cada algoritmo e, crucialmente, a interação entre o algoritmo e o volume de dados (ex: a degradação do v4 acelera mais rápido que a do v7?).

### 9.2 Randomização e alocação
A ordem de execução dos blocos (Runs completos de v4 ou v7) será alternada ou sorteada para evitar vieses temporais da máquina hospedeira, mas cada Run começa com um banco de dados "zerado" (DROP/CREATE).

### 9.3 Balanceamento e contrabalanço
Os grupos são perfeitamente balanceados: ambos processam exatamente a mesma quantidade de registros e o mesmo *payload* de dados não-chave.

### 9.4 Número de grupos e sessões
- **Grupos:** 2 (v4 e v7).
- **Sessões:** Cada carga completa até 50M é uma sessão. Serão realizadas no mínimo 3 repetições (sessões) completas para cada grupo para garantir consistência dos dados.

---

## 10. População, sujeitos e amostragem

### 10.1 População-alvo
Sistemas de banco de dados transacionais (OLTP) de alta performance que utilizam armazenamento em B-Trees.

### 10.2 Critérios de inclusão de sujeitos
Registros sintéticos contendo IDs válidos segundo as RFCs (4122 e 9562) e payload de texto aleatório simulando dados reais.

### 10.3 Critérios de exclusão de sujeitos
Execuções interrompidas por falhas externas (energia, crash do SO) serão descartadas integralmente.

### 10.4 Tamanho da amostra planejado (por grupo)
50.000.000 (Cinquenta milhões) de operações de escrita (amostras) por grupo, por repetição.

### 10.5 Método de seleção / recrutamento
Geração de dados sintéticos via algoritmos pseudo-aleatórios (`secrets`, `faker`).

### 10.6 Treinamento e preparação dos sujeitos
Não aplicável.

---

## 11. Instrumentação e protocolo operacional

### 11.1 Instrumentos de coleta (questionários, logs, planilhas, etc.)
- **Script de Carga (Python):** Gera carga e registra latência percebida pelo cliente em arquivos CSV (`client_metrics.csv`).
- **MySQL Performance Schema:** Coleta métricas precisas de I/O e tempo de execução interno (`db_metrics.csv`).
- **Shell Scripts:** Monitoram uso de recursos do SO (`iostat`, `docker stats`) a cada segundo.

### 11.2 Materiais de suporte (instruções, guias)
- `README.md` no repositório com instruções de setup.
- Arquivo `docker-compose.yml` padronizado.

### 11.3 Procedimento experimental (protocolo – visão passo a passo)
1.  **Setup:** Inicializar container MySQL com limite de RAM e criar esquema da tabela.
2.  **Warm-up:** Inserir 10.000 registros e descartar métricas.
3.  **Execução:** Iniciar loop de inserção em lotes (batch size = 1000).
4.  **Coleta Contínua:** Registrar latência de cada lote.
5.  **Pontos de Controle:** Ao atingir 1M, 10M e 50M, pausar carga por 10s, forçar *flush* e medir tamanho dos arquivos `.ibd` e contadores de *Page Splits*.
6.  **Teardown:** Exportar logs, destruir container e volumes.
7.  **Repetição:** Reiniciar processo para o próximo tratamento.

### 11.4 Plano de piloto (se haverá piloto, escopo e critérios de ajuste)
Será realizado um piloto com 500.000 registros para validar o pipeline de coleta de dados e garantir que a gravação de logs não introduz *overhead* significativo.

---

## 12. Plano de análise de dados (pré-execução)

### 12.1 Estratégia geral de análise (como responderá às questões)
A análise será quantitativa comparativa. Serão gerados gráficos de linha (Eixo X: Quantidade de Linhas, Eixo Y: Latência/Tamanho) para visualizar a divergência entre as curvas do UUIDv4 e UUIDv7.

### 12.2 Métodos estatísticos planejados
- Cálculo de estatísticas descritivas: Média, Mediana, Desvio Padrão, Percentil 95 e 99.
- Cálculo de Delta Percentual ($\% \Delta$) de melhoria/piora entre os grupos.

### 12.3 Tratamento de dados faltantes e outliers
- Dados faltantes invalidam o Run.
- Outliers extremos (> 5 segundos) serão investigados; se correlacionados com *freezes* do SO host, serão removidos da média (trimming), mas relatados nos anexos.

### 12.4 Plano de análise para dados qualitativos (se houver)
Não aplicável.

---

## 13. Avaliação de validade (ameaças e mitigação)

### 13.1 Validade de conclusão
- *Ameaça:* Variância natural de desempenho do SSD.
- *Mitigação:* Executar múltiplas rodadas (3x) e calcular a média dos resultados.

### 13.2 Validade interna
- *Ameaça:* Interferência de processos no Host (SO).
- *Mitigação:* Executar em ambiente dedicado ou durante período de inatividade, desligando serviços não essenciais.

### 13.3 Validade de constructo
- *Ameaça:* A latência do cliente inclui tempo de rede (localhost) e serialização Python.
- *Mitigação:* Monitorar métricas internas do banco (`wait_events`) para isolar o tempo real de processamento do DB.

### 13.4 Validade externa
- *Ameaça:* Hardware local (NVMe) é diferente de Storage de Nuvem (EBS).
- *Mitigação:* Focar nas métricas lógicas (IOPS, Page Splits) que independem da velocidade do disco, garantindo generalização.

### 13.5 Resumo das principais ameaças e estratégias de mitigação
- **Host Noise:** Isolamento e repetição.
- **Client Bottleneck:** Monitoramento de CPU do cliente.
- **Cache Warm-up:** Procedimento de *Warm-up* antes da coleta.

---

## 14. Ética, Privacidade e Conformidade

### 14.1 Questões éticas (uso de sujeitos, incentivos, etc.)
O experimento não envolve seres humanos ou animais.

### 14.2 Consentimento informado
Não aplicável.

### 14.3 Privacidade e proteção de dados
Todos os dados utilizados são sintéticos (falsos), gerados aleatoriamente. Não há processamento de PII (Personal Identifiable Information).

### 14.4 Aprovações necessárias (comitê de ética, jurídico, DPO, etc.)
Aprovação técnica do PI e/ou Orientador Acadêmico.

---

## 15. Recursos, Infraestrutura e Orçamento

### 15.1 Recursos humanos e papéis
- **Engenheiro de Software Sênior (Autor):** Planejamento, execução, coleta e análise.

### 15.2 Infraestrutura técnica necessária
- Workstation Linux (Ubuntu/Pop!_OS).
- Docker Engine & Docker Compose.
- Python 3.10+.
- Mínimo de 150GB de armazenamento livre.

### 15.3 Materiais e insumos
- Bibliotecas Open Source (`mysql-connector-python`, `uuid6`, `numpy`, `pandas`).

### 15.4 Orçamento e custos estimados
- **Financeiro:** Custo zero (utilização de hardware e software já disponíveis).
- **Horas:** Estimativa de 40-60 horas de dedicação técnica.

---

## 16. Cronograma, marcos e riscos operacionais

### 16.1 Macrocronograma (até o início da execução)
- **20/11 - 25/11:** Planejamento e Design (Concluído).
- **26/11 - 01/12:** Desenvolvimento dos Scripts.
- **02/12 - 05/12:** Teste Piloto e Ajustes.
- **06/12:** Início da Execução Oficial (Coleta).

### 16.2 Dependências entre atividades
A Execução Oficial (06/12) depende estritamente da validação dos scripts no Piloto.

### 16.3 Riscos operacionais e plano de contingência
- **Falha de Hardware:** Utilizar backup em nuvem (instância AWS EC2 Spot) caso a workstation falhe.
- **Perda de Dados:** Logs são salvos em CSV a cada lote; em caso de falha, reinicia-se apenas o Run afetado.

---

## 17. Governança do experimento

### 17.1 Papéis e responsabilidades formais
O PI (João Victor Temponi) é responsável pela integridade técnica e decisão de *Go/No-Go*.

### 17.2 Ritos de acompanhamento pré-execução
Revisão de código e *Dry-run* (simulação) dos scripts antes da coleta massiva.

### 17.3 Processo de controle de mudanças no plano
Qualquer alteração nas variáveis (ex: tamanho do payload) exige atualização deste documento e descarte de dados prévios para manter a consistência.

---

## 18. Plano de documentação e reprodutibilidade

### 18.1 Repositórios e convenções de nomeação
- GitHub: Repositório contendo `/src`, `/docs` e `/results`.
- Nomenclatura: `run_{data}_{algoritmo}_{volume}.csv`.

### 18.2 Templates e artefatos padrão
- `Makefile` com comandos padronizados (`make setup`, `make run-v4`, `make run-v7`).
- `requirements.txt` com versões pinadas das bibliotecas.

### 18.3 Plano de empacotamento para replicação futura
O projeto incluirá um *Container Dev* ou instruções claras de Docker para que qualquer pesquisador possa replicar o ambiente exato com um único comando.

---

## 19. Plano de comunicação

### 19.1 Públicos e mensagens-chave pré-execução
- Stakeholders Técnicos: Comunicar o objetivo de validação de performance e potencial economia de infraestrutura.

### 19.2 Canais e frequência de comunicação
- Atualizações via commits no repositório e relatório final em formato Markdown/PDF.

### 19.3 Pontos de comunicação obrigatórios
- Início da Execução.
- Conclusão da Coleta.
- Publicação dos Resultados.

---

## 20. Critérios de prontidão para execução (Definition of Ready)

### 20.1 Checklist de prontidão (itens que devem estar completos)
- [ ] Plano experimental aprovado (v1.2).
- [ ] Scripts de automação (Carga e Métricas) desenvolvidos e sem erros no linter.
- [ ] Ambiente Docker configurado com limitação de memória testada.
- [ ] Disco com espaço suficiente verificado.
- [ ] Piloto executado com sucesso.

### 20.2 Aprovações finais para iniciar a operação
- **Aprovado por:** João Victor Temponi Daltro de Castro.
- **Data da Aprovação:** 12/12/2025.
  
## Fluxograma de Operacionalização

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
