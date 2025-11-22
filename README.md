# Análise Comparativa de Degradação de Performance de Escrita e Tamanho de Índice: UUIDv4 (Aleatório) vs. UUIDv7 (Time-ordered) em SGBD Relacional

## 1. Identificação

### 1.1 Título do Experimento
Análise Comparativa de Degradação de Performance de Escrita e Tamanho de Índice: UUIDv4 (Aleatório) vs. UUIDv7 (Time-ordered) em SGBD Relacional

### 1.2 ID
`UUIDv4-vs-UUIDv7-EXP-MED`

### 1.3 Versão do Documento e Histórico de Revisão
- **v1.0 – 20/05/2024:** Criação inicial do planejamento.  
- **v1.1 – 21/05/2024:** Ajuste no título para remover menção específica a microserviços.

### 1.4 Datas
- **Criação:** 20/05/2024  
- **Última atualização:** 21/05/2024

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
Identificadores aleatórios (UUIDv4) geram um padrão de acesso não sequencial em estruturas de índice B-Tree. Diferente de inserções ordenadas que preenchem páginas de memória de maneira sequencial, UUIDs aleatórios exigem carregamento frequente de páginas arbitrárias do disco para a memória e provocam divisões de páginas (page splits).

O objetivo é medir o custo oculto dessa aleatoriedade: o aumento desproporcional de I/O e o desperdício de armazenamento conforme a tabela cresce, comparado ao padrão UUIDv7 ordenado por tempo.

### 2.2 Contexto Organizacional e Técnico
- **Ambiente do Experimento:** Ambiente controlado e isolado para evitar interferência de outras aplicações.  
- **SGBD:** MySQL, escolhido como representativo de bancos relacionais amplamente utilizados. Resultados com aplicabilidade também para SQL Server, PostgreSQL etc.  
- **Cenário de Teste:** Simulação de carga focada em operações de escrita de alta ingestão, comuns em logs, IoT e sistemas financeiros.  
- **Configuração:** Hardware com limitação conhecida de RAM, forçando operações de I/O em disco, onde as diferenças entre UUIDv4 e UUIDv7 se tornam mais evidentes.

### 2.3 Trabalhos e Evidências Prévias
- **Literatura Técnica:** A especificação IETF RFC 9562 argumenta que identificadores ordenados por tempo melhoram a localidade de referência em B-Trees.  
- **Benchmarks de bancos de dados:** Estudos de índices clusterizados mostram que chaves aleatórias causam page splits excessivos, resultando em índices maiores e mais lentos quando comparados com chaves sequenciais (ex.: BIGINT).

### 2.4 Referencial Teórico e Empírico Essencial
- **B-Tree & Page Splitting:** Índices B-Tree crescem eficientemente quando novos valores são adicionados à direita da árvore (valores crescentes). Valores aleatórios exigem inserções no meio de nós já preenchidos, causando splits, aumentando custo de CPU e o tamanho físico do arquivo.  
- **Localidade de Cache:** O desempenho do banco depende fortemente do buffer pool. Acesso aleatório reduz o cache hit ratio, forçando leituras/escritas lentas em disco.  
- **K-Sortable IDs:** Conceito de identificadores aproximadamente ordenáveis como UUIDv7, que combinam a unicidade do UUID com o desempenho de chaves sequenciais.

