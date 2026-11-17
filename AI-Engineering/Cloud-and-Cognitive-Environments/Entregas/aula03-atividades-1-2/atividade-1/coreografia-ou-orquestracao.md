# Atividade 1 · Coreografia ou orquestração (Cases 2 e 3)

Para cada case foram respondidas as duas decisões do slide "Duas decisões, uma depois da outra": **(1) síncrono ou assíncrono** e **(2) coreografia ou orquestração**, com quem conduz e o que quebra se uma etapa falhar.

## Resumo

| | Case 2 · Consulta RAG | Case 3 · Análise de ligações |
|---|---|---|
| Decisão 1 | **Síncrono** | **Assíncrono** |
| Decisão 2 | **Orquestração** | **Orquestração** |
| Quem conduz | A função da API (AWS Lambda), no próprio código | Orquestrador de fluxo (AWS Step Functions Standard) |
| O que dispara | O pedido HTTP do analista ou cliente | O evento "gravação nova" no S3 (via EventBridge) |
| O que quebra se falhar | A pergunta fica sem resposta, mas nada fica pela metade: o usuário pode perguntar de novo | A ligação fica pela metade: transcrita e não analisada, ou analisada sem mascarar (vazamento de dado pessoal) |
| Componentes (slide "Da decisão ao componente") | API (API Gateway) + Função (Lambda) + Registros (CloudWatch) | Barramento só como gatilho (EventBridge) + Orquestrador (Step Functions) + Funções + Registros e alarmes (CloudWatch) |

## Case 2 · Consulta à base de normas (RAG)

**Decisão 1 · Síncrono.** Quem pergunta precisa da resposta agora ("em segundos") e espera na mesma requisição. O volume (76.000/mês ≈ 3.500 por dia útil) não chega em rajadas que exijam fila.

**Decisão 2 · Orquestração, conduzida pela função da API.** A ordem é fixa e cada etapa usa o resultado da anterior (vetor → trechos → prompt → resposta → registro). Toda consulta precisa de trilha de auditoria num lugar só. Não se usa Step Functions: o fluxo é linear, curto (< 30 s) e sem estado a guardar entre etapas, então a própria Lambda conduz e trata a falha. Isso evita a cobrança por transição e a latência extra.

**Se uma etapa falhar:** se o modelo passar de 20 s ou sofrer throttling, a função registra a consulta com status `FALHA_MODELO` e devolve uma mensagem com as fontes já encontradas (**sinalizar**). Se o registro de auditoria falhar, ela tenta de novo 3 vezes e, se persistir, não entrega a resposta (503), porque nenhuma consulta pode ficar sem registro. Diagramas: `case2/c4-n2.mmd` e `case2/sequencia.mmd`.

## Case 3 · Análise das ligações do call center

**Decisão 1 · Assíncrono.** Ninguém espera o resultado na hora: a análise de qualidade e conformidade é consumida depois, em relatório. A transcrição de uma ligação de 6 min leva minutos. As gravações chegam em rajadas no fim de cada turno (20.000/mês ≈ 900 por dia útil), e o fluxo precisa absorver isso sem perder ligação.

**Decisão 2 · Orquestração (Step Functions Standard), iniciada por evento.** A regra de bolso se aplica inteira:
- **Ordem fixa:** cada etapa usa o resultado da anterior (transcrever → mascarar → analisar → gravar).
- **Impacto da falha:** a ligação pode ficar pela metade, e o pior caso é enviar texto **não mascarado** ao modelo (LGPD).
- **Prova:** é preciso mostrar o que aconteceu com **cada** ligação, numa trilha completa num lugar só (histórico de execução).
- **Dono:** um time cuida do fluxo inteiro.

O evento de gravação nova só **inicia** o fluxo: o EventBridge entrega o `Object Created` do S3 a uma execução do Step Functions. Daí em diante o orquestrador conduz. É a combinação descrita na aula: "um evento pode iniciar o fluxo e um orquestrador conduzir as etapas".

**Por que não coreografia:** com cada etapa reagindo ao evento da anterior (transcrição pronta → mascarar → texto mascarado → analisar), ninguém enxerga a ligação inteira. Garantir "nenhuma ligação pela metade" exigiria uma fila de erros por etapa e uma rotina de reconciliação, e a trilha ficaria espalhada em logs de três serviços.

**Se uma etapa falhar (sequência em `case3/sequencia.mmd`):**

| Etapa que falha | Ação do orquestrador | Tipo |
|---|---|---|
| Transcrição (erro do job, limite de jobs simultâneos) | `Retry` 2x com backoff exponencial | **Tentar de novo** |
| Transcrição falha de vez | `Catch` → grava o evento de falha (`status=FALHA_TRANSCRICAO`, motivo, tentativas) no DynamoDB, execução marcada como falha → alarme no CloudWatch; o áudio fica para reprocessar | **Sinalizar** |
| Áudio sem fala ou transcrição vazia | Grava `status=DESCARTADA_SEM_CONTEUDO` e encerra sem chamar o modelo | **Descartar** |
| Verificação do mascaramento encontra CPF, cartão ou telefone remanescente | **Não** envia ao modelo; grava `status=BLOQUEADA_PII` → alarme | **Sinalizar** |
| Modelo com throttling | `Retry` 3x com backoff; se persistir, `status=FALHA_ANALISE` → alarme | **Tentar de novo → sinalizar** |

Toda ligação termina com exatamente um status na tabela de Resultados (DynamoDB) (`OK`, `DESCARTADA_*`, `BLOQUEADA_PII` ou `FALHA_*`). Isso responde a "quem garante que nenhuma ligação fique pela metade": o orquestrador garante.

## Conferência C4 × sequência (item 3 do exercício)

| Elemento do C4 nível 2 (case 3) | Participante da sequência |
|---|---|
| Central de atendimento (sistema externo) | ✓ Central de atendimento |
| Gravações · Amazon S3 | ✓ Gravações (S3) |
| Disparo · Amazon EventBridge | ✓ Disparo (EventBridge) |
| Orquestrador · AWS Step Functions | ✓ Orquestrador (Step Functions) |
| Transcrição + mascaramento · Amazon Transcribe | ✓ Transcrição (Transcribe) |
| Verificação de PII · AWS Lambda | ✓ Verificação de PII (Lambda) |
| Análise · Amazon Bedrock (Claude Haiku 4.5) | ✓ Análise (Bedrock) |
| Resultados · Amazon DynamoDB | ✓ Resultados (DynamoDB) |

8 elementos no C4 e 8 participantes na sequência, os mesmos nos dois. No case 2 também batem: 7 e 7.
