**Projeto:** Janus (Plataforma de Monitoramento e Análise de Narrativas Midiáticas)
**Documento Referência:** HLA v1.2 & Proposta de Projeto v1.0
**Data:** 26 de Dezembro de 2025

---

## 1. Estratégia de Seleção Tecnológica

A arquitetura do Janus impõe desafios distintos em diferentes estágios do pipeline de dados. Para atender aos requisitos de **baixa latência** na coleta e **processamento intensivo (CPU-bound)** na análise, adotou-se uma estratégia de persistência e linguagens poliglota.

Os critérios de decisão foram:
1.  **Natureza da Carga:** Distinção clara entre serviços I/O-bound (Coleta/Gateway) e CPU-bound (PLN/Inferência).
2.  **Ecossistema:** Disponibilidade de bibliotecas maduras para a função específica (ex: Hugging Face para Python, Project Reactor para Java).
3.  **Interoperabilidade:** Uso estrito de **Apache Kafka** como espinha dorsal de comunicação assíncrona, desacoplando as stacks tecnológicas.

---

## 2. Detalhamento da Stack por Microserviço

### 2.1. Collector Service (Coleta)
Serviço crítico de I/O, responsável pela descoberta de fontes, gestão de limites de taxa e ingestão de dados brutos.

* **Linguagem:** **Java 21 (LTS)**.
* **Framework/Libs:** **Spring Boot 3** com **Spring WebFlux** (Project Reactor) para concorrência não-bloqueante.
* **Resiliência e Performance:**
    * **Resilience4j:** Implementação de *Circuit Breakers* para lidar com a instabilidade de fontes externas.
    * **Redis (LRU):** Deduplicação de URLs para evitar o reprocessamento de notícias já coletadas.
* **Estratégias de Ingestão de Baixo Custo:**
    * **OSINT:** Manipulação de parâmetros do Google News RSS (`hl=pt-BR`, `gl=BR`, `ceid=BR:pt-419`) para monitoramento focado no Brasil sem chaves de API.
    * **Híbrido:** Integração com NewsData.io e NewsAPI.org com lógica de fallback automático para feeds RSS diretos (G1, Folha, UOL) ao atingir limites de taxa.
* **Extração de Conteúdo:** Orquestração de chamadas para bibliotecas Python especializadas como **Newspaper4k** ou **Crawl4AI** para converter HTML em Markdown estruturado.

### 2.2. Babel Service (Normalização e Contexto) 
Gateway de entrada para o pipeline de processamento, responsável pela identificação de idioma e injeção de contexto (`LocaleContext`).

* **Linguagem:** **Python 3.11+**.
* **Framework:** **FastAPI** (ASGI `Uvicorn`) para suporte assíncrono moderno.
* **Justificativa:** Facilidade de integração com bibliotecas de detecção de idioma (LID) e pré-processamento de texto.

### 2.3. Knowledge Extractor (O "Cérebro" PLN)
O coração da inteligência, puramente CPU/GPU-bound. Realiza NER, Análise de Sentimento e Sumarização.

* **Linguagem:** **Python 3.11+**.
* **Framework:** **Faust** ou consumidores nativos Kafka para processamento de streams.
* **Bibliotecas Core:** `PyTorch`, `Spacy` (NER) e `Hugging Face Transformers` (LLMs para resumo e sentimento).

### 2.4. Narrative Analyzer (Análise e Grafos)
Serviço lógico-matemático que calcula vetores e interage com o banco de grafos.

* **Linguagem:** **Python 3.11+**.
* **Bibliotecas Core:** `NumPy` & `Scikit-learn` (Delta Semântico) e driver oficial para `Neo4j`.

### 2.5. API Gateway (BFF & Agregação)
Ponto único de contato para o Frontend, agregando dados via GraphQL.

* **Linguagem:** **Java 21 (LTS)**.
* **Framework:** **Spring Boot 3** com **Spring WebFlux** e **Spring GraphQL**.

---

## 3. Resumo da Stack Tecnológica

| Serviço | Natureza | Tecnologia Selecionada | Justificativa Concisa |
| :--- | :--- | :--- | :--- |
| **Collector Service** | I/O Intensivo | **Java 21 (WebFlux)** | Uniformidade com Gateway e performance reativa superior para I/O massivo. |
| **Babel Service** | Híbrido | **Python (FastAPI)** | Suporte assíncrono e integração nativa com bibliotecas de NLP. |
| **Knowledge Extractor** | CPU/GPU Intensivo | **Python (PyTorch/Spacy)** | Hegemonia no ecossistema de Deep Learning e Transformers. |
| **Narrative Analyzer** | Lógica/Matemática | **Python (NumPy/Scikit)** | Eficiência em cálculos vetoriais e drivers flexíveis para Grafos. |
| **API Gateway** | Agregação/Vazão | **Java 21 (Spring WebFlux)** | Modelo não-bloqueante ideal para orquestração de I/O em GraphQL. |

---

## 4. Infraestrutura de Suporte (Cross-Cutting)

Para suportar a stack acima, a infraestrutura base (Containerização e Mensageria) segue padronizada:

* **Message Broker:** **Apache Kafka** (Garante o desacoplamento entre o Collector e os Workers em Python).
* **Containerização:** **Docker** (Imagens *distroless* para Java e imagens otimizadas com suporte a GPU (CUDA) para os serviços Python).
* **Observabilidade:** **OpenTelemetry** (Instrumentação automática compatível tanto com Java Agent quanto com bibliotecas Python).
