# Janus

**Plataforma de Inteligência de Narrativas Midiáticas Globais**

O Janus é um sistema distribuído orientado a eventos projetado para desconstruir a guerra da informação em tempo real. A plataforma agrega fluxos globais de notícias, utiliza Processamento de Linguagem Natural (PLN) para quantificar viés e dissonância semântica entre nações conflitantes, e visualiza essas narrativas através de mapas de calor geoespaciais e grafos de conhecimento.

**Destaques da Arquitetura:**
* **Padrão:** Kappa Architecture (Log-centric, event sourcing).
* **Core:** Kafka, Zookeeper, Python (Transformers/Spacy).
* **Persistência Poliglota:** Neo4j (Grafos), MongoDB (Docs), PostGIS (Geo/Tempo).
* **Funcionalidades Chave:** Detecção automatizada de viés, extração de entidades multilíngue e comparação de fontes (Split-View).

---
📚 **Documentação:**
- [Proposta de Projeto](docs/project-proposal.md)
- [Arquitetura de Alto Nível (HLA)](docs/high-level-architecture.md)
