A plataforma do Vitalis Saúde Digital é formada por 4 partes:

1. App Paciente → a tela onde o paciente vai conseguir fazer o agendamento da consulta dele.
2. Back-end → onde ocorre o sistema para gerenciar os dados clínicos, histórico médico, controle de agendamento e relatórios.
3. Gateaway de pagamento → para os usuários realizarem o pagamento das consultas.
4. Prescrição ICP-Brasil → Serviço de assinatura que valida os documentos médicos ( receitas, atestados e etc ) com o padrão PKI-Brasil.

---

Formato dos dados

Toda a comunicação síncrona e assíncrona foi decidido que vamos usar o formato JSON para a comunicação entre os sistemas. 

Exemplo 1 de Payload de integração com o gateaway:

{
  "event": "payment.succeeded",
  "timestamp": "2026-11-03T14:30:00Z",
  "data": {
    "transaction_id": "tx_9876543210",
    "appointment_id": "apt_1029384756",
    "patient_id": "pat_550e8400-e29b-41d4-a716-446655440000",
    "amount": 15000,
    "currency": "BRL",
    "status": "paid",
    "payment_method": "pix"
  }
}

Exemplo 2 de envio do payload do módulo de ICP-Brasil:

{
  "prescription_id": "presc_4455667788",
  "appointment_id": "apt_1029384756",
  "doctor": {
    "id": "doc_11223344",
    "name": "Dr. Carlos Eduardo",
    "crm": "123456/SP",
    "cpf": "123.456.789-00"
  },
  "patient": {
    "id": "pat_550e8400-e29b-41d4-a716-446655440000",
    "name": "Maria Silva",
    "cpf": "987.654.321-11"
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

Os mecanismos de comunicação que optamos foi através de API's REST e Web Hooks. API's REST para interação em tempo real dos usuários e os médicos, além dos Web Hook que vamos utilizar para notificação de eventos externos onde o tempo de resposta do serviço de origem não deve bloquear a experiência do usuário.

---

Tratamento de falhas

Vamos adotar os seguintes métodos para evitar falhas:

Timeout Estritos → Configuração de tempo limite de resposta ( Ex: 5s pra gateaway e 8s pra serviço )

Padrão Circuit Breaker → Monitoramento da taxa de erro de API's externas. Caso o Gateaway apresenta erros, o circuito é "aberto", interrompendo chamadas temporárias e retornando retornos controlados em vez de congelar a requisição do usuário

Retentativas com Exponential Backoff → Webhooks recebidos com falha ou chamadas assíncronas interrompidas são reprocessados em intervalos progressivamente maiores ( ex.: 2s, 4s, 8s, 16s, até um máximo de 5 tentativas )

Dead Letter Queues → Mensagens de integração que falharem após todas as tentativas automáticas de retentativa são direcionadas para uma fila DLQ no serviço de mensageria para análise

---

Regras de Consistência e Prevenção de Duplicidade

Para evitar erros como cobranças duplicadas ou emissão, vamos seguir esses pontos:

Idempotência de requisições → Todas as chamadas mutacionais de pagamento ou de agendamento vão precisar de um cabeçalho HTTP exclusivo contendo o UUID. exemplo: Se uma requisição der problema por questão de instabilidade na rede, vai poder retomar sem reprocessar a cobrança.

Transação compensatória → Como envolve muitos sistemas e dentro de um processo pode envolver uma ação de cada um, caso falhe em algum ponto deve ter algo pra compensar. exemplo: ( se der cobrança mas falhar no agendamento, deve ter um estorno para compensar )

Restrições de Unicidade e Trava Operacional → Campos críticos no banco de dados (como transaction_id, signature_hash e appointment_id) possuem restrições únicas (UNIQUE INDEX), impedindo persistências duplicadas em concorrência de requisições