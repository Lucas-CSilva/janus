# Definição de Arquitetura de Alto Nível (HLA)

**Projeto:** Janus (Plataforma de Monitoramento e Análise de Narrativas Midiáticas)
**Versão da Arquitetura:** 1.2 (Nomenclatura Refinada)
**Responsável:** Arquiteto de Software Sênior
**Data:** 06 de Dezembro de 2025

---

## 1. Visão Geral e Estilo Arquitetural

A solução mantém a arquitetura baseada em **Microserviços Orientados a Eventos (Event-Driven Architecture)**, implementando rigorosamente o padrão **Kappa Architecture**. Todos os dados são tratados como um fluxo contínuo de eventos imutáveis, permitindo processamento em tempo real e reprocessamento histórico (*replay*) sem alteração de código, essencial para a natureza evolutiva da análise geopolítica.

A comunicação entre serviços é estritamente assíncrona para operações de escrita/processamento (via Message Broker) e síncrona (via GraphQL) apenas para leitura na camada de borda.

---

## 2. Catálogo de Serviços e Responsabilidades

A arquitetura do Janus é composta por cinco serviços principais, desacoplados e especializados, organizados para maximizar a coesão e minimizar o acoplamento.

### 2.1. Serviço de Coleta ("Collector Service")
Responsável pela fronteira de entrada de dados, garantindo resiliência contra falhas externas e alta qualidade do dado bruto.

* **Sub-componente: Gestor de Fontes e Resiliência**
    * **Circuit Breaker:** Implementação de padrão de proteção para fontes instáveis ou com *rate-limits* agressivos. Se uma fonte falhar repetidamente, o circuito abre para evitar desperdício de recursos.
    * **Estratégia de Adaptive Polling:** Fontes com alta frequência de atualização (ex: Live Blogs) são visitadas mais vezes que fontes estáticas.
* **Sub-componente: Classificador de Relevância (Noise Filter) **
    * **Função:** Filtrar conteúdo "Não Relacionado a Conflito" com base em vocabulário e contexto semântico. Apenas artigos com *score* de relevância > 0.75 avançam no pipeline, garantindo pureza dos dados.
* **Saída:** Evento `ArtigoBrutoColetado` (payload original + metadados).

### 2.2. Serviço de Normalização e Tradução ("Babel Service")
Responsável pela gestão multilíngue e padronização, garantindo que o restante do pipeline opere de forma agnóstica ao idioma original.

* **Responsabilidades:**
    * **Identificação de Idioma (LID):** Detecção automática do idioma da fonte.
    * **Tradução Neural:** Tradução automática para o inglês (idioma pivô da análise) utilizando modelos de tradução de máquina, preservando sempre o texto original para auditoria e *compliance*.
* **Saída:** Evento `ArtigoNormalizado` (Texto original + Texto traduzido).

### 2.3. Extrator de Conhecimento ("Knowledge Extractor")
Conjunto de *workers* escaláveis horizontalmente (Padrão *Competing Consumers*).

* **Processador de Entidades e Geoespacial:** Executa NER (*Named Entity Recognition*) multilíngue para extração de atores políticos e resolve toponímia para coordenadas padronizadas.
* **Analisador de Viés e Sentimento:**
    * Calcula polaridade (-1 a +1) e subjetividade (0 a 1).
    * Identifica "adjetivação carregada" (framing) próxima às entidades chave.
* **Gerador de Resumos:** Sumarização abstrativa focada nos parágrafos onde as entidades detectadas aparecem, utilizando LLMs otimizados para resumo.

### 2.4. Analisador de Narrativas ("Narrative Analyzer")
Serviço especializado em análise comparativa e arbitragem de fontes.

* **Responsabilidades:**
    * Consome clusters de notícias sobre o mesmo "Evento Canônico".
    * **Cálculo de Delta Semântico:** Algoritmo que compara vetores de sentimento e escolha léxica entre duas fontes distintas (ex: *BBC* vs *Russia Today*).
    * **Detecção de Dissonância:** Gera um alerta se a divergência de narrativa ultrapassar um *threshold* configurado.
* **Saída:** Objeto `RelatorioComparativo` persistido no banco de grafos (arestas de divergência).

### 2.5. Gateway de API e Visualização ("API Gateway")
Camada BFF (*Backend for Frontend*) utilizando **GraphQL**.

* **Responsabilidades:**
    * Exposição de esquema unificado e hierárquico para o Frontend.
    * **Caching Inteligente:** Uso de Redis para cachear resultados de *queries* pesadas (ex: "Top Conflitos da Semana") e CDN para ativos estáticos.
    * Transformação de dados para formatos de visualização (GeoJSON para mapas, JSON Graph para redes).

---
## 3. Fluxo de Execução e Pipeline de Dados

O fluxo foi refinado para eliminar falsos positivos e garantir a normalização linguística antes da inferência.

1.  **Ingestão e Deduplicação (Collector):** O Collector busca a URL. Verifica ineditismo consultando um **Cache Distribuído (Redis)** com política LRU. Se inédito, baixa o conteúdo.
2.  **Classificação de Ruído (Collector):** O *Classificador de Relevância* avalia o texto. Se aprovado, publica o evento.
3.  **Tradução (Babel):** O *Babel Service* traduz o conteúdo para o inglês, se necessário, e normaliza datas/formatos.
4.  **Enriquecimento (Knowledge Extractor):** Processamento paralelo de NER, Sentimento e Resumo sobre o texto normalizado.
5.  **Clusterização e Arbitragem (Narrative Analyzer):** O artigo é associado a um evento existente. O *Narrative Analyzer* calcula as diferenças narrativas contra outras fontes do cluster.
6.  **Persistência Poliglota:** Os dados enriquecidos são roteados para os armazenamentos especializados (Sink Connectors).
7.  **Consumo (API Gateway):** O Frontend solicita dados via GraphQL; o *API Gateway* agrega as fontes de dados em uma resposta única.

---

## 4. Estratégia de Persistência (Poliglota)

Mantém-se a estratégia poliglota, refinada para maior eficiência operacional e analítica.

### 4.1. Armazenamento Documental (MongoDB / Elasticsearch)
* **Dados:** Texto integral (original e traduzido), metadados ricos, logs de processamento.
* **Justificativa:** Fonte da verdade para leitura detalhada e *Full-Text Search*. Flexibilidade para esquemas variáveis de notícias.

### 4.2. Armazenamento em Grafos (Neo4j)
* **Dados:** Nós (Atores, Fontes, Eventos) e Relações (Cita, Ataca, Apoia, Contradiz).
* **Justificativa:** Essencial para visualizar redes de influência e responder perguntas como "Quem financia quem?". As arestas de "Contradiz" armazenam o *score* do Delta Semântico.

### 4.3. Séries Temporais e Espacial (PostGIS + TimescaleDB)
* **Dados:** Geometrias de conflito e métricas de sentimento timestamped.
* **Justificativa:** Backend performático para o Mapa de Calor (GIS) e Timeline, permitindo *queries* espaciais complexas impossíveis em bancos NoSQL comuns.

### 4.4. Cache Distribuído (Redis)
* **Uso:** Deduplicação de URLs (chaves com TTL longo) e cache de respostas de API.
* **Justificativa:** Baixa latência e alta taxa de transferência.

---

## 5. Lógica Algorítmica e Visualização

Detalhamento das lógicas de negócio críticas para a proposta de valor.

### **5.1. Algoritmo de Mapa de Calor (Geo-Heatmap Logic)**
O mapa utiliza uma grade hexagonal (H3 Grid) para agregação espacial.

* **Fórmula do Índice de Calor ($GHI_{hex}$):**
    $$GHI_{hex} = \left( \alpha \cdot \sum N_{docs} \right) + \left( \beta \cdot \sum |S_{avg}| \right) \cdot D(t)$$
    * $N_{docs}$: Volume de notícias na célula hexagonal.
    * $|S_{avg}|$: Intensidade absoluta do sentimento (notícias extremas geram mais calor que neutras).
    * $D(t)$: Fator de decaimento temporal (eventos recentes têm peso maior).
    * $\alpha, \beta$: Pesos configuráveis via painel administrativo.

### **5.2. Comparador de Fontes (Split-View Logic)**
O backend entrega um objeto `DiffAnalysis` contendo:
* **Entidades Comuns:** Interseção de atores citados.
* **Sentimento Divergente:** Mapa de calor textual destacando onde a Fonte A é positiva e a Fonte B é negativa sobre o mesmo sujeito.
* **Vocabulário Único:** Termos exclusivos (ex: Fonte A usa "Rebeldes", Fonte B usa "Terroristas").

---

## 6. Requisitos Não-Funcionais e Operacionais

Seção adicionada para garantir a robustez e manutenibilidade do sistema em produção.

### **6.1. Observabilidade e Monitoramento**
Adoção do padrão **OpenTelemetry**.
* **Distributed Tracing:** Rastreamento de `CorrelationID` ponta a ponta (Collector -> Babel -> Knowledge Extractor -> Narrative Analyzer).
* **Métricas Chave (SLIs):**
    * *Collector Lag:* Tempo entre publicação e coleta.
    * *Extractor Confidence:* Alerta se a confiança média do NER cair abaixo de 80%.
    * *Translation Latency:* Monitoramento de gargalo no Babel Service.

### **6.2. Estratégia de Disaster Recovery (DR)**
* **Event Sourcing (Kafka):** Retenção de tópicos de dados brutos configurada para 7 a 30 dias. Permite o **Replay** integral dos eventos para reprocessamento em caso de correção de lógica no *Knowledge Extractor* ou *Narrative Analyzer*.
* **Snapshots:** Backups diários dos bancos de dados com Point-In-Time Recovery (PITR).

### **6.3. Gestão de Escalabilidade**
* **Auto-scaling:** Provisionamento dinâmico de *workers* do Brain Core baseado na métrica de *Consumer Lag* do Kafka.

---

## 7. Conclusão

A arquitetura v1.2 do **Projeto Janus** consolida a maturidade técnica da plataforma. Com a definição clara dos serviços (`collector`, `babel`, `knowledge-extractor`, `narrative-analyzer`, `api-gateway`) e a adoção de padrões robustos de observabilidade, o sistema está apto a suportar a complexidade da análise de narrativas geopolíticas globais com confiabilidade e precisão.
