# Dezembro

## *05/12/2025*

### Otimização de Engenharia de Prompt e Arquitetura Neuro-Simbólica para Datas

**Contexto (O Problema):**
O agente de IA sofria com "alucinações aritméticas" ao calcular datas relativas e falhava na interpretação do contexto de calendário.
Inicialmente, o contexto era injetado via um **Grid ASCII** (simulando um calendário visual). Identifiquei que isso gerava ruído na tokenização: o modelo perdia a referência espacial devido à quebra de tokens de espaçamento e quebras de linha, resultando em agendamentos errados.

**A Solução (O que eu fiz):**
Arquitetei uma solução híbrida (Neuro-Simbólica) e refatorei a injeção de contexto:

  * **Otimização de Contexto (Attention-Friendly):** Substituí o Grid ASCII por uma estrutura linear de strings semânticas (ex: `"- Sexta-feira [HOJE]: 09/12"`). Isso eliminou a necessidade do modelo ter raciocínio espacial visual, entregando a informação mastigada para o mecanismo de *Attention*.
  * **Pipeline de Sanitização:** Implementei um *Tool Calling* onde um LLM (`gpt-4o-mini`) atua apenas como normalizador de linguagem natural (convertendo gírias como "fds" ou "próxima sexta"), removendo a responsabilidade lógica dele.
  * **Motor Determinístico:** Desenvolvi o `DateEngine` em Python (com `dateparser` e `Regex`) para realizar o cálculo exato da data baseada na entrada sanitizada, garantindo precisão de 100% em operações de calendário.

**Resultados & Impacto:**

  * **Redução drástica de alucinação:** A mudança do formato ASCII para Lista Semântica eliminou erros de interpretação de dias da semana.
  * **Eficiência de Tokens:** A nova representação textual é mais densa, consumindo menos tokens por requisição (redução de custo e latência).
  * **Experiência do Usuário:** O sistema agora compreende perfeitamente referências relativas complexas ("sábado sem ser este, o outro") e contextos brasileiros.


**Próximo Passo:**
Para deixar isso ainda mais profissional, você pode mencionar se usou alguma biblioteca para validar essa redução de tokens (como o `tiktoken`) ou se foi empírico. Quer que eu escreva um script rápido usando `tiktoken` para você comparar quantos tokens o Grid ASCII gastava vs a sua nova Lista, para adicionar um número exato no "Resultados"?
## *02/12/2025*
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

