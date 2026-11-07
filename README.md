# AI Automation Portfolio

Projetos de IA aplicada a processos de negócio — n8n, Claude API, RAG e agentes.

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

---
**Stack:** n8n (Docker) · Claude API · JSON
