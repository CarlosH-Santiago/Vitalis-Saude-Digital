# Relação com o Negócio e Valor da Solução de Dados

Este documento explica de forma direta como as escolhas técnicas feitas no projeto atendem às necessidades operacionais, financeiras e regulatórias da **Vitalis Saúde Digital**.

---

## A. Data Storage (Armazenamento)

A estratégia de armazenamento foi dividida por tipo de dado e tempo de retenção para equilibrar custo e disponibilidade:

* **Controle de custos no longo prazo (S3 Glacier):** A Lei nº 13.787/2018 exige a guarda de prontuários e exames por 20 anos. Manter terabytes de imagens médicas (DICOM) e PDFs no storage padrão geraria um custo insustentável. Com as regras de ciclo de vida no **AWS S3**, arquivos com mais de 90 dias migram automaticamente para o **S3 Glacier Deep Archive**, reduzindo o custo de armazenamento em mais de 90% (menos de US$ 0,001 por GB/mês).
* **Continuidade do atendimento (RDS Multi-AZ e PITR):** O banco operacional roda no **AWS RDS PostgreSQL** com replicação síncrona em duas zonas de disponibilidade (Multi-AZ). Se o servidor principal falhar, o nó reserva assume em menos de 60 segundos. O backup contínuo dos logs de transação (Point-in-Time Recovery a cada 5 minutos) garante **RPO < 5 minutos** e **RTO < 1 hora**, impedindo perda de dados clínicos.
* **Telas rápidas para o médico (Redis):** O **AWS ElastiCache for Redis** armazena sessões ativas e faz o bloqueio temporário de horários de agenda (`SET NX`). Isso alivia o PostgreSQL de leituras repetitivas e mantém as consultas do Prontuário Eletrônico do Paciente (PEP) respondendo em menos de 100 ms.

---

## B. Data Security (Segurança dos Dados)

A segurança protege a empresa contra penalidades legais e garante a confiança dos médicos e pacientes:

* **Conformidade com a LGPD e mitigação de multas:** Dados de saúde são sensíveis (Art. 11 da LGPD). A plataforma utiliza criptografia **AES-256** em repouso no RDS e S3, e **TLS 1.3** em trânsito. Para relatórios, os dados de pacientes são pseudonimizados com hash HMAC SHA-256 antes de entrarem no Data Warehouse, mitigando o risco de sanções da ANPD (que podem chegar a 2% do faturamento).
* **Segurança jurídica médica (CFM):** Conforme a Resolução CFM nº 2.314/2022, prontuários não podem ser alterados após finalizados. A tabela `PRONTUARIO` é logicamente imutável: qualquer retificação exige a inserção de um adendo auditável com carimbo de tempo, servindo de respaldo pericial para o médico e para a clínica em eventuais litígios.
* **Controle de acesso por função (RBAC):** Profissionais da recepção e do financeiro visualizam apenas dados cadastrais e de pagamento. O acesso ao histórico clínico, diagnósticos (CID-10) e prescrições é restrito exclusivamente ao médico responsável pelo atendimento.
* **Trilha de auditoria (`LOG_ACESSO`):** Qualquer consulta a dados clínicos gera um registro imutável com IP, identificador do usuário e horário exato. O formato *append-only* (sem `UPDATE` ou `DELETE`) cumpre o Marco Civil da Internet e viabiliza auditorias de segurança imediatas.

---

## C. Data Integration (Integração de Sistemas)

As integrações padronizadas melhoram a rotina da clínica e eliminam processos manuais:

* **Cobrança automática e sem duplicidade:** A integração via API REST com o gateway gera cobranças PIX e cartão. O uso de **chaves de idempotência** (`Idempotency-Key`) no cabeçalho HTTP impede cobranças duplicadas em caso de oscilação na internet do paciente. A confirmação é recebida por **Webhooks assinados** com HMAC SHA-256, liberando a consulta sem intervenção manual da recepção.
* **Receitas digitais válidas (ICP-Brasil):** A integração com a autoridade certificadora assina prescrições e atestados com padrão PKI-Brasil e carimbo do tempo. O PDF resultante tem validade legal em qualquer farmácia do país, eliminando blocos de papel, carimbos físicos e custos com motoboy para envio de receitas de medicamentos controlados.
* **Proteção contra falhas externas (Circuit Breaker e DLQ):** Se o gateway de pagamento ou o serviço de assinatura ficarem lentos, a aplicação aplica **timeouts estritos** (5s para pagamento e 8s para assinatura digital). Falhas recorrentes acionam o *Circuit Breaker*, evitando travar a aplicação, e mensagens não entregues seguem para filas de contingência (**Amazon SQS DLQ**) para reprocessamento automático.

