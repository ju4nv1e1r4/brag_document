# Dezembro
## *08/12/2025*
#### Arquitetura de Inferência de Alta Performance (Worker N+1)

**Contexto:**
Necessidade de servir múltiplos modelos de NLP (NER, Zero-Shot) em produção com baixa latência e custo reduzido, sem depender de GPUs caras.
A arquitetura anterior (ou a falta de uma) dificultava a gestão de versões e o scaling horizontal.

**A Solução Técnica:**
Desenvolvi um **Worker de Inferência Unificado** altamente otimizado para CPU.
* **Engine:** Utilização de **ONNX Runtime + Tokenizers**, garantindo inferências na ordem de milissegundos em hardware padrão.
* **Initialization Strategy:** Implementei carregamento de modelos no *startup* da aplicação de forma **assíncrona**. Isso impede o bloqueio do Event Loop durante a inicialização dos workers e elimina o "Cold Start" para o usuário final.
* **Resource Management:** Sistema de cache local em **Volume Docker**. O download do GCS ocorre apenas na atualização, reduzindo tráfego de rede e tempo de boot.
* **Model Lifecycle (CLI):** Criei uma ferramenta CLI em Python para gerenciar o ciclo de vida dos modelos (download, upload para GCS e atualização de referências), preparando o terreno para um futuro pipeline de GitOps.

**Impacto & Resultados:**
* **Eficiência:** Capacidade de rodar múltiplos modelos (N+1) em um único contêiner com consumo de memória compartilhado.
* **UX:** Latência de inferência imperceptível para o usuário final devido ao pré-carregamento e quantização.
* **Developer Experience:** Processo de atualização de modelos desacoplado do código da aplicação via CLI.

**Stack:** Python, Redis, ONNX Runtime, Docker Volumes, Google Cloud Storage.
