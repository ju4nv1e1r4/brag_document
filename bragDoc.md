# Dezembro

## rquitetura de Serving Multi-Modelo (N+1 Worker)

**O Desafio:**
Precisávamos escalar a utilização de IA na empresa, permitindo o uso de múltiplos modelos (Análise de Sentimento, Classificação Zero-Shot, etc.) sem aumentar linearmente os custos de infraestrutura ou a complexidade de deploy.

**A Solução (Arquitetura):**
Desenvolvi um **Worker de ML Unificado** baseado em filas (Redis) que atua como um orquestrador de múltiplos agentes de IA.

**Componentes Chave:**
1.  **Dispatcher (Produtor):** Interface padronizada que serializa inputs e valida tipos antes do enfileiramento.
2.  **Task Queue (Redis):** Fila única (`ml_models:tasks`) que absorve picos de tráfego, garantindo que a aplicação principal nunca trave.
3.  **Model Router (Consumidor):** Um worker Python que utiliza o *Strategy Pattern* para rotear dinamicamente cada mensagem para o modelo específico (Agente) baseado na tag `action`.
4.  **Agentes Isolados:** Módulos independentes para cada modelo, permitindo que a lógica de negócio de um modelo mude sem quebrar os outros.

**Resultado:**
* Capacidade de adicionar novos modelos ("N+1") com esforço marginal (apenas código Python, zero infra).
* Otimização de recursos: Múltiplos modelos compartilham o mesmo runtime e bibliotecas base.
* Resiliência: Implementação de *Fail-Open* e *Retries* centralizados.
