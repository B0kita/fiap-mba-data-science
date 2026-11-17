# Aula 1 (bônus) · Requisitos e região: onboarding digital Quantum Finance

## Requisitos funcionais
1. Receber pelo app a foto do documento (frente e verso) e uma selfie com prova de vida.
2. Extrair do documento nome, CPF, data de nascimento e número do documento.
3. Confirmar que a selfie é de uma pessoa real (liveness) e a mesma da foto do documento.
4. Consultar o bureau de crédito com o CPF extraído.
5. Decidir e responder: **aprovado**, **recusado** ou **análise manual**, e registrar a decisão com o motivo.

## Requisitos não funcionais (com número testável)
1. **Latência:** responder em até **30 s em 95%** das aberturas (p95 ≤ 30 s), medidos do envio da selfie até a resposta.
2. **Volume:** suportar **5.000 aberturas/mês** no cenário base e **25.000/mês (pico de 5×)** em campanha, com até **50 aberturas simultâneas** sem degradar o p95.
3. **Disponibilidade e retenção:** API com **99,9%** de disponibilidade mensal. Imagens dos documentos guardadas criptografadas (SSE-KMS) e apagadas automaticamente após **90 dias** da decisão (ciclo de vida do S3).

## Modelo de serviço e região
- **Serverless/FaaS (AWS Step Functions Express, chamado de forma síncrona pela API)** para o processamento: o volume é baixo e irregular (média de ~7 aberturas/hora, picos de campanha), então paga-se só por execução e a escala acompanha o pico de 5× sem planejamento de capacidade. As etapas têm ordem fixa (documento → biometria → bureau → decisão), por isso um orquestrador.
- **Região sa-east-1 (São Paulo)** para armazenamento, banco e computação: os dados pessoais sensíveis ficam em repouso no Brasil (LGPD) e a latência para o usuário brasileiro é menor. **Exceção registrada:** se o Amazon Textract ou o Rekognition não estiverem disponíveis em sa-east-1 (conferir na calculadora), a extração (AnalyzeID) e a biometria são chamadas em **us-east-1**, só para processamento, sem retenção de imagens no destino. A transferência internacional fica coberta pelas garantias contratuais da AWS e precisa ser validada pelo DPO.
