# Integração entre os Sistemas

A plataforma Vitalis Saúde Digital é composta por cinco partes principais:

1. Aplicativo do paciente: interface utilizada pelo paciente para
   consultar horários disponíveis, agendar consultas e acompanhar seus
   atendimentos.
2. Back-end: responsável pelas regras de negócio, gerenciamento dos
   pacientes, médicos, agendamentos e informações clínicas. Também
   realiza a comunicação com os demais serviços.
3. Gateway de pagamento: responsável por processar os pagamentos
   das consultas e informar ao sistema quando uma transação é aprovada
   ou recusada.
4. Serviço de assinatura digital ICP-Brasil: utilizado para assinar
   e validar documentos médicos, como receitas e atestados, garantindo
   sua autenticidade.
5. Integração com convênios médicos: responsável por consultar a
   situação do plano de saúde, verificar a cobertura da consulta e
   solicitar autorização à operadora quando necessário.

O sistema utilizará o Amazon RDS com PostgreSQL para armazenar os dados
da aplicação e o Amazon S3 para armazenar documentos e exames.

## Formato dos dados

Para a comunicação entre os sistemas, será utilizado o formato JSON,
tanto nas requisições das APIs quanto nas mensagens de eventos.

Exemplo 1 -- Notificação de pagamento aprovado pelo gateway:
{
  "event": "payment.succeeded",
  "timestamp": "2026-11-03T14:30:00Z",
  "data": {
    "transaction_id": "tx_9876543210",
    "appointment_id": "apt_1029384756",
    "patient_id": "pat_550e8400",
    "amount": 15000,
    "currency": "BRL",
    "status": "paid",
    "payment_method": "pix"
  }
}

Exemplo 2 -- Envio de uma receita para assinatura digital:

{
  "prescription_id": "presc_4455667788",
  "appointment_id": "apt_1029384756",
  "doctor": {
    "id": "doc_11223344",
    "crm": "123456/SP"
  },
  "patient": {
    "id": "pat_550e8400"
  },
  "medications": [
    {
      "name": "Amoxicilina 500mg",
      "dosage": "1 cápsula a cada 8 horas",
      "duration": "7 dias"
    }
  ],
  "certificate_type": "ICP_BRASIL_A3"
}

Os dados pessoais e clínicos utilizados nesses exemplos são ilustrativos. Na comunicação real, somente as informações necessárias para cada operação serão compartilhadas.

## Mecanismos de comunicação

A comunicação entre o aplicativo e o back-end será realizada por meio de APIs REST, permitindo operações como cadastro de pacientes, consulta de horários e agendamento de consultas.

Para situações que não precisam de resposta imediata, serão utilizados processos assíncronos. O Amazon SQS será responsável por organizar as mensagens e permitir que o back-end processe eventos sem depender de uma resposta imediata de outros serviços.

Os Webhooks serão utilizados para receber notificações de serviços externos, como a confirmação de um pagamento pelo gateway. Dessa forma, o sistema poderá atualizar o status da consulta sem precisar consultar o gateway repetidamente.

A integração com as operadoras de convênios também será realizada por meio das interfaces disponibilizadas por cada empresa. Quando houver APIs REST disponíveis, elas serão utilizadas para consultar a elegibilidade do paciente, verificar a cobertura e solicitar autorizações. Como cada operadora pode ter um padrão próprio de integração, o back-end será responsável por adaptar as informações trocadas.

## Integração com convênios médicos

A plataforma permitirá que pacientes utilizem seus planos de saúde para realizar consultas. Para isso, o back-end consultará os serviços disponibilizados pelas operadoras e verificará se o plano está ativo, se o paciente tem direito ao atendimento e se a consulta está coberta.

O fluxo funcionará da seguinte maneira:

1. O paciente informa os dados do convênio no aplicativo.
2. O back-end consulta a operadora para verificar a situação do plano e
   a elegibilidade do paciente.
3. O sistema verifica se a consulta está coberta pelo plano.
4. Quando necessário, o back-end envia uma solicitação de autorização à
   operadora.
5. Após receber a resposta, o sistema atualiza o status do agendamento
   e informa o paciente.
6. Caso o atendimento não seja coberto ou a autorização seja negada, o
   paciente será informado e poderá verificar outras formas de
   pagamento.

Serão armazenadas as informações necessárias para identificar o convênio, o plano, o número de autorização (quando houver) e o status da solicitação. Esses dados ficarão vinculados ao paciente e ao agendamento correspondente.

Tratamento de falhas: caso a operadora esteja indisponível, o sistema não deverá considerar a consulta autorizada. A solicitação poderá ser realizada novamente, e o paciente será informado de que a validação está pendente.

Prevenção de duplicidade: cada solicitação de autorização deverá possuir um identificador próprio, permitindo reconhecer tentativas repetidas e evitar solicitações duplicadas à operadora.

## Tratamento de falhas

Para evitar que problemas em serviços externos prejudiquem o funcionamento da plataforma, serão adotadas algumas medidas:

Timeout: as requisições terão um tempo máximo de espera. Inicialmente, será utilizado um limite de 5 segundos para o gateway de pagamento e 8 segundos para o serviço de assinatura digital. Esses valores poderão ser ajustados conforme os testes de desempenho.

Circuit Breaker: caso um serviço externo apresente muitas falhas consecutivas, o sistema interromperá temporariamente as chamadas para ele. Isso evita que uma instabilidade externa prejudique as demais
operações da plataforma.

Retentativas (Retry): quando ocorrer uma falha temporária, o sistema poderá tentar realizar a operação novamente, aumentando o intervalo entre as tentativas (2, 4, 8 e 16 segundos), com limite de cinco tentativas. Esse mecanismo será aplicado somente às operações que podem ser repetidas com segurança.

Dead Letter Queue (DLQ): mensagens que não forem processadas mesmo após as tentativas serão encaminhadas para uma fila separada no Amazon SQS. Assim, poderão ser analisadas e processadas novamente quando o problema for resolvido.

## Consistência dos dados e prevenção de duplicidade

Como a plataforma trabalha com pagamentos, agendamentos, documentos médicos e autorizações de convênios, é importante evitar que uma mesma operação seja executada mais de uma vez.

Para isso, serão utilizadas as seguintes medidas:

Idempotência: as requisições de pagamento e agendamento utilizarão uma chave única, enviada pelo cabeçalho HTTP Idempotency-Key. Caso o usuário tente repetir uma operação por causa de uma falha na conexão, o sistema reconhecerá a chave e evitará executar a mesma operação novamente.

Transações compensatórias: quando uma operação envolver mais de um serviço, o sistema deverá prever como corrigir possíveis falhas. Por exemplo, se o pagamento for aprovado, mas o agendamento não for
concluído, o sistema deverá solicitar o cancelamento ou estorno do pagamento. No caso de convênios, se uma autorização for obtida, mas o atendimento não acontecer, o sistema deverá seguir as regras da
operadora para cancelar ou encerrar a autorização, quando aplicável.

Restrições de unicidade: o banco de dados utilizará restrições UNIQUE para impedir registros duplicados em campos que exigem identificação única, como o identificador da transação e o hash da assinatura digital. Para os agendamentos, também serão aplicadas regras para impedir que dois pacientes ocupem o mesmo horário com o mesmo médico. As solicitações de autorização de convênio também terão identificadores próprios para permitir seu controle.

Essas medidas ajudam a manter os dados consistentes, mesmo quando ocorrem falhas de comunicação ou várias requisições são realizadas ao mesmo tempo.