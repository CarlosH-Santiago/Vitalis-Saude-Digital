# Integração entre os Sistemas

A plataforma Vitalis Saúde Digital é composta por cinco partes principais:

1. **Aplicativo do paciente:** permite consultar horários, agendar consultas e acompanhar atendimentos.
2. **Back-end:** gerencia pacientes, médicos, agendamentos e informações clínicas.
3. **Gateway de pagamento:** processa os pagamentos das consultas.
4. **Assinatura digital ICP-Brasil:** garante a autenticidade de receitas, atestados e outros documentos médicos.
5. **Integração com convênios:** verifica a situação do plano, a cobertura das consultas e solicita autorizações às operadoras.

O armazenamento será feito pelo **Amazon RDS (PostgreSQL)** para dados da aplicação e pelo **Amazon S3** para documentos e exames.

## Formato dos dados

A comunicação entre os sistemas utilizará JSON, tanto nas requisições das APIs quanto nas mensagens de eventos.

**Exemplo – Pagamento aprovado:**

```json
{
  "event": "payment.succeeded",
  "data": {
    "transaction_id": "tx_9876543210",
    "appointment_id": "apt_1029384756",
    "amount": 15000,
    "currency": "BRL",
    "status": "paid",
    "payment_method": "pix"
  }
}
```

Os valores e dados apresentados são fictícios. O campo `amount` representa o valor em centavos.

## Mecanismos de comunicação

A comunicação entre o aplicativo e o back-end será realizada por **APIs REST**, utilizadas em operações como cadastro, consulta de horários e agendamentos.

Para tarefas que não exigem resposta imediata, será utilizado o **Amazon SQS**, responsável pelo processamento de mensagens assíncronas.

Os **Webhooks** receberão notificações externas, como confirmações de pagamento. A integração com convênios utilizará as interfaces disponibilizadas pelas operadoras, conforme seus padrões de comunicação.

## Integração com convênios médicos

O paciente poderá informar seu plano de saúde no aplicativo. O back-end consultará a operadora para verificar:

- Se o plano está ativo.
- Se o paciente tem direito ao atendimento.
- Se a consulta está coberta.
- Se é necessária autorização prévia.

Após receber a resposta, o sistema atualizará o status do agendamento. Caso a operadora esteja indisponível, a solicitação ficará pendente, sem considerar o atendimento autorizado.

Os dados do convênio, plano e número de autorização serão vinculados ao paciente e ao agendamento.

## Tratamento de falhas

Para evitar problemas causados por serviços externos, serão adotadas as seguintes medidas:

- **Timeout:** limite de espera de 5 segundos para o gateway e 8 segundos para a assinatura digital. As integrações com convênios também terão limite de espera.
- **Circuit Breaker:** interrompe temporariamente chamadas a serviços que apresentam falhas frequentes.
- **Retry:** realiza novas tentativas em intervalos progressivos (2, 4, 8 e 16 segundos), respeitando o limite de cinco tentativas.
- **Dead Letter Queue (DLQ):** mensagens que não forem processadas serão encaminhadas para uma fila separada no Amazon SQS, permitindo análise e reprocessamento.

## Consistência dos dados e prevenção de duplicidade

Para evitar cobranças, agendamentos e solicitações duplicadas, serão utilizadas três medidas:

**Idempotência:** pagamentos e agendamentos utilizarão o cabeçalho HTTP `Idempotency-Key`. O sistema armazenará o resultado da primeira operação para reconhecer novas tentativas e evitar sua execução duplicada.

**Transações compensatórias:** caso uma etapa falhe, o sistema deverá corrigir as operações anteriores. Por exemplo, se o pagamento for aprovado, mas o agendamento não for concluído, será solicitado o estorno. Para convênios, serão seguidas as regras da operadora para cancelamento de autorizações.

**Restrições de unicidade:** o PostgreSQL utilizará restrições `UNIQUE` para evitar registros duplicados, como identificadores de transações e hashes de assinaturas. Também haverá regras para impedir que dois pacientes ocupem o mesmo horário com o mesmo médico.

Essas medidas ajudam a manter os dados da plataforma consistentes, mesmo em situações de falhas ou requisições simultâneas.
