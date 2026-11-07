# AI Automation Portfolio

Projetos de IA aplicada a processos de negócio — n8n, Claude API, RAG e agentes.

---

## 1. Validador de documentos financeiros com IA

**Problema:** conferir manualmente se os valores de documentos financeiros batem é lento e sujeito a erro.

**Solução:** um workflow no n8n envia o documento para o Claude (Haiku 5.5), que confere os valores e devolve um JSON estruturado. Casos conformes seguem automaticamente; casos divergentes vão para validação humana (*human-in-the-loop*).

**Fluxo:** Trigger → HTTP Request (Claude API) → Parse do JSON → If (conforme?) → Enviar ao cliente / Validação humana

**Técnicas aplicadas:**
- System prompt com papel definido
- Dados delimitados com tags XML
- Saída estruturada em JSON
- Credencial da API isolada (nenhuma chave no workflow)

**Testes:**
| Caso | Esperado | Resultado |
|---|---|---|
| Itens somam R$ 1.200 / total R$ 1.200 | conforme | ✅ conforme |
| Itens somam R$ 1.200 / total R$ 1.250 | divergente | ✅ divergente |

**Como usar:** importe o arquivo `.json` no n8n e crie uma credencial *Header Auth* com `x-api-key`.

**Stack:** n8n (self-hosted) · Claude API · JSON

---

## 2. Assistente de LGPD com RAG (Retrieval-Augmented Generation)

📁 Arquivos: [`02-rag-lgpd/`](./02-rag-lgpd)

**Problema:** consultar uma lei extensa é demorado, e LLMs genéricos podem "inventar" artigos.

**Solução:** um agente de IA que responde dúvidas sobre a LGPD **apenas com base no texto oficial da lei**, citando o artigo, e recusa perguntas fora do escopo.

**Arquitetura:**
- **Ingestão:** Formulário (PDF) → Data Loader → Text Splitter → Embeddings (Ollama `nomic-embed-text`) → PostgreSQL + pgvector
- **Consulta:** Chat → AI Agent (Claude Haiku 5.5) → ferramenta de busca vetorial → resposta com citação do artigo

**Decisões técnicas:**
- **Chunks de 1000 caracteres, overlap 200:** pedaços pequenos para busca precisa, com sobreposição para não cortar artigos no meio.
- **Embeddings locais (Ollama):** custo zero e os documentos não saem da máquina — relevante para dados sigilosos.
- **Mesmo modelo de embeddings na ingestão e na consulta:** requisito para que os vetores sejam comparáveis.
- **Haiku para geração:** a busca faz o trabalho pesado; um modelo menor responde com qualidade, mais rápido e mais barato.
- **pgvector em Docker:** mesma tecnologia do Supabase, sem dependência de cota.

**Testes e ajustes:**
| Caso | Esperado | Resultado |
|---|---|---|
| O que é dado pessoal sensível? | Art. 5º, II | ✅ |
| Quais as bases legais? | Art. 7º, incisos I a X | ✅ após aumentar os trechos recuperados (4 → 8) |
| Pergunta fora do escopo | Recusa | ✅ após delimitar o escopo no system prompt |

**Aprendizados:** respostas incompletas vieram da **recuperação** (poucos trechos), não do modelo. Respostas fora do escopo vieram de um **system prompt** pouco restritivo. Cada ajuste foi validado reexecutando todos os casos (teste de regressão).

**Como usar:**
1. Suba o banco: `docker run -d --name pgvector -e POSTGRES_PASSWORD=<sua_senha> -p 5432:5432 pgvector/pgvector:pg16`
2. Instale o Ollama e rode `ollama pull nomic-embed-text`
3. Importe os dois workflows no n8n e configure as credenciais (Postgres, Ollama e Anthropic)
4. Envie o PDF pelo formulário de ingestão e converse pelo chat do workflow de consulta

**Stack:** n8n (self-hosted) · Claude API · Ollama · PostgreSQL/pgvector (Docker)

---

**Autor:** Lucas Jussen · [LinkedIn](https://linkedin.com/in/lucasjussen)
