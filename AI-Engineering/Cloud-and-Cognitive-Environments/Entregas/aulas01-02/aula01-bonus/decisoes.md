# Aula 1 (bônus) · Decisões de serviço

| Necessidade | Escolha | Descartada | Por quê | Requisito |
|---|---|---|---|---|
| Receber documento e selfie | **Amazon API Gateway** (REST, integração `StartSyncExecution`) + URL pré-assinada do S3 para o upload | Upload pelo corpo da API | Imagens de ~1 MB passando pela API encarecem e esbarram no limite de payload. A URL pré-assinada manda direto ao S3 | RF1, RNF1 |
| Guardar o documento | **Amazon S3 Standard** (SSE-KMS, ciclo de vida de 90 dias) | Banco relacional | Arquivo grande, acesso por chave, barato por GB, expiração automática | RNF3 |
| Conduzir o fluxo | **AWS Step Functions Express** (execução síncrona) | Lambda única orquestrando no código / Step Functions Standard | A ordem é fixa e uma etapa depende da outra: não chama o bureau (que custa) se o documento não validou. O Express responde de forma síncrona em < 30 s e cobra por execução e duração. O Standard é assíncrono e mais caro por transição para alto volume | RF5, RNF1 |
| Ler o documento | **Amazon Textract · AnalyzeID** | Modelo próprio de OCR | Sem time de ML. Extrai campos de documento de identidade e cobra por página | RF2 |
| Biometria | **Amazon Rekognition · DetectFaces + CompareFaces** | Fornecedor externo de biometria | Detecta o rosto na selfie e compara com a foto do documento, cobrando por imagem. **Evolução recomendada:** Face Liveness (+US$ 0,015/abertura) contra fraude com foto de foto | RF3 |
| Consultar crédito | **Bureau de crédito (sistema externo)**, chamado pela tarefa HTTP do Step Functions | Lambda intermediária | A tarefa HTTPS nativa dispensa uma função só para repassar a chamada. O custo por consulta é do contrato com o bureau, fora da AWS | RF4 |
| Registrar a proposta e a decisão | **Amazon DynamoDB** on-demand | Amazon RDS | Acesso por id da proposta, sem servidor, paga por escrita. O RDS teria custo fixo 24 h | RF5 |
