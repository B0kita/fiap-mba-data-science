# Case 2 · Premissas e estimativa de custo (calculadora AWS)

Região: **us-east-1**. Preços de tabela AWS (on-demand) conferidos em nov/2026, sem free tier. Os valores estão em US$.

## 1. Dados do caso (fonte: slide "Checkpoint 2", aula 2)

| Premissa | Valor | Origem |
|---|---|---|
| Consultas de analistas | 150 analistas × 20 consultas/dia × 22 dias úteis | Enunciado do case 2 |
| Consultas de clientes | 5.000 clientes × 2 consultas/mês | Enunciado do case 2 |
| Pergunta | 100 tokens | Regra de aula |
| Instruções do sistema | 400 tokens | Regra de aula |
| Trechos recuperados | **5 × 500 tokens = 2.500 tokens** (mantido; justificativa abaixo) | Regra de aula + decisão do grupo |
| Resposta | 240 palavras × 1,7 = **408 tokens** | Regra de aula |
| Índice consultado | 66.000 vetores (60.000 da carga + 6.000 do 1º mês) | Saída do case 1 |
| Lambda | 512 MB, ~5 s por consulta (a maior parte esperando o modelo) | Premissa do grupo |
| Registro de auditoria | ~2,5 KB por item (pergunta + resposta ~2 KB em texto + ids, fontes, contagem de tokens) → 3 WRU | Premissa do grupo (DynamoDB cobra 1 WRU por KB arredondado para cima) |

**Por que 5 trechos:** cada trecho traz 400 tokens novos (≈ 2/3 de página), então 5 trechos cobrem ≈ 3,3 páginas de norma. Isso basta para perguntas sobre uma regra ou produto e cabe com folga na janela de contexto. Com 3 trechos, o risco é a norma aplicável ficar de fora (artigo + exceção em páginas diferentes). Com 10, a entrada dobra quase sem ganho, porque os últimos trechos tendem a ser menos relevantes. A sensibilidade está na seção 4.

## 2. Contas (volume × quantidade por unidade)

| Grandeza | Conta | Resultado |
|---|---|---|
| Consultas de analistas | 150 × 20 × 22 | 66.000 |
| Consultas de clientes | 5.000 × 2 | 10.000 |
| **Consultas/mês** | 66.000 + 10.000 | **76.000** |
| Entrada por consulta | 100 + 400 + 5 × 500 | **3.000 tokens** |
| Saída por consulta | 240 × 1,7 | **408 tokens** |
| **Tokens de entrada/mês (LLM)** | 76.000 × 3.000 | **228 milhões** |
| **Tokens de saída/mês (LLM)** | 76.000 × 408 | **31,0 milhões** |
| Tokens de embedding das perguntas | 76.000 × 100 | **7,6 milhões** |
| Buscas vetoriais | 1 por consulta × 76.000 | **76.000 consultas** |
| Dados processados por busca | 66.000 vetores × 4.096 B | 0,27 GB = 0,00027 TB |
| Dados processados/mês | 76.000 × 0,00027 TB | **20,5 TB** |
| Requisições API / Lambda | 1 por consulta | **76.000** |
| GB-s Lambda | 76.000 × 5 s × 0,5 GB | **190.000 GB-s** |
| WRU DynamoDB | 76.000 × 3 | **228.000 WRU** |
| GB de auditoria acumulados | 76.000 × 2,5 KB | **+0,19 GB/mês** |

Erro comum evitado: a entrada **não** é só a pergunta (100 tokens). Os trechos recuperados são 83% da entrada (2.500 ÷ 3.000).

## 3. Preço por serviço (o que vai na calculadora)

| Serviço (calculadora) | Unidade e preço | Conta | US$/mês |
|---|---|---|---|
| Bedrock · Claude Haiku 4.5 · entrada | US$ 1,00 / 1 M tokens | 228 × 1,00 | **228,00** |
| Bedrock · Claude Haiku 4.5 · saída | US$ 5,00 / 1 M tokens | 31,008 × 5,00 | **155,04** |
| Bedrock · Titan Embeddings V2 | US$ 0,02 / 1 M tokens | 7,6 × 0,02 | **0,15** |
| S3 Vectors · consultas | US$ 2,50 / 1 M consultas | 0,076 × 2,50 | **0,19** |
| S3 Vectors · dados processados | US$ 0,004 / TB (índice < 100 mil vetores) | 20,5 × 0,004 | **0,08** |
| S3 Vectors · dados retornados | US$ 0,01 / GB acima de 512 KB por consulta | 5 × ~6 KB = 30 KB < 512 KB | **0,00** |
| API Gateway · HTTP API | US$ 1,00 / 1 M requisições | 0,076 × 1,00 | **0,08** |
| Lambda · requisições | US$ 0,20 / 1 M | 0,076 × 0,20 | **0,02** |
| Lambda · duração (x86) | US$ 0,0000166667 / GB-s | 190.000 × … | **3,17** |
| DynamoDB on-demand · escrita | US$ 0,625 / 1 M WRU | 0,228 × 0,625 | **0,14** |
| DynamoDB · armazenamento | US$ 0,25 / GB-mês | 0,19 × 0,25 | **0,05** |
| CloudWatch Logs | US$ 0,50 / GB; ~3 KB/consulta | 0,23 GB × 0,50 | **0,11** |
| **Total** | | | **≈ US$ 387,03 / mês** |

**Custo por consulta:** 387,03 ÷ 76.000 = **≈ US$ 0,0051** (meio centavo de dólar).
**Item mais caro:** tokens do modelo de linguagem, US$ 383,04, ou **99%** do total. Desse valor, 60% é entrada (trechos) e 40% é saída.

## 4. Sensibilidade

| Cenário | Entrada/mês | Saída/mês | LLM US$/mês | Δ vs base |
|---|---|---|---|---|
| Base: Haiku, 5 trechos, 408 tokens de saída | 228 M | 31,0 M | 383,04 | — |
| 3 trechos | 76.000 × 2.000 = 152 M | 31,0 M | 307,04 | −20% |
| Resposta de 150 palavras (255 tokens) | 228 M | 19,4 M | 324,90 | −15% |
| Claude Sonnet 4.5 (US$ 3 / US$ 15) | 228 M | 31,0 M | 1.149,12 | +200% |
| Volume 10× (760 mil consultas) | 2,28 B | 310 M | 3.830,40 | +900% (linear) |

A busca e o embedding da pergunta somam menos de US$ 0,50. Otimizar custo aqui significa **mandar menos texto ao modelo** (menos trechos, trechos menores, resposta mais curta, cache de perguntas repetidas), não trocar a base vetorial.

## 5. Como lançar na calculadora (calculator.aws)

1. Mesma estimativa do case 1, grupo **"Case 2 · Consulta RAG"**, região US East (N. Virginia).
2. **Amazon Bedrock** → Anthropic Claude Haiku 4.5 → on-demand → input 228.000.000, output 31.008.000 tokens/mês. Em outra linha: Titan Text Embeddings V2 → input 7.600.000.
3. **Amazon S3 Vectors** → 76.000 consultas/mês; índice de 66.000 vetores de 1.024 dim (armazenamento já contado no case 1).
4. **Amazon API Gateway** → HTTP API, 76.000 requisições/mês, payload médio < 512 KB.
5. **AWS Lambda** → 76.000 requisições, 5.000 ms, 512 MB, x86.
6. **Amazon DynamoDB** → on-demand, 228.000 writes (WRU), 0,19 GB de armazenamento.
7. **CloudWatch** → Logs 0,23 GB ingeridos/mês.
8. Exportar o PDF e copiar o link público (em `../custo/`).
