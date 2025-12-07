**Projeto:** Janus (Plataforma de Monitoramento e Análise de Narrativas Midiáticas)
**Documento Referência:** HLA v1.2 & Proposta de Projeto v1.0
**Data:** 07 de Dezembro de 2025

---
## 1. Estratégia de Seleção Tecnológica

A arquitetura do Janus impõe desafios distintos em diferentes estágios do pipeline de dados. Para atender aos requisitos de **baixa latência** na coleta e **processamento intensivo (CPU-bound)** na análise, adotou-se uma estratégia de persistência e linguagens poliglota.

Os critérios de decisão foram:
1.  **Natureza da Carga:** Distinção clara entre serviços I/O-bound (Coleta/Gateway) e CPU-bound (PLN/Inferência).
2.  **Ecossistema:** Disponibilidade de bibliotecas maduras para a função específica (ex: Hugging Face para Python, Project Reactor para Java).
3.  **Interoperabilidade:** Uso estrito de **Apache Kafka** como espinha dorsal de comunicação assíncrona, desacoplando as stacks tecnológicas.

---

## 2. Detalhamento da Stack por Microserviço

Abaixo apresenta-se a definição técnica para cada um dos cinco serviços core definidos na HLA.

### 2.1. Collector Service (Coleta)
Serviço crítico de I/O, responsável por manter milhares de conexões HTTP simultâneas e gerenciar *backpressure*.

* **Linguagem:** **Go (Golang)**
* **Framework/Libs:** `Colly` (Scraping), `Sarama` (Kafka Client).
* **Justificativa Arquitetural:**
    * **Modelo de Concorrência (Goroutines):** O Go permite a execução de milhares de "crawlers" leves simultaneamente com um footprint de memória minúsculo comparado a Threads de SO tradicionais. Ideal para o *Adaptive Polling* e alta vazão de requisições de rede.
    * **Performance Bruta:** Compilação nativa e gerenciamento eficiente de garbage collection para ciclos de vida curtos de objetos (requests HTTP).

### 2.2. Babel Service (Normalização e Tradução)
Gateway de entrada para o pipeline de processamento, atuando como orquestrador de chamadas de tradução.

* **Linguagem:** **Python 3.11+**
* **Framework:** **FastAPI** (com servidor ASGI `Uvicorn`).
* **Justificativa Arquitetural:**
    * Embora envolva I/O (chamadas a APIs de tradução ou modelos locais), a necessidade de pré-processamento de texto e detecção de idioma (LID) com bibliotecas como `fasttext` ou `langdetect` torna o Python a escolha natural para manter a coesão com o restante do núcleo de IA. O FastAPI garante a performance assíncrona necessária para não bloquear enquanto aguarda a inferência da tradução.

### 2.3. Knowledge Extractor (O "Cérebro" PLN)
O coração da inteligência, puramente CPU/GPU-bound. Realiza NER, Análise de Sentimento e Sumarização.

* **Linguagem:** **Python 3.11+**
* **Framework:** **Faust** (Stream Processing) ou Consumidores Nativos Kafka.
* **Bibliotecas Core:** `PyTorch`, `Spacy` (NER), `Hugging Face Transformers` (LLMs para resumo e sentimento).
* **Justificativa Arquitetural:**
    * **Hegemonia de Ecossistema:** O Python é a *lingua franca* da Inteligência Artificial. Utilizar qualquer outra linguagem exigiria "pontes" ou reimplementações ineficientes de modelos SOTA (State-of-the-Art).
    * **Integração:** Facilidade de carregar modelos pré-treinados e realizar fine-tuning específico para o domínio geopolítico.

### 2.4. Narrative Analyzer (Análise e Grafos)
Serviço lógico-matemático que calcula vetores e interage com estruturas de grafos complexas.

* **Linguagem:** **Python 3.11+**
* **Framework:** Workers Python puros.
* **Bibliotecas Core:** `NumPy` & `Scikit-learn` (Cálculo de Delta Semântico/Cosseno), `Neo4j Python Driver`.
* **Justificativa Arquitetural:**
    * O cálculo do "Delta Semântico" exige manipulação vetorial densa, onde o Python (via C-extensions do NumPy) é extremamente performático. Além disso, a flexibilidade para modelar as queries Cypher dinâmicas para o Neo4j é superior no ambiente Python.

### 2.5. API Gateway (BFF & Agregação)
Ponto único de contato para o Frontend. Precisa agregar dados de múltiplas fontes (Elastic, Neo4j, PostGIS) e entregá-los via GraphQL.

* **Linguagem:** **Java 21 (LTS)**
* **Framework:** **Spring Boot 3** com **Spring WebFlux** (Project Reactor) e **Spring GraphQL**.
* **Justificativa Arquitetural:**
    * **I/O Não-Bloqueante:** O modelo reativo do WebFlux é ideal para um Gateway que passa a maior parte do tempo aguardando respostas de bancos de dados ou caches (Redis). Ele maximiza o throughput com um número fixo de threads.
    * **Robustez do GraphQL:** O ecossistema Java (Spring for GraphQL) oferece tipagem estrita, tratamento de erros robusto e instrumentação de observabilidade superior para APIs públicas complexas.

---

## 3. Resumo da Stack Tecnológica

| Serviço | Natureza | Tecnologia Selecionada | Justificativa Concisa |
| :--- | :--- | :--- | :--- |
| **Collector Service** | I/O Intensivo (Network) | **Go (Golang)** | Concorrência massiva via goroutines para scraping de alta escala com baixo consumo de memória. |
| **Babel Service** | Híbrido (I/O + Lógica) | **Python (FastAPI)** | Integração nativa com libs de NLP para detecção de idioma e suporte assíncrono moderno. |
| **Knowledge Extractor** | CPU/GPU Intensivo | **Python (PyTorch/Spacy)** | Acesso obrigatório ao ecossistema de Deep Learning e Transformers (Hugging Face). |
| **Narrative Analyzer** | Lógica/Matemática | **Python (NumPy/Scikit)** | Eficiência em cálculos vetoriais (Delta Semântico) e drivers flexíveis para Grafos. |
| **API Gateway** | Agregação/Alta Vazão | **Java 21 (Spring WebFlux)** | Modelo reativo não-bloqueante ideal para orquestração de I/O e robustez na implementação de GraphQL. |

---

## 4. Infraestrutura de Suporte (Cross-Cutting)

Para suportar a stack acima, a infraestrutura base (Containerização e Mensageria) segue padronizada:

* **Message Broker:** **Apache Kafka** (Garante o desacoplamento entre o Collector em Go e os Workers em Python).
* **Containerização:** **Docker** (Imagens *distroless* para Go/Java e imagens otimizadas com suporte a GPU (CUDA) para os serviços Python).
* **Observabilidade:** **OpenTelemetry** (Instrumentação automática compatível tanto com Java Agent quanto com bibliotecas Python/Go).
