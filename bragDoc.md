# AVISO IMPORTANTE

Este documento trata de informações verídicas mas o texto foi revisado e corrigido por um LLM, basicamente pra melhorar a escrita. 

**Dica:** Use IA pra te ajudar a ir mais rapido onde você sabe que pode ir.

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

## *10/12/2025*

## Otimização de Contexto em LLM com Injeção Dinâmica via Redis

**Contexto (O Problema):**
O módulo de agendamento sofria com instabilidade nas respostas da LLM (alucinações) devido à poluição do contexto. Ao retornar listas extensas de horários diretamente no histórico da conversa, excedíamos a janela de atenção útil do modelo, fazendo com que ele se confundisse entre modalidades ou inventasse horários. Além disso, o fluxo se perdia quando dados intermediários (como CPF ou data de nascimento) eram solicitados, exigindo que o usuário reiniciasse a intenção.

**A Solução (O que eu fiz):**
Arquitetou-se um padrão de **Dynamic Context Injection** (Injeção Dinâmica de Contexto) para desacoplar a recuperação de dados da apresentação para a IA:

* **Gerenciamento de Estado com Redis:** Implementação de uma estratégia de cache com TTL (10 min) para armazenar estados transientes da agenda (`ctx:agenda:today`, `ctx:agenda:instruction`).
* **Higiene de Contexto:** Criação de lógica para limpar chaves conflitantes (ex: apagar horários quando o foco muda para escolha de modalidades), garantindo que a LLM tenha acesso apenas à "verdade" do momento atual.
* **Pipeline de Injeção no System Prompt:** Desenvolvimento do método `_get_dynamic_agenda_context` que intercepta a construção do prompt e injeta os dados "quentes" do Redis diretamente nas instruções do sistema, mantendo o histórico de conversa do usuário limpo.
* **Resiliência de Fluxo (State Recovery):** Implementação de verificação de "fluxo pendente" (ex: `was_booking_flow`). Se o sistema precisa interromper o agendamento para pedir um dado (ex: Data de Nascimento), ele agora "lembra" da intenção original e retoma a busca de aulas automaticamente após a inserção do dado.

**Resultados & Impacto:**
* **Redução de Alucinações:** A segregação entre instruções de fluxo e dados brutos aumentou significativamente a precisão do modelo ao sugerir horários.
* **Otimização de Tokens:** Envio sob demanda de informações críticas apenas no System Prompt, economizando tokens de input no histórico longo.
* **UX Mais Fluida:** O usuário não precisa mais repetir o comando "quero agendar" após fornecer um dado cadastral faltante; o sistema reconecta o fluxo automaticamente.
* **Melhoria de Observabilidade:** Logs estruturados indicando quando o contexto dinâmico foi injetado ou quando falhou (tratamento de erro robusto no acesso ao Redis).

## *11/12/2025*

## Sistema Autônomo de Recuperação de 'No-Show' e Validação de Presença com LLM

**Contexto (O Problema):**
O processo de validação de presença em aulas experimentais carecia de automação, dificultando a retenção de alunos que faltavam (no-show) e a coleta de feedback daqueles que compareciam. Havia um *gap* de comunicação pós-aula que resultava em dados inconsistentes no CRM e perda de oportunidades de reagendamento.

**A Solução (O que eu fiz):**
* **Arquitetura Assíncrona com Redis:** Projetei e implementei um serviço em **Python** que utiliza filas do **Redis** para agendar *jobs* de verificação com delay, gatilhados automaticamente 1 hora após o término da aula.
* **Agente LLM + Sistema:** O sistema injeta o contexto do aluno (status da aula) no prompt, permitindo que o modelo tome decisões dinâmicas baseadas no estado atual.
* **Lógica de Negócio e Atualização de Estado:** Implementei um fluxo de decisão robusto:
    * *Cenário 1 (Presença Confirmada):* O sistema valida o dado silenciosamente, economizando tokens e evitando spam.
    * *Cenário 2 (Incerteza):* O LLM inicia uma busca ativa. Se o aluno confirma presença, o sistema atualiza o registro via API e solicita feedback (análise de sentimento). Se o aluno confirma falta, o agente atua na retenção oferecendo novos horários (reagendamento).

