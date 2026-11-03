# Armazenamento e Persistência

_Responsável: Hudnei Sued Passos Santana_

A Vitalis Saúde Digital adota uma arquitetura de persistência poliglota: cada categoria de dado é armazenada na tecnologia mais adequada ao seu padrão de acesso, volume e exigência regulatória. A combinação **PostgreSQL + Redis + Amazon S3** atende ao núcleo operacional, aos dados temporários de baixa latência e aos arquivos não estruturados, enquanto o **Amazon Redshift** complementa a camada analítica.

## A. Armazenamento

### A.1 Tecnologias, justificativas e dados armazenados

| Tecnologia          | Serviço                                  | Dados armazenados                                                                               | Por que foi escolhida                                                                                                                                             |
| :------------------ | :--------------------------------------- | :---------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PostgreSQL**      | Amazon RDS Multi-AZ (região `sa-east-1`) | Cadastros, consultas, prontuários, prescrições, planos, pagamentos, tickets e logs de auditoria | Banco relacional com ACID, integridade referencial e restrições `UNIQUE`/`FK`/`CHECK`; suporta PITR, réplica de leitura e é maduro para dados sensíveis de saúde. |
| **Redis**           | Amazon ElastiCache for Redis             | Sessões de teleconsulta, tokens, cache, locks e horários temporariamente reservados             | Armazenamento em memória com latência sub-milissegundo e TTL nativo; evita sobrecarregar o PostgreSQL com dados que expiram.                                      |
| **Amazon S3**       | S3 (Cloudflare R2 como alternativa/DR)   | Exames, laudos, PDFs, imagens DICOM e gravações de teleconsultas                                | Object storage com alta durabilidade, escalabilidade praticamente ilimitada, criptografia AES-256, políticas de ciclo de vida e URLs pré-assinadas.               |
| **Amazon Redshift** | Redshift                                 | `FATO_ATENDIMENTO_ANALITICO` e dimensões do Data Warehouse                                      | Armazenamento colunar em esquema estrela que isola relatórios pesados do banco operacional.                                                                       |

### A.2 Como os dados são organizados

**PostgreSQL (OLTP).** Os dados são organizados em modelo relacional normalizado (3FN), separados por domínio: `core` (paciente, médico, consulta, prontuário, prescrição), `financeiro` (plano, paciente–plano, transação de pagamento), `suporte` (ticket) e `auditoria` (`LOG_ACESSO`). As chaves primárias são `BIGSERIAL`, as relações usam chaves estrangeiras e há índices em CPF, CRM, `id_paciente` e `data_hora`. As tabelas `CONSULTA` e `LOG_ACESSO` são particionadas por mês/ano, e `LOG_ACESSO` é _append-only_ (sem `UPDATE`/`DELETE`).

**Redis.** As chaves seguem namespaces por finalidade: `sessao:{id_consulta}`, `token:{id_usuario}`, `slot:{id_medico}:{data_hora}` e `cache:agenda:{id_medico}`. Cada tipo possui TTL próprio (sessão de 30 a 60 minutos, token de 15 minutos, reserva de horário de 5 a 10 minutos), e cache, sessão e locks podem usar bases lógicas ou instâncias separadas para evitar que uma limpeza de cache derrube sessões ativas.

**Amazon S3.** Os arquivos são separados por ambiente e finalidade (`vitalis-prod-exames`, `vitalis-prod-laudos`, `vitalis-prod-gravacoes` e `vitalis-lake-raw`), com prefixos por data e vínculo clínico: `exames/{ano}/{mes}/{id_paciente}/{id_consulta}/arquivo.pdf`. Usa-se criptografia SSE-KMS, versionamento e políticas de ciclo de vida por prefixo.

**Amazon Redshift.** Os dados analíticos seguem o esquema estrela já modelado: a tabela fato `FATO_ATENDIMENTO_ANALITICO` conectada às dimensões `DIM_TEMPO`, `DIM_PACIENTE`, `DIM_MEDICO` e `DIM_PLANO_SAUDE`, com chaves de distribuição e ordenação voltadas às consultas por período, médico e clínica.

### A.3 Volume esperado

As estimativas abaixo são premissas de planejamento a serem validadas com métricas reais após o lançamento:

- 50 clínicas clientes, 200 médicos ativos e 50.000 pacientes ativos no primeiro ano;
- 20.000 consultas/mês, com crescimento de aproximadamente 10% ao mês no primeiro ano e duplicação da base em 18 a 24 meses;
- prontuário médio de 10 KB, prescrição de 1 KB, cadastro de 2 KB, log de 0,5 KB por evento e arquivo de 5 MB;
- 30% das consultas geram algum anexo (exame, laudo ou imagem).