---

## D. Data Architecture + Data Modeling

A união da arquitetura em nuvem com a modelagem de dados responde diretamente às 7 necessidades do negócio:

### 1. Dados mais organizados
O banco relacional separa claramente os domínios funcionais:
* **Clínico:** `PACIENTE`, `MEDICO`, `CONSULTA`, `PRONTUARIO`, `PRESCRICAO_DIGITAL`.
* **Financeiro:** `PLANO_SAUDE`, `PACIENTE_PLANO`, `TRANSACAO_PAGAMENTO`.
* **Auditoria:** `TICKET_SUPORTE_CRM`, `LOG_ACESSO`.
* **Analítico:** Tabela fato `FATO_ATENDIMENTO_ANALITICO` e dimensões de tempo, médico, paciente e plano. Cada dado tem uma única fonte da verdade, sem tabelas soltas.

### 2. Redução de inconsistências
A normalização em **3ª Forma Normal (3FN)** impede anomalias de cadastro. Restrições de banco (`UNIQUE` no CPF, no CRM/UF e no ID da consulta do prontuário) barram duplicidades na raiz. As travas no Redis evitam que dois pacientes agendem a mesma vaga do médico (*overbooking*).

### 3. Melhoria na tomada de decisão (Isolamento OLTP vs. OLAP)
A separação física entre o **PostgreSQL (operacional)** e o **Amazon Redshift (Data Warehouse)** é a decisão mais importante da arquitetura:
* A diretoria roda relatórios analíticos de múltiplos anos no Redshift e no Power BI sem consumir memória ou CPU do PostgreSQL.
* O médico atende na ponta sem lentidão ou travamentos de tela durante a escrita do prontuário.

### 4. Aumento da segurança
Os dados clínicos residem em sub-redes privadas da AWS, inacessíveis pela internet pública. Além disso, a separação entre a tabela de cadastro (`PACIENTE`) e a clínica (`PRONTUARIO`) garante que permissões de faturamento nunca exponham diagnósticos médicos.

### 5. Integração entre diferentes sistemas
A modelagem inclui identificadores de rastreabilidade externa:
* `codigo_gateway` na transação financeira para conciliação bancária automática com a operadora.
* `assinatura_digital` (hash SHA-256) na prescrição para validação imediata do documento no validador do ITI/CFM.
* `id_consulta_origem` no Data Warehouse para auditar a linhagem de cada métrica até o registro original.

### 6. Facilidade na evolução do sistema
A divisão modular de microsserviços e a persistência poliglota permitem expandir a plataforma para novas especialidades (como nutrição ou exames laboratoriais) sem precisar reestruturar o núcleo do banco existente.

### 7. Utilização dos dados para gerar valor de negócio (KPIs)
O modelo analítico do Redshift transforma registros brutos em decisões práticas:

| Indicador (KPI) | Origem no Modelo | Decisão de Negócio Viabilizada |
| :--- | :--- | :--- |
| **Taxa de Faltas (*No-Show*)** | `Fato_Atendimento.fl_atendimento_concluido` | Identifica dias e turnos com mais faltas e aciona lembretes automáticos via WhatsApp para diminuir horas ociosas de consultório. |
| **Tempo de Espera e Consulta** | `tempo_espera_min` e `duracao_atendimento_min` | Avalia atrasos e ajusta o tempo padrão dos agendamentos por especialidade. |
| **Rentabilidade por Convênio** | `valor_consulta` e `valor_repasse_medico` | Compara a margem líquida entre atendimentos particulares e planos de saúde para orientar reajustes de contratos. |
| **Sazonalidade Regional** | `DIM_TEMPO` e `DIM_PACIENTE.regiao_brasil` | Ajusta as escalas de médicos plantonistas de telemedicina em períodos de surtos sazonais (ex.: síndromes gripais). |
