IMPORTANT NOTICE
This document contains factual information, but the text has been reviewed and corrected by an LLM, basically to improve the writing.

Tip: Use AI to help you go faster where you know you can go.

December
05/12/2025
Prompt Engineering Optimization and Neuro-Symbolic Architecture for Dates
Context (The Problem): The AI ​​agent suffered from "arithmetic hallucinations" when calculating relative dates and failed to interpret the calendar context. Initially, the context was injected via an ASCII Grid (simulating a visual calendar). I identified that this generated noise in tokenization: the model lost spatial reference due to token spacing and line breaks, resulting in incorrect scheduling.

The Solution (What I did): I designed a hybrid solution (Neuro-Symbolic) and refactored the context injection:

Context Optimization (Attention-Friendly): I replaced the ASCII Grid with a linear structure of semantic strings (e.g., "- Friday [TODAY]: 09/12"). This eliminated the need for the model to have visual spatial reasoning, delivering the information pre-digested to the Attention mechanism.

Sanitation Pipeline: I implemented a Tool Calling where an LLM (gpt-4o-mini) acts only as a natural language normalizer (converting slang like "fds" or "next Friday"), removing its logical responsibility.

Deterministic Engine: I developed the DateEngine in Python (with dateparser and Regex) to perform the exact date calculation based on the sanitized input, guaranteeing 100% accuracy in calendar operations.

Results & Impact:

Drastic reduction in hallucinations: The change from ASCII format to Semantic List eliminated errors in interpreting days of the week.
Token Efficiency: The new textual representation is denser, consuming fewer tokens per request (reducing cost and latency).
User Experience: The system now perfectly understands complex relative references ("Saturday other than this one, the other one") and Brazilian contexts.

02/12/2025
High-Performance Inference Architecture (N+1 Worker)
Context: Need to serve multiple NLP models (NER, Zero-Shot) in production with low latency and reduced cost, without relying on expensive GPUs. The previous architecture (or lack thereof) made version management and horizontal scaling difficult.

The Technical Solution: I developed a highly CPU-optimized Unified Inference Worker.

Engine: Use of ONNX Runtime + Tokenizers, guaranteeing millisecond inferences on standard hardware.
Initialization Strategy: I implemented asynchronous model loading at application startup. This prevents Event Loop blocking during worker initialization and eliminates "Cold Start" for the end user.
Resource Management: Local caching system in Docker Volume. GCS download only occurs during updates, reducing network traffic and boot time.
Model Lifecycle (CLI): I created a Python CLI tool to manage the model lifecycle (download, upload to GCS, and reference updates), paving the way for a future GitOps pipeline.
Impact & Results:

Efficiency: Ability to run multiple models (N+1) in a single container with shared memory consumption.
UX: Imperceptible inference latency for the end user due to preloading and quantization.
Developer Experience: Model update process decoupled from application code via CLI.
Stack: Python, Redis, ONNX Runtime, Docker Volumes, Google Cloud Storage.

10/12/2025
Context Optimization in LLM with Dynamic Injection via Redis
Context (The Problem): The scheduling module suffered from instability in LLM responses (hallucinations) due to context pollution. By returning extensive lists of times directly in the conversation history, we exceeded the model's useful attention span, causing it to get confused between modalities or invent times. Furthermore, the flow was lost when intermediate data (such as CPF or date of birth) was requested, requiring the user to restart the intent.

The Solution (What I did): A Dynamic Context Injection pattern was architected to decouple data retrieval from the presentation to the AI:

State Management with Redis: Implementation of a caching strategy with TTL (10 min) to store transient agenda states (ctx:agenda:today, ctx:agenda:instruction).

Context Hygiene: Creation of logic to clean up conflicting keys (e.g., deleting schedules when the focus shifts to choosing modalities), ensuring that the LLM only has access to the "truth" of the current moment.
Injection Pipeline in the System Prompt: Development of the _get_dynamic_agenda_context method that intercepts the prompt construction and injects "hot" data from Redis directly into the system instructions, keeping the user's conversation history clean.
Flow Resilience (State Recovery): Implementation of "pending flow" checking (e.g., was_booking_flow). If the system needs to interrupt scheduling to request data (e.g., Date of Birth), it now "remembers" the original intent and automatically resumes the class search after the data is entered.

Results & Impact:

Reduction of Hallucinations: The segregation between flow instructions and raw data significantly increased the model's accuracy in suggesting schedules.

Token Optimization: On-demand sending of critical information only in the System Prompt, saving input tokens in the long history.
More Fluid UX: The user no longer needs to repeat the "I want to schedule" command after providing missing registration data; the system automatically reconnects the flow.
Improved Observability: Structured logs indicating when the dynamic context was injected or when it failed (robust error handling in Redis access).

11/12/2025
Autonomous System for 'No-Show' Recovery and Attendance Validation with LLM
Context (The Problem): The attendance validation process in experimental classes lacked automation, making it difficult to retain students who were absent (no-show) and to collect feedback from those who attended. There was a post-class communication gap that resulted in inconsistent data in the CRM and lost rescheduling opportunities.

The Solution (What I did):

Asynchronous Architecture with Redis: I designed and implemented a service in Python that uses Redis queues to schedule delayed verification jobs, triggered automatically 1 hour after the end of the class.

LLM Agent + System: The system injects the student's context (class status) into the prompt, allowing the model to make dynamic decisions based on the current state.

Business Logic and State Update: I implemented a robust decision flow:
Scenario 1 (Attendance Confirmed): The system validates the data silently, saving tokens and avoiding spam.

Scenario 2 (Uncertainty): The LLM initiates an active search. If the student confirms attendance, the system updates the record via API and requests feedback (sentiment analysis). If the student confirms absence, the agent acts on retention by offering new times (rescheduling).

Results & Impact:

Sales Cycle Automation: Eliminated the need for manual follow-up by the sales/customer service team to confirm trial classes.

Data Enrichment: Ensured that the database reflects reality (presence/absence) through direct validation with the end user via chat.

User Experience: Created an immediate and personalized point of contact, increasing the chances of rescheduling (lead recovery) in case of unforeseen circumstances.

August
11/08/2025
Latency Optimization in an LLM System via Architectural Migration (BigQuery → Postgres) Context (The Problem):
The conversation management system (Human-LLM) used BigQuery as a persistence backend. Being an analytics-oriented database (OLAP), BigQuery introduced high and inadequate latency for the real-time read/write operations required during the chat flow, degrading the user experience (UX) and increasing the total AI response time.

The Solution (What I did):

Data Module Refactoring: I rewrote the data access layer, decoupling the business logic from the infrastructure and implementing repositories optimized for PostgreSQL.

Migration Automation (CLI): I developed a robust utility in Bash Script to orchestrate the data migration.

Flexible Ingestion Strategies: I implemented argument logic (flags) in the script to support different deployment scenarios:

--strategy truncate-and-load: For initial loads or staging environments (idempotence).

--strategy insert: For incremental ingestion without downtime.

--limit: For controlled testing and integrity validation on subsets of data.

Infrastructure: I configured the environment to support the transition from an OLAP to OLTP paradigm.

Results & Impact:

Critical Latency Reduction: Switching to PostgreSQL drastically reduced the retrieval time of the conversation history, accelerating the generation of LLM responses.

Operational Robustness: The automated migration tool eliminated manual errors and standardized the data deployment process between environments.

Architecture Improvement: The system now follows best practices for separating the transactional database (PostgreSQL for the app) and the analytical database (BigQuery maintained only for logs/analytics, if applicable).

May
05/14/2025
Implementation of Distributed Observability with OpenTelemetry in a Generative AI Stack (The Problem):
The generative AI microservices ecosystem relied on traditional linear logs (standard Python logging). This created information silos, hindering the tracing of complex requests between services, the identification of latency bottlenecks in inference, and the debugging of production failures.

The Solution (What I did):

Migration to OpenTelemetry: I replaced the legacy logging strategy with the implementation of the Distributed Tracing pattern using OpenTelemetry, ensuring adherence to CNCF standards.

Abstraction via Decorator Pattern: I developed a utility class (Wrapper) that exposes a custom Python Decorator. This allows for the automatic instrumentation of functions and methods, capturing contexts, metadata, and spans without polluting the business logic ("Zero-touch instrumentation" for the team).

Visualization Infrastructure: I configured and integrated Jaeger via Docker into the microservices stack, enabling a graphical interface for visualizing the call tree (Traces) and dependencies.

Results & Impact:

End-to-End Visibility: Transformed "black box" monitoring into a granular view of the request lifecycle, essential for LLM pipelines.

Reduced MTTR (Mean Time to Recovery): The graphical visualization of traces in Jaeger drastically reduced the time needed to identify the root cause of errors and timeouts.

Code Standardization: The use of the decorator ensured that new services are observable by default, reducing the configuration boilerplate for the rest of the team.