**Resultados & Impacto:**
* **Automação do Ciclo de Vendas:** Eliminou a necessidade de follow-up manual por parte da equipe de vendas/atendimento para confirmação de aulas experimentais.
* **Enriquecimento de Dados:** Garantiu que a base de dados reflita a realidade (presença/ausência) através da validação direta com o usuário final via chat.
* **Experiência do Usuário:** Criou um ponto de contato imediato e personalizado, aumentando as chances de remarcação (recuperação de lead) em casos de imprevistos.

# Agosto

## 11/08/2025

## Otimização de Latência em Sistema LLM via Migração Arquitetural (BigQuery → Postgres)**Contexto (O Problema):**
O sistema de gerenciamento de conversas (Human-LLM) utilizava o BigQuery como backend de persistência. Por ser um banco orientado a análises (OLAP), o BigQuery introduzia uma latência elevada e inadequada para as operações de leitura/escrita em tempo real necessárias durante o fluxo de chat, degradando a experiência do usuário (UX) e aumentando o tempo de resposta total da IA.

**A Solução (O que eu fiz):**

* **Refatoração do Módulo de Dados:** Reescrevi a camada de acesso a dados (Data Access Layer), desacoplando a lógica de negócio da infraestrutura e implementando repositórios otimizados para PostgreSQL.
* **Automação de Migração (CLI):** Desenvolvi um utilitário robusto em **Bash Script** para orquestrar a migração dos dados.
* **Estratégias de Ingestão Flexíveis:** Implementei lógica de argumentos (`flags`) no script para suportar diferentes cenários de deploy:
* `--strategy truncate-and-load`: Para cargas iniciais ou ambientes de staging (idempotência).
* `--strategy insert`: Para ingestão incremental sem downtime.
* `--limit`: Para testes controlados e validação de integridade em subconjuntos de dados.


* **Infraestrutura:** Configurei o ambiente para suportar a transição de um paradigma OLAP para OLTP.

**Resultados & Impacto:**

* **Redução Crítica de Latência:** A troca para o PostgreSQL reduziu drasticamente o tempo de *retrieval* do histórico de conversas, acelerando a geração de respostas do LLM.
* **Robustez Operacional:** A ferramenta de migração automatizada eliminou erros manuais e padronizou o processo de deploy de dados entre ambientes.
* **Melhoria de Arquitetura:** O sistema agora segue boas práticas de separação entre banco transacional (PostgreSQL para o app) e analítico (BigQuery mantido apenas para logs/analytics, se aplicável).

  # Maio

  ## 14/05/2025

## Implementação de Observabilidade Distribuída com OpenTelemetry em Stack de GenAI**Contexto (O Problema):**
O ecossistema de microsserviços de IA Generativa dependia de logs lineares tradicionais (`logging` padrão do Python). Isso gerava silos de informação, dificultando o rastreamento (tracing) de requisições complexas entre serviços, a identificação de gargalos de latência na inferência e o *debugging* de falhas em produção.

**A Solução (O que eu fiz):**

* **Migração para OpenTelemetry:** Substituí a estratégia de logging legada pela implementação do padrão de **Distributed Tracing** utilizando OpenTelemetry, garantindo aderência aos padrões da CNCF.
* **Abstração via Decorator Pattern:** Desenvolvi uma classe utilitária (Wrapper) que expõe um *Python Decorator* customizado. Isso permite instrumentar funções e métodos automaticamente, capturando contextos, metadados e spans sem poluir a lógica de negócio ("Zero-touch instrumentation" para o time).
* **Infraestrutura de Visualização:** Configurei e integrei o **Jaeger** via Docker na stack de microsserviços, habilitando uma interface gráfica para visualização da árvore de chamadas (Traces) e dependências.

**Resultados & Impacto:**

* **Visibilidade End-to-End:** Transformou o monitoramento de "caixa preta" para uma visão granular do ciclo de vida da requisição, essencial para pipelines de LLMs.
* **Redução do MTTR (Mean Time to Recovery):** A visualização gráfica dos traces no Jaeger reduziu drasticamente o tempo necessário para identificar a causa raiz de erros e timeouts.
* **Padronização de Código:** O uso do decorador garantiu que novos serviços nasçam observáveis por padrão, reduzindo o *boilerplate* de configuração para o restante do time.
