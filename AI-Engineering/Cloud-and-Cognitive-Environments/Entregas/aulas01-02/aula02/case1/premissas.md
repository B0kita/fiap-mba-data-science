# Case 1 · Premissas e estimativa de custo (calculadora AWS)

Região: **us-east-1**. Preços de tabela AWS (on-demand) conferidos em nov/2026. O free tier **não** foi considerado, para a estimativa valer para uma conta que já consome. Os valores estão em US$.

## 1. Dados do caso (fonte: slide "Checkpoint 1", aula 2)

| Premissa | Valor | Origem |
|---|---|---|
| Carga inicial | 2.000 documentos | Enunciado do case 1 |
| Fluxo mensal | 200 documentos novos ou atualizados/mês | Enunciado do case 1 |
| Páginas por documento | 20 páginas de ~100 KB | Enunciado do case 1 |
| Tokens por página | 600 tokens | Regra de aula (350 palavras × 1,7) |
| Trecho | 500 tokens, 20% de sobreposição → 400 tokens novos por trecho | Regra de aula |
| Vetor | 1.024 números × 4 bytes = 4.096 bytes | Regra de aula |
| Metadados por vetor | ~2 KB (texto do trecho para o case 2 montar o prompt + doc_id, página, versão) | Premissa do grupo: 500 tokens ≈ 294 palavras ≈ 1,9 mil caracteres |
| Lambda | 1 GB de memória, ~10 s por documento | Premissa do grupo: leitura do PDF + 30 chamadas ao Titan de ~0,2 s + gravação |
| Documentos digitais | sem OCR | Premissa do grupo (ver decisoes.md) |

## 2. Contas (volume × quantidade por unidade)

**Trechos por documento:** 20 páginas × 600 tokens = 12.000 tokens; 12.000 ÷ 400 tokens novos = **30 trechos/doc**.

**Tokens de embedding por documento:** 30 trechos × 500 tokens = **15.000 tokens/doc**. A sobreposição faz o modelo receber 25% a mais que o texto do documento (15.000 vs 12.000). Erro comum: contar só 12.000.

| Grandeza | Carga inicial (2.000 docs) | Por mês (200 docs) |
|---|---|---|
| Trechos (= vetores) | 2.000 × 30 = **60.000** | 200 × 30 = **6.000** |
| Tokens de embedding | 60.000 × 500 = **30 milhões** | 6.000 × 500 = **3 milhões** |
| GB de arquivos (S3) | 2.000 × 20 × 100 KB = **4,0 GB** | 200 × 20 × 100 KB = **+0,4 GB** |
| GB de vetores (só o vetor) | 60.000 × 4.096 B = **0,246 GB** | 6.000 × 4.096 B = **+0,025 GB** |
| GB de vetores + metadados | 60.000 × 6.144 B = **0,369 GB** | 6.000 × 6.144 B = **+0,037 GB** |
| Invocações Lambda | **2.000** | **200** |
| GB-segundo Lambda | 2.000 × 10 s × 1 GB = **20.000** | 200 × 10 s × 1 GB = **2.000** |

## 3. Preço por serviço (o que vai na calculadora)

| Serviço (calculadora) | Unidade e preço | Carga inicial (única) | Mês 1 (recorrente) |
|---|---|---|---|
| Amazon Bedrock · Titan Text Embeddings V2 | US$ 0,02 / 1 M tokens | 30 M × 0,02 = **US$ 0,60** | 3 M × 0,02 = **US$ 0,06** |
| Amazon S3 Standard · armazenamento | US$ 0,023 / GB-mês | — | 4,4 GB × 0,023 = **US$ 0,10** |
| Amazon S3 · PUT/GET | US$ 0,005 / 1.000 PUT; US$ 0,0004 / 1.000 GET | 2.000 PUT + 2.000 GET = **US$ 0,01** | 200 + 200 = **< US$ 0,01** |
| Amazon S3 Vectors · gravação (PUT) | US$ 0,20 / GB lógico gravado | 0,369 × 0,20 = **US$ 0,07** | 0,037 × 0,20 = **US$ 0,01** |
| Amazon S3 Vectors · armazenamento | US$ 0,06 / GB-mês | — | 0,406 GB × 0,06 = **US$ 0,02** |
| AWS Lambda · requisições | US$ 0,20 / 1 M | 2.000 → **< US$ 0,01** | 200 → **< US$ 0,01** |
| AWS Lambda · duração (x86) | US$ 0,0000166667 / GB-s | 20.000 × … = **US$ 0,33** | 2.000 × … = **US$ 0,03** |
| Amazon EventBridge | Eventos de serviços AWS no barramento padrão: grátis | **US$ 0,00** | **US$ 0,00** |
| Amazon SQS (DLQ) | US$ 0,40 / 1 M requisições | ~0 (só falhas) | ~0 |
| CloudWatch Logs | US$ 0,50 / GB ingerido; ~5 KB por execução | 10 MB → **< US$ 0,01** | 1 MB → **< US$ 0,01** |
| **Total** | | **≈ US$ 1,02 (uma vez)** | **≈ US$ 0,22 / mês** |

**Custo por documento ingerido (mês):** US$ 0,22 ÷ 200 = **≈ US$ 0,0011/doc**.

## 4. Leitura do resultado

- Os tokens são o maior custo variável (US$ 0,06 de US$ 0,22 no mês), mas o total é baixo: a ingestão cresce com **trechos**, não com documentos. Dobrar a sobreposição de 20% para 40% faria cada doc virar 40 trechos (+33% de tokens e vetores).
- O armazenamento acumula: a cada mês entram +0,4 GB no S3 e +0,037 GB no S3 Vectors. Em 12 meses: (4 + 12 × 0,4) GB × 0,023 = US$ 0,20/mês só de originais.
- **Sensibilidade a OCR:** se os documentos forem escaneados, o Textract (DetectDocumentText, US$ 1,50 / 1.000 páginas) acrescenta 40.000 páginas × 0,0015 = **US$ 60 na carga inicial** e 4.000 × 0,0015 = **US$ 6/mês**. Passaria a ser o item mais caro: no mês, US$ 6 contra US$ 0,22 do resto (27×).
- **Documento atualizado:** o S3 versionado guarda a versão antiga. Premissa: uma regra de ciclo de vida apaga versões não atuais após 30 dias. Os vetores antigos são apagados pela Lambda antes de indexar a versão nova.

## 5. Como lançar na calculadora (calculator.aws)

1. Create estimate → região **US East (N. Virginia)**.
2. **Amazon Bedrock** → modelo Titan Text Embeddings V2 → input tokens/mês = 3.000.000 (recorrente). Lançar a carga inicial como um grupo separado, "Carga inicial", com 30.000.000.
3. **Amazon S3** → S3 Standard 4,4 GB; PUT 200; GET 200. **S3 Vectors** (se aparecer como opção do S3): armazenamento 0,41 GB; dados gravados 0,037 GB/mês; consultas = 0 neste case.
4. **AWS Lambda** → 200 requisições/mês, 10.000 ms, 1.024 MB, x86, sem free tier.
5. **Amazon SQS** → 1.000 requisições (margem para falhas). **EventBridge** → 200 eventos de serviço AWS (custo 0).
6. Exportar o PDF e copiar o link público (em `../custo/`).
