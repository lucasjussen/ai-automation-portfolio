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
- **Chunks de 1000 caracteres, overlap 200:** pedaços pequenos para busca precisa, com sobreposição para não cortar artigos no meio. A lei virou 142 trechos.
- **Embeddings locais (Ollama):** custo zero e os documentos não saem da máquina — relevante para dados sigilosos.
- **Mesmo modelo de embeddings na ingestão e na consulta:** requisito para que os vetores sejam comparáveis.
- **Haiku para geração:** a busca faz o trabalho pesado; um modelo menor responde com qualidade, mais rápido e mais barato.
- **pgvector em Docker:** mesma tecnologia do Supabase, sem dependência de cota.

**Testes e ajustes:**
| Caso | Esperado | Resultado |
|---|---|---|
| O que é dado pessoal sensível? | Art. 5º, II | ✅ |
| Quais as bases legais? | Art. 7º, incisos I a X | ✅ após remover trechos duplicados da base e recuperar 8 trechos por busca |
| Pergunta fora do escopo | Recusa | ✅ após delimitar o escopo no system prompt |

**Avaliação (eval):** conjunto de 10 casos (7 sobre a lei, 3 fora do escopo) executado automaticamente no n8n, com nota objetiva por caso (a resposta contém o artigo esperado ou a recusa).
Resultado: **10/10 (100%)**.
Limitação identificada: a ferramenta de busca foi acionada em 5 das 7 perguntas sobre a lei — em parte dos casos o modelo respondeu com conhecimento próprio. Próximo passo: incluir no eval a verificação de uso da ferramenta.

**Aprendizados:**
- **Resposta incompleta era problema de recuperação, não do modelo.** A ingestão do PDF foi executada três vezes, e cada trecho ficou salvo em triplicado. A busca devolvia cópias do mesmo trecho e o fim do Art. 7º ficava de fora. O diagnóstico veio da contagem de registros (426 = 142 × 3); a correção foi limpar a tabela e reingerir uma única vez.
- **Resposta fora do escopo era problema de instrução.** O agente respondia perguntas gerais com conhecimento próprio; delimitar o escopo no system prompt resolveu.
- **Nota final não basta.** O eval deu 100%, mas a contagem de chamadas da ferramenta mostrou que nem toda resposta veio da base — avaliar o caminho, não só o resultado.
- Cada ajuste foi validado reexecutando todos os casos (teste de regressão).

**Como usar:**
1. Suba o banco: `docker run -d --name pgvector -e POSTGRES_PASSWORD=<sua_senha> -p 5432:5432 pgvector/pgvector:pg16`
2. Instale o Ollama e rode `ollama pull nomic-embed-text`
3. Importe os dois workflows no n8n e configure as credenciais (Postgres, Ollama e Anthropic)
4. Envie o PDF **uma vez** pelo formulário de ingestão e converse pelo chat do workflow de consulta
5. Para reingerir o mesmo documento, limpe a tabela antes: `TRUNCATE n8n_vectors;`
6. Para rodar o eval: crie a Data table `eval_lgpd` (colunas `pergunta` e `esperado`) e clique em *Execute workflow*

**Próximos passos:** ingestão idempotente (remover os trechos de um arquivo antes de reinseri-lo) e verificação de uso da ferramenta no eval.

**Stack:** n8n (self-hosted) · Claude API · Ollama · PostgreSQL/pgvector (Docker)

---

**Autor:** Lucas Jussen · [LinkedIn](https://linkedin.com/in/lucasjussen)