| Item                 | Cálculo                                          | Volume estimado                                    |
| :------------------- | :----------------------------------------------- | :------------------------------------------------- |
| Dados clínicos       | 20.000 consultas × 10 KB                         | ~200 MB/mês (~2,4 GB/ano)                          |
| Prescrições          | 30.000 prescrições × 1 KB                        | ~30 MB/mês                                         |
| Cadastros            | 50.000 pacientes × 2 KB + médicos/planos         | ~150 MB no ano 1                                   |
| Logs de auditoria    | 20.000 consultas × 100 eventos × 0,5 KB          | ~1 GB/mês (~12 GB/ano)                             |
| **Total PostgreSQL** | Soma dos itens acima + índices                   | **~1,3 GB/mês (~16 GB/ano; ~80–100 GB em 5 anos)** |
| Arquivos no S3       | 6.000 arquivos × 5 MB                            | **~30 GB/mês (~360 GB/ano; ~1,8 TB em 5 anos)**    |
| Redis                | 2.000 sessões concorrentes × 5 KB + cache/tokens | **~1 a 2 GB de memória com folga**                 |

O volume estruturado é modesto para o PostgreSQL, que suporta com tranquilidade centenas de gigabytes. O crescimento relevante está no S3, motivando o uso de políticas de ciclo de vida.

### A.4 Plano de crescimento da infraestrutura

| Fase                    | Período          | Ações                                                                                                                                                                                                    |
| :---------------------- | :--------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Fase 1 — Lançamento** | 0 a 12 meses     | RDS `db.r6g.xlarge` Multi-AZ com uma réplica de leitura; ElastiCache com dois nós; bucket S3 em Standard; Auto Scaling e Application Load Balancer; monitoramento e alarmes.                             |
| **Fase 2 — Expansão**   | 12 a 24 meses    | Escala vertical do RDS, réplicas de leitura adicionais, RDS Proxy/PgBouncer para pooling; particionamento mensal/anual de `CONSULTA` e `LOG_ACESSO`; Redis em modo cluster; S3 Intelligent-Tiering.      |
| **Fase 3 — Maturidade** | 24 meses ou mais | Particionamento por clínica/tenant, arquivamento de logs e arquivos antigos no S3 Glacier, Redshift RA3 com concurrency scaling, ambiente de DR em outra zona/região e testes periódicos de restauração. |

O plano mantém o objetivo de **RPO inferior a 5 minutos** e **RTO inferior a 1 hora**, com backups PITR no RDS e versionamento no S3. A separação entre OLTP (PostgreSQL) e OLAP (Redshift) evita que o crescimento analítico degrade o atendimento clínico.

## B. Persistência

### B.1 Dados persistentes e dados temporários

| Categoria                             | Exemplos                                                           | Tecnologia                              | Retenção                                                    |
| :------------------------------------ | :----------------------------------------------------------------- | :-------------------------------------- | :---------------------------------------------------------- |
| **Cadastrais (persistentes)**         | Paciente, médico, plano de saúde e vínculo paciente–plano          | PostgreSQL                              | Enquanto o vínculo estiver ativo, com histórico de vigência |
| **Clínicos (persistentes)**           | Prontuário, anamnese, diagnóstico, prescrição digital              | PostgreSQL                              | Mínimo de 20 anos (Lei nº 13.787/2018)                      |
| **Financeiros (persistentes)**        | Transações de pagamento e conciliação                              | PostgreSQL                              | Conforme obrigação fiscal e contábil                        |
| **Auditoria (persistentes)**          | `LOG_ACESSO` (login, leitura, alteração, exportação)               | PostgreSQL particionado + arquivo no S3 | Mínimo de 6 meses (Marco Civil da Internet)                 |
| **Arquivos (persistentes)**           | Exames, laudos, imagens DICOM e gravações                          | Amazon S3                               | Vinculados ao prazo do prontuário, com lifecycle            |
| **Sessão e cache (temporários)**      | Sessão de teleconsulta, tokens, cache, locks e horários reservados | Redis                                   | TTL curto (minutos), sem persistência definitiva            |
| **Mídia em tempo real (temporários)** | Áudio e vídeo WebRTC da teleconsulta                               | Memória/WebRTC                          | Efêmera; gravação só existe se configurada e enviada ao S3  |

Os dados **persistentes** são os que precisam sobreviver ao atendimento e possuir valor legal ou clínico: prontuários, cadastros, prescrições, pagamentos e logs. Já os dados **temporários** existem apenas durante a operação — a sessão da teleconsulta, os tokens de autenticação, o cache de consultas frequentes e os horários bloqueados temporariamente. No Redis, esses dados recebem TTL e são descartados automaticamente, enquanto no PostgreSQL e no S3 a gravação é definitiva e auditável.

### B.2 Inserção, atualização e exclusão

**Inserção.** Todo dado entra pela API do backend, com validação de formato e regras de negócio antes da gravação. Prontuários só podem ser inseridos após o início do atendimento, e cada operação é executada em transação ACID. Arquivos são enviados ao S3 com hash SHA-256 registrado nos metadados.

