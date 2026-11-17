# Aula 1 (bônus) · Evento ou sequência?

**Sequência.** O cliente espera a decisão na mesma chamada (≤ 30 s), e cada etapa depende da anterior: o CPF lido pelo Textract alimenta o bureau, e a biometria precisa da foto do documento. O Step Functions Express, chamado de forma síncrona pela API, conduz as etapas em ordem (e só paga o bureau depois de validar o documento) e decide o desvio em cada falha: documento ilegível vira **recusado**; biometria abaixo do limite ou bureau fora do ar vira **análise manual**. Nenhuma proposta fica sem status.
