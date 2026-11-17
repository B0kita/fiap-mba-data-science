# Aula 1 (bônus) · Premissas de custo: dois cenários

Preços de tabela AWS (nov/2026). S3, Step Functions, API Gateway e DynamoDB em **sa-east-1**; Textract e Rekognition em sa-east-1 se disponíveis (senão us-east-1, conferir). Os preços de sa-east-1 são mais altos que os de us-east-1 (o S3 Standard, por exemplo, sai ≈ US$ 0,0405/GB): **conferir cada linha na calculadora**. O bureau de crédito é contrato externo, fora da AWS.

| Premissa por abertura | Valor | Origem |
|---|---|---|
| Páginas de documento | 2 (frente e verso) | Premissa do grupo |
| Imagens armazenadas | 3 × ~1 MB = 3 MB | Premissa do grupo |
| Imagens biométricas | 1 DetectFaces + 1 CompareFaces | Fluxo da sequência (mesmos serviços da calculadora de referência da aula 2) |
| Step Functions Express | 1 execução, ~10 s, 64 MB (Textract + Rekognition + bureau) | Premissa do grupo, a medir |
| DynamoDB | 1 escrita de ~2 KB = 2 WRU | Premissa do grupo |

| Item | Preço unitário | Base: 5.000/mês | Pico: 25.000/mês |
|---|---|---|---|
| Textract AnalyzeID | US$ 0,025/página | 10.000 pág → **US$ 250,00** | 50.000 pág → **US$ 1.250,00** |
| Rekognition DetectFaces + CompareFaces | US$ 0,001/imagem × 2 | 10.000 img → **US$ 10,00** | 50.000 img → **US$ 50,00** |
| S3 Standard (sa-east-1), retenção de 90 dias ≈ 3 meses de estoque | ≈ US$ 0,0405/GB-mês | 15 GB × 3 = 45 GB → **US$ 1,82** | 225 GB → **US$ 9,11** |
| Step Functions Express | US$ 1,00/1 M execuções + US$ 0,00001667/GB-s | 5.000 + (5.000 × 10 s × 0,0625 GB = 3.125 GB-s) → **US$ 0,06** | 25.000 + 15.625 GB-s → **US$ 0,29** |
| API Gateway REST API | ≈ US$ 3,50/1 M | 5.000 req → **US$ 0,02** | 25.000 req → **US$ 0,09** |
| DynamoDB on-demand (escrita) | ≈ US$ 0,625/1 M WRU (us-east-1; sa-east-1 é maior) | 10.000 WRU → **US$ 0,01** | **US$ 0,03** |
| **Total AWS** | | **≈ US$ 262/mês (US$ 0,052/abertura)** | **≈ US$ 1.310/mês** |

**Com Face Liveness** (evolução recomendada contra fraude): +US$ 0,015/abertura → +US$ 75 (base) e +US$ 375 (pico).

**Serviço que mais pesa:** o Textract AnalyzeID (95%). A alavanca é cobrar 1 página quando o documento tiver os dados na frente (CNH) e só reprocessar após falha de qualidade da foto, validada no app antes do envio.