**Atualização.** Dados cadastrais podem ser atualizados com registro de auditoria. O prontuário é **imutável após o registro**: correções são feitas por novo registro de retificação, preservando o histórico. Status de consulta e de pagamento evoluem por máquina de estados controlada. `LOG_ACESSO` não permite `UPDATE`.

**Exclusão.** Cadastros usam _soft delete_ (`fl_ativo = false`), preservando o histórico. A exclusão física só ocorre quando não há obrigação legal de guarda. Como a LGPD (Art. 18) prevê eliminação, mas a Lei nº 13.787/2018 exige guarda de prontuários por 20 anos, aplica-se **anonimização/pseudonimização** em vez de exclusão para dados clínicos. No Redis, a exclusão é automática por TTL; no S3, por políticas de ciclo de vida e expiração; no Redshift, por rotinas de retenção.

### B.3 Integridade dos dados

A integridade é garantida em quatro níveis: **de entidade** (chave primária em todas as tabelas), **referencial** (chaves estrangeiras entre consulta, prontuário, prescrição e pagamento), **de domínio** (`NOT NULL`, `CHECK` e tipos definidos, como status e valores não negativos) e **de unicidade** (`UNIQUE` em CPF, CRM + UF, `id_consulta` do prontuário e código do gateway). Transações ACID garantem que uma operação seja concluída por inteiro ou revertida. A imutabilidade do prontuário e o caráter _append-only_ dos logs preservam a rastreabilidade, e a validação na aplicação (CPF, CRM, CID-10) impede dados inconsistentes antes da gravação.

### B.4 Prevenção de perda de informações

A prevenção de perda combina redundância, backup e recuperação: o PostgreSQL roda em RDS **Multi-AZ**, com backups automáticos, snapshots e **Point-in-Time Recovery**, mantendo **RPO inferior a 5 minutos** e **RTO inferior a 1 hora**. O S3 usa versionamento, replicação e alta durabilidade, com _Object Lock_ para documentos de guarda obrigatória. O Redis não é fonte de verdade: se perder dados, sessões e caches são recriados a partir do PostgreSQL. O Redshift mantém snapshots próprios. Complementam o plano o monitoramento com alarmes, testes periódicos de restauração e um ambiente de recuperação de desastres.

### B.5 Prevenção de duplicidade de registros

A duplicidade é tratada em camadas complementares:

| Camada             | Mecanismo                                                                                       | Efeito                                                                    |
| :----------------- | :---------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------ |
| **Banco de dados** | `PACIENTE.cpf UNIQUE`                                                                           | Impede dois cadastros com o mesmo CPF                                     |
| **Banco de dados** | `MEDICO (crm, uf_crm) UNIQUE`                                                                   | Impede credenciamento duplicado do mesmo profissional                     |
| **Banco de dados** | `PRONTUARIO.id_consulta UNIQUE`                                                                 | Garante exatamente um prontuário por consulta (1:1)                       |
| **Banco de dados** | `PACIENTE_PLANO (id_paciente, id_plano, data_inicio) UNIQUE`                                    | Evita vínculo de convênio repetido na mesma vigência                      |
| **Banco de dados** | `CONSULTA (id_medico, data_hora) UNIQUE` ou _exclusion constraint_                              | Impede dois agendamentos no mesmo horário                                 |
| **Banco de dados** | `TRANSACAO_PAGAMENTO.codigo_gateway UNIQUE`                                                     | Garante idempotência da cobrança no gateway                               |
| **Data Warehouse** | `DIM_PACIENTE.id_paciente_anonimizado UNIQUE` e `FATO_ATENDIMENTO_ANALITICO.id_consulta_origem` | Evita duplicar paciente ou atendimento na carga analítica                 |
| **Transações**     | ACID e chaves de idempotência                                                                   | Uma gravação é concluída por inteiro ou desfeita, sem registros parciais  |
| **Redis**          | `SET NX` com TTL para reserva de horário                                                        | Cria um lock temporário e impede que dois pacientes reservem o mesmo slot |
| **S3**             | Hash SHA-256 do arquivo no nome/metadados e versionamento                                       | Evita upload duplicado do mesmo exame ou laudo                            |
| **Aplicação/ETL**  | Normalização de CPF (somente dígitos), _trim_, padronização de caixa e _upsert_                 | Reduz duplicidade por variação de preenchimento antes de chegar ao banco  |

Além das restrições, o próprio modelo já separa responsabilidades — dados cadastrais em `PACIENTE` e dados clínicos em `PRONTUARIO` — o que evita redundância e mantém uma única fonte de verdade para cada informação. O pipeline de ETL usa a chave natural `id_consulta_origem` para garantir que cada consulta gere apenas uma linha no fato analítico.
