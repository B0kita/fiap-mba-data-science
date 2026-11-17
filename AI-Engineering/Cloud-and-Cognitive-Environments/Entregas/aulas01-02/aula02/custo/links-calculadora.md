# Estimativa na calculadora AWS · Cases 1 e 2

Link público: https://calculator.aws/#/estimate?id=5fbc6c892ef1c2a0ba558b2c608c7d6dcffdfd15
PDF exportado: `estimativa-calculadora.pdf` (região US East (N. Virgínia), gerada em 17/11/2026). As linhas "Case 1" e "Case 2" são as deste exercício; cada linha traz o case, a unidade e o volume na descrição.

| Case | Linhas na calculadora | Calculadora (US$/mês) | Conta manual (`premissas.md`) | Diferença |
|---|---|---|---|---|
| 1 · Ingestão (mês) | Titan V2 0,06 + Lambda 0,03 + S3 Standard 0,10 + vetores (S3 Vectors) ≈ 0,03 | ≈ 0,22 | ≈ 0,22 | Igual |
| 1 · Carga inicial (única) | Titan V2 30 M tokens 0,60 + gravação de vetores 0,08 | 0,68 | ≈ 1,02 | Lambda da carga (0,33) e PUT/GET do S3 (0,01) entram só na conta manual |
| 2 · Consulta RAG | Haiku 4.5 e Titan 383,14 + S3 Vectors 0,30 + API Gateway 0,08 + Lambda 3,19 + DynamoDB 0,19 | ≈ 386,90 | ≈ 387,03 | < 0,1%: o CloudWatch Logs (0,11) não foi lançado, e o Bedrock usa valores inteiros (75.600 req × 3.016 entrada / 410 saída = 228,0 M / 31,0 M tokens) |

Observações de lançamento:
- A calculadora do Bedrock só aceita solicitações por minuto e horas por dia inteiros. Case 2: 6 req/min × 7 h × 30 dias = 75.600 req.
- Lambda lançada sem nível gratuito, igual às premissas. EventBridge (eventos de serviços AWS) e SQS (só falhas) sem custo relevante, não lançados.
