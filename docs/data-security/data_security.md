# Segurança de Dados

# Classificação, Criptografia e Segredos

A arquitetura da plataforma adota diretrizes de proteção e privacidade alinedas à LGPD (Lei nº 13.709/2018) e às exigências de segurança para dados de saúde.

## 1. Classificação dos Dados

Os dados armazenados no PostgreSQL, Redis, S3 e Redshift são divididos por níveis de sensibilidade para definir o grau de proteção necessário:

| Categoria | Nível | Onde fica | Exemplos | Proteção Aplicada |
| :--- | :--- | :--- | :--- | :--- |
| **Público** | Baixo | PostgreSQL / S3 | Termos de uso, catálogo de serviços. | Acesso livre, integridade garantida via HTTPS. |
| **Interno** | Médio | PostgreSQL / Redis / Redshift | Métricas, logs operacionais, cache de agenda. | Acesso restrito à equipe técnica. |
| **Pessoal** | Alto | PostgreSQL | Nome, CPF, e-mail, CRM do médico. | Criptografia em repouso e mascaramento. |
| **Sensível** | Máximo | PostgreSQL / S3 | Prontuários (PEP), prescrições, laudos, exames. | Criptografia forte (AES-256), acesso restrito e retenção legal de 20 anos. |

## 2. Mecanismos de Criptografia

- **Em trânsito:** Todo o tráfego entre os clientes (web/mobile) e os servidores utiliza **HTTPS com TLS 1.3**. As chamadas de vídeo da telemedicina utilizam criptografia de ponta a ponta (E2EE) de forma efêmera na memória.
- **Em repouso:** O banco de dados PostgreSQL e os arquivos armazenados no Amazon S3 utilizam criptografia padrão **AES-256**, garantindo que os dados fiquem ilegíveis caso haja acesso direto aos discos de armazenamento.

## 3. Gestão de Segredos

Para evitar vazamentos de senhas e chaves no repositório de código:

- **Zero Hardcoding:** É estritamente proibido gravar senhas de banco de dados, chaves de API ou tokens diretamente no código-fonte.
- **Variáveis de Ambiente:** Todas as credenciais sensíveis são injetadas na aplicação em tempo de execução através de **variáveis de ambiente** configuradas de forma segura no servidor, mantendo o código limpo e seguro no GitHub.

## 4. Retenção e Conformidade com a LGPD

- **Dados temporários:** Dados de sessão e cache no Redis possuem tempo de expiração curto (TTL) e são descartados automaticamente.
- **Prontuários e exames:** Armazenados no PostgreSQL e S3 pelo prazo obrigatório de **20 anos** (Lei nº 13.787/2018). Pedidos de exclusão pela LGPD são atendidos por **anonimização**, preservando a obrigação legal. Documentos físicos digitalizados são armazenados no S3 com criptografia, permitindo o descarte do papel após a verificação de integridade.
- **Logs de auditoria:** Mantidos por no mínimo 6 meses conforme o Marco Civil da Internet.

## Controle de acesso, autenticação e auditoria

**Responsável:** Gustavo Bezerra Nonato - COO / Operações de TI  
**Escopo:** Entregas 6.B, 6.C e 6.E da Atividade 3.

Esta proposta define como o VitalisConnect deve identificar usuários, limitar o acesso aos dados e registrar operações relevantes. A referência para os recursos é a [modelagem de dados](data-modeling/data_modeling.md), que descreve pacientes, médicos, consultas, prontuários, prescrições, pagamentos e logs.

Os controles abaixo são requisitos de projeto. O repositório ainda não contém uma aplicação executável nem configurações de infraestrutura que comprovem sua implantação. A classificação dos dados e a criptografia pertencem às seções de Emerson; a matriz de riscos, à seção de Mateus.

## 6.B Controle de acesso por perfil

### 6.B.1 Regras de autorização

O modelo utiliza RBAC: cada perfil recebe apenas as permissões necessárias à sua função. O perfil, sozinho, não autoriza acesso a todos os registros de uma tabela. A API também deve conferir o titular do dado, o vínculo assistencial, a instituição e a situação do recurso. Na ausência de uma permissão expressa ou dos vínculos necessários, o acesso deve ser negado.

Essas verificações devem ocorrer no servidor em cada requisição, inclusive em listagens, downloads e exportações. Ocultar botões na interface não substitui a autorização. Identificadores enviados pelo cliente servem para localizar recursos; a identidade e as permissões devem vir da sessão validada. Essa abordagem segue as orientações de [autorização da OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).

### 6.B.2 Matriz de permissões

Na matriz, **próprio** significa pertencente ao usuário autenticado; **vinculado** significa associado a um atendimento autorizado na mesma instituição. Todas as permissões dependem de conta ativa e sessão válida.

| Recurso ou operação | Administrador | Médico | Paciente | Sistema externo |
| --- | --- | --- | --- | --- |
| Contas, perfis e bloqueios | Gerenciar no escopo administrativo atribuído, com aprovação para concessões privilegiadas | Consultar a própria conta e solicitar correções cadastrais | Consultar a própria conta e alterar contatos após verificação | Sem acesso |
| Cadastro de pacientes | Consultar dados administrativos mínimos; corrigir mediante solicitação registrada | Consultar dados necessários dos pacientes vinculados | Consultar dados próprios; alterações de identidade passam por validação | Sem acesso por padrão |
| Agenda e `CONSULTA` | Organizar agendas e cancelar agendamentos com motivo registrado | Consultar agenda própria e registrar andamento dos atendimentos atribuídos | Agendar, consultar e cancelar consultas próprias conforme as regras do serviço | Apenas operações previstas no contrato da integração |
| `PRONTUARIO` | Sem acesso ao conteúdo clínico pelo perfil administrativo | Ler histórico autorizado; registrar evolução no atendimento sob sua responsabilidade | Consultar os próprios registros disponibilizados; solicitar correção | Sem acesso |
| `PRESCRICAO_DIGITAL` | Consultar apenas estado operacional, sem medicamentos ou posologia | Criar e assinar no atendimento atribuído; cancelar com justificativa e preservação do original | Consultar e baixar prescrições próprias emitidas | Serviço de assinatura recebe somente o documento autorizado e devolve o resultado |
| Exames e laudos no S3 | Sem acesso ao conteúdo clínico | Consultar documentos de pacientes vinculados | Enviar e consultar documentos próprios | Sem acesso por padrão |
| `PLANO_SAUDE` e `PACIENTE_PLANO` | Manter catálogo e validar vínculos no escopo autorizado | Consultar cobertura necessária ao atendimento | Consultar o próprio vínculo e solicitar atualização | Somente validação de cobertura, quando contratada |
| `TRANSACAO_PAGAMENTO` | Consultar e conciliar com permissão financeira específica | Sem acesso pelo perfil clínico | Consultar situação e comprovantes de pagamentos próprios | Gateway informa o estado de transações previamente vinculadas à sua integração |
| `TICKET_SUPORTE_CRM` | Tratar chamados administrativos, sem acesso ao prontuário | Sem acesso por padrão | Abrir e acompanhar chamados próprios | Sem acesso por padrão |
| Relatórios no Data Warehouse | Consultar indicadores agregados com permissão gerencial específica | Consultar indicadores agregados da própria atividade, se habilitado | Sem acesso direto | Sem acesso direto |
| Auditoria | Somente com função de auditor concedida separadamente | Sem acesso direto | Sem acesso direto; solicitações seguem o canal de atendimento | Sem acesso direto |
| Exclusão de prontuários, prescrições e logs | Não permitida pelo perfil | Não permitida | Não permitida | Não permitida |

“Administrador” não equivale a superusuário do banco. Operação, financeiro, gestão de identidades e auditoria devem ser permissões separadas. A conta usada no suporte não recebe acesso clínico nem poder para conceder privilégios a si própria. Um auditor autorizado pode consultar evidências, mas não alterar ou apagar os registros de auditoria.

### 6.B.3 Limites por registro e por instituição

- **Paciente:** a API resolve o vínculo entre a conta autenticada e `PACIENTE.id_paciente`. Alterar um identificador na URL não deve permitir ler dados de outra pessoa.
- **Médico:** a API verifica o credenciamento, a instituição e o vínculo com o atendimento. A leitura do histórico depende de finalidade assistencial e vínculo autorizado, com prazo de revisão; uma consulta antiga não concede acesso permanente. Escrever uma evolução exige que o atendimento esteja atribuído ao profissional e em situação compatível.
- **Instituição:** uma clínica não pode consultar registros de outra apenas por conhecer seus identificadores. O vínculo institucional deve ser obtido no servidor e aplicado também aos resultados de buscas, relatórios e arquivos.
- **Sistema externo:** cada parceiro possui identidade própria e escopos restritos. O gateway não recebe prontuários; o serviço de assinatura não recebe a base de pacientes. O acesso ocorre pela API, sem credenciais diretas de PostgreSQL, Redis, S3 ou Redshift.
- **Documentos:** o backend autoriza o download antes de emitir uma URL temporária do S3. A proposta adota validade máxima de dois minutos e objeto específico. A URL funciona como credencial durante sua validade, não deve aparecer em logs e não pode ser reutilizada como link público permanente. Revogar a sessão impede novas URLs, mas uma URL já emitida pode continuar válida até expirar.
- **Correções clínicas:** um registro finalizado é preservado. A correção deve gerar um adendo vinculado ao original, com autor, motivo e data, em vez de sobrescrever o conteúdo. O modelo de adendos ainda precisa ser incorporado à modelagem.

Contas técnicas internas também devem ser separadas por finalidade. A conta de carga analítica lê somente os campos aprovados para o DW; a conta de backup executa a rotina de cópia; a conta de auditoria grava eventos. Nenhuma delas deve reutilizar uma conta humana ou receber privilégios gerais por conveniência.

### 6.B.4 Concessão, revisão e revogação

O cadastro público cria apenas o perfil Paciente. O perfil Médico depende da validação do credenciamento. Privilégios administrativos são concedidos por solicitação registrada e aprovação de outro responsável autorizado, sem autoaprovação. A ativação de um perfil privilegiado exige MFA configurado.

A proposta prevê revisão trimestral das permissões e revisão imediata em mudança de função, suspensão de credenciamento ou encerramento do vínculo. A revogação deve invalidar sessões e impedir novas autorizações, inclusive quando um token ainda não venceu. A aplicação precisa consultar o estado atual da conta e a versão de suas permissões; se essa verificação estiver indisponível, deve negar o acesso protegido.

## 6.C Autenticação e gestão de sessões

### 6.C.1 Autenticação de pessoas

Autenticação confirma a identidade; autorização determina quais operações essa identidade pode executar. O projeto prevê um provedor de identidade compatível com OpenID Connect, usando Authorization Code com PKCE no acesso web/mobile. O backend valida a sessão ou o token antes de aplicar a matriz de permissões. A escolha e a configuração do provedor ainda fazem parte da implementação.

| Perfil | Política proposta |
| --- | --- |
| Administrador e auditor | MFA obrigatório em cada nova sessão; chave de segurança ou passkey com verificação do usuário como opção preferencial |
| Médico | MFA obrigatório em cada nova sessão; passkey com verificação do usuário ou senha e código TOTP de aplicativo autenticador |
| Paciente | Senha e MFA disponível; segundo fator obrigatório para exportar o histórico completo, alterar fatores de acesso ou confirmar troca de contato usado na recuperação |
| Sistema externo | Identidade de serviço com autenticação própria; não utiliza login humano nem códigos TOTP compartilhados |

O TOTP deve ser de uso único dentro da janela aceita. Códigos de recuperação também são individuais e consumidos uma única vez. SMS não será o mecanismo padrão de segundo fator. Uma passkey só satisfaz a política de MFA quando a configuração exigir e verificar a autenticação local do usuário. A combinação dos fatores e o processo de recuperação devem seguir as orientações de [MFA da OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html).

A política inicial limita tentativas de senha e de segundo fator por conta e origem, com atraso progressivo e bloqueio temporário após cinco falhas em quinze minutos. Esse parâmetro é uma decisão do projeto, sujeita a avaliação operacional. As respostas de erro devem ser genéricas para não revelar se uma conta existe. A confirmação do primeiro fator não deve liberar endpoints protegidos enquanto o segundo fator exigido estiver pendente.

A proteção de senhas, segredos TOTP e chaves fica vinculada às definições da seção de criptografia. MFA de acesso e assinatura digital de prescrição são controles diferentes: autenticar o médico não substitui a assinatura do documento.

### 6.C.2 Sessões e reautenticação

Como configuração inicial, a proposta adota token de acesso com validade de até quinze minutos, encerramento por trinta minutos de inatividade e duração absoluta da sessão de oito horas. Os limites devem ser aplicados no servidor. A interface deve avisar antes do encerramento durante um atendimento e permitir reautenticação, sem prolongar a sessão indefinidamente.

No navegador, a sessão deve usar cookie `Secure`, `HttpOnly` e `SameSite`, com defesa contra CSRF nas operações de alteração. O identificador deve ser renovado após login e mudança de privilégio. No aplicativo móvel, tokens devem ficar no armazenamento seguro do sistema operacional. Quando houver refresh token, a renovação deve usar rotação, detecção de reutilização e revogação da sessão comprometida. As regras de expiração e renovação se apoiam no guia de [gestão de sessões da OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

Logout, bloqueio de conta, troca de senha, recuperação de MFA e revogação de privilégio invalidam as sessões correspondentes no servidor. A validação de tokens verifica assinatura, emissor, destinatário, expiração e escopos; o estado da conta e a versão das permissões continuam obrigatórios em cada acesso. Usar JWT não elimina a necessidade de revogação.

Conceder privilégios, exportar prontuários ou trocar fatores de autenticação exige confirmação recente do segundo fator, realizada nos últimos cinco minutos. Esse prazo também é um parâmetro proposto. A renovação silenciosa do token não conta como nova confirmação do usuário.

### 6.C.3 Recuperação de acesso

Na perda do segundo fator, o usuário utiliza outro fator previamente cadastrado ou um código de recuperação. Se não houver alternativa, a solicitação passa por verificação de identidade no atendimento. Para médicos e administradores, a recuperação manual exige aprovação de outro responsável autorizado. O suporte não deve desabilitar MFA apenas porque recebeu um pedido por telefone ou e-mail.

Ao concluir a recuperação, o sistema revoga sessões anteriores e fatores substituídos, registra o evento e avisa os canais previamente cadastrados. Um contato informado durante o próprio pedido não pode ser a única prova de identidade. Tokens de recuperação devem ter prazo curto e uso único; seus valores não podem aparecer nos logs.

### 6.C.4 Autenticação de integrações

Para comunicação entre serviços, a proposta utiliza OAuth 2.0 Client Credentials com identidade distinta por parceiro e ambiente. A preferência é autenticação do cliente por certificado ou chave privada, conforme suporte do parceiro. Os escopos delimitam operações como atualização de pagamento e retorno de assinatura. O guia de [OAuth 2.0 da OWASP](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html) fundamenta a separação entre autenticação de clientes e acesso de usuários.

Webhooks devem ter sua assinatura ou autenticidade verificada pelo mecanismo do provedor. O backend valida o horário e o identificador do evento para impedir repetição, confere a associação da transação ao parceiro e rejeita mudanças de estado incompatíveis. Receber um evento autenticado não autoriza alterar qualquer consulta. A definição dos endpoints e payloads pertence à entrega de integração de João Guilherme.

## 6.E Auditoria e rastreabilidade

### 6.E.1 Operações registradas

| Grupo | Eventos obrigatórios | Evidência necessária |
| --- | --- | --- |
| Autenticação | Login aceito ou recusado, MFA aceito ou recusado, logout e sessão revogada | Identidade conhecida, origem, horário e resultado |
| Recuperação | Solicitação e conclusão da recuperação; cadastro, troca ou remoção de fator | Conta afetada, responsável pela validação e referência da aprovação |
| Permissões | Concessão, alteração, revogação e tentativa de autoelevação | Executor, conta afetada, perfis anteriores e novos, aprovador e motivo |
| Dados clínicos | Leitura, criação de evolução, adendo, emissão e cancelamento de prescrição | Ator, recurso, identificador, versão, finalidade e resultado |
| Arquivos e exportações | Solicitação de download, emissão de URL temporária e exportação de histórico | Objeto ou conjunto solicitado, solicitante, quantidade e resultado |
| Exclusões | Solicitação, aprovação, execução e tentativa negada de exclusão | Recurso, motivo, solicitante, executor, aprovação e resultado |
| Integrações | Autenticação inválida, webhook aceito, rejeitado ou repetido e alteração de pagamento | Identidade do serviço, evento externo, recurso e correlação |
| Segurança da auditoria | Consulta/exportação de logs, mudança de retenção e tentativa de alterar evidências | Ator privilegiado, ação, configuração afetada e resultado |

Falhas e negativas também são eventos de auditoria. A emissão de uma URL não prova que o arquivo foi baixado: os registros de acesso ao objeto devem complementar essa evidência. Operações sobre prontuários e prescrições finalizados devem preservar os originais. Pedidos de exclusão seguem um fluxo específico de análise e retenção; não são um botão de remoção definitiva disponível aos quatro perfis.

### 6.E.2 Estrutura dos eventos

A entidade `LOG_ACESSO` existente contém `id_log`, `id_usuario`, `acao`, `recurso`, `id_registro`, `resultado`, `ip_origem` e `data_hora`. Ela serve de base, mas precisa ser ampliada para identificar serviços, instituições, versões e operações distribuídas.

| Campo proposto | Finalidade |
| --- | --- |
| `evento_id` | UUID gerado pelo servidor, preservado em reenvios para evitar duplicação |
| `ocorrido_em` e `recebido_em` | Horários em UTC com fuso, distinguindo a ação do recebimento pelo coletor |
| `ator_tipo` e `ator_id` | Identificar usuário, serviço ou ator não identificado, sem inventar um usuário para login inválido |
| `instituicao_id` | Delimitar o contexto institucional; pode ser nulo em eventos anteriores à autenticação |
| `perfil_efetivo` | Registrar o perfil utilizado na decisão |
| `correlacao_id` | Relacionar requisição, evento externo e operação interna sem guardar o token de sessão |
| `motivo_codigo` | Motivo padronizado da negativa ou alteração, evitando texto clínico livre |
| `versao_anterior` e `versao_nova` | Referenciar versões do recurso sem copiar seu conteúdo |
| `aprovacao_id` | Relacionar concessões privilegiadas e exclusões ao processo de aprovação |

Exemplo ilustrativo de acesso negado, com identificadores fictícios:

```json
{
  "evento_id": "f52809b1-8ab5-42bd-b4f5-0b144a501e29",
  "ocorrido_em": "2026-11-03T14:20:00Z",
  "recebido_em": "2026-11-03T14:20:01Z",
  "ator_tipo": "USUARIO",
  "ator_id": "usuario-exemplo-17",
  "instituicao_id": "clinica-exemplo-2",
  "perfil_efetivo": "MEDICO",
  "acao": "LEITURA",
  "recurso": "PRONTUARIO",
  "id_registro": 842,
  "resultado": "NEGADO",
  "motivo_codigo": "SEM_VINCULO_ASSISTENCIAL",
  "correlacao_id": "b5c1cf33-a1c2-41f0-82cc-d1d32ac98951",
  "ip_origem": "192.0.2.10"
}
```

Os eventos não devem conter senhas, códigos MFA, tokens, cookies, URLs assinadas, chaves, CPF, diagnóstico, medicamentos ou cópias de prontuários. Campos de origem devem ser validados e caracteres de controle tratados para impedir injeção nos logs. O IP é obtido da conexão ou de proxy confiável configurado, nunca de um cabeçalho arbitrário. Essas restrições e a proteção das evidências seguem o guia de [logging da OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html).

### 6.E.3 Preservação e imutabilidade

A gravação operacional deve usar uma identidade que apenas insere eventos em `LOG_ACESSO`, sem `UPDATE`, `DELETE`, `TRUNCATE`, propriedade da tabela ou poder de alterar suas permissões. A leitura pertence a uma função de auditoria separada. Essas restrições protegem contra a aplicação comum, mas não tornam a tabela imutável diante de um administrador do banco.

Para preservar evidências fora do banco operacional, os eventos devem seguir para um coletor e para um bucket de auditoria separado dos arquivos clínicos, preferencialmente em conta de segurança distinta. A proposta usa S3 Versioning e Object Lock em modo Compliance, com retenção aplicada a cada versão. Nesse modo, versões protegidas não podem ser sobrescritas ou removidas antes do vencimento da retenção. Novas versões e marcadores de exclusão exigem acompanhamento: a conferência deve usar o identificador da versão preservada, conforme a documentação de [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html).

O coletor deve registrar contagem de eventos, intervalo de horários, identificador do lote e resumo criptográfico em manifesto também protegido. A reconciliação compara a origem com os lotes arquivados para identificar perdas e duplicações. O hash ajuda a detectar alteração; a preservação depende também da retenção, do controle de acesso e da separação administrativa. Se houver criptografia com KMS, a chave deve permanecer disponível durante a retenção, com controles separados para desativação ou exclusão.

Como proposta operacional inicial, os eventos ficam pesquisáveis por noventa dias e preservados por doze meses no arquivo protegido, contados da ocorrência. Esses prazos não são apresentados como exigência legal: precisam ser conciliados com a política de retenção de Emerson e Joalisson antes da implantação. Uma investigação pode exigir suspensão do descarte. O acesso às evidências também gera auditoria, e sua exportação exige finalidade registrada.

### 6.E.4 Falhas de coleta e monitoramento

Nas alterações críticas, a mudança de negócio e o evento pendente de publicação devem ser gravados na mesma transação, usando uma outbox persistente. O publicador reenvia eventos com o mesmo `evento_id` até receber confirmação do coletor. A publicação pode ser repetida; o consumidor deve tratar duplicações. Uma transação desfeita não deve produzir evento de alteração bem-sucedida; sua falha é registrada separadamente.

Leituras clínicas, exportações e concessões de acesso só são liberadas após o aceite durável do evento por uma fila persistente ou armazenamento local protegido. Eventos de autenticação também precisam de coleta durável no provedor ou no serviço responsável. Se não houver caminho durável disponível, novas operações protegidas são interrompidas e a equipe é acionada. Redis de sessão e memória do processo não serão usados como única cópia de auditoria.

Uma indisponibilidade temporária do S3 permite acumular eventos no armazenamento durável. O sistema deve alertar sobre atraso de entrega, crescimento da fila, interrupção da coleta, falhas repetidas de login, tentativas de acesso entre instituições e exportações incomuns. A equipe de operações investiga a causa; o responsável por segurança acompanha incidentes. A recuperação exige reconciliar eventos pendentes e arquivados antes de considerar a coleta normalizada.

## Dependências para implementação

| Dependência identificada | Encaminhamento |
| --- | --- |
| `USUARIO` ainda não está definido, embora seja referenciado em `LOG_ACESSO` | Alinhar com Hudnei e Joalisson a identidade local vinculada ao par emissor/identificador do provedor, seu estado, perfis e vínculos com paciente ou médico |
| O modelo atual não define instituição nem vínculo assistencial com prazo e estado | Modelar esses relacionamentos antes de ativar acesso entre clínicas; a ausência do vínculo deve resultar em negativa |
| Adendos clínicos, aprovações, outbox e campos ampliados de auditoria não existem no esquema atual | Incluir na modelagem e nas migrações antes de implementar os controles correspondentes |
| O arquivo de armazenamento da Atividade 3 está vazio | Confirmar com Hudnei a persistência definitiva; PostgreSQL, S3 e Redshift são referências já presentes nas entregas anteriores, enquanto Redis é previsto para sessões |
| Retenção, proteção de segredos e recuperação do arquivo de auditoria | Compatibilizar com Emerson e Joalisson, incluindo preservação das chaves e testes de recuperação |
| Identidades de parceiros e verificação de webhooks | Alinhar escopos, contratos e eventos com João Guilherme |

## Critérios de aceite

Os cenários abaixo orientam a futura validação da implementação. Eles não representam testes já executados.

| Cenário | Resultado esperado |
| --- | --- |
| Paciente altera o identificador de uma consulta para o de outro titular | Nenhum dado é retornado; negativa registrada |
| Médico tenta ler prontuário sem vínculo ou de outra instituição | Acesso negado, inclusive por listagens e downloads |
| Médico credenciado consulta atendimento atribuído com MFA válido | Acesso permitido e leitura auditada |
| Administrador de suporte tenta ler anamnese ou conceder privilégios a si próprio | Ambas as ações são negadas |
| Login de médico conclui a senha, mas não conclui MFA | Nenhuma sessão com acesso clínico é liberada |
| Conta bloqueada reutiliza token ainda dentro da validade | A API recusa o acesso pelo estado atual da conta |
| Código TOTP ou de recuperação é reapresentado após uso | Nova tentativa é recusada e registrada |
| Gateway envia webhook repetido ou tenta alterar transação de outro parceiro | Repetição não duplica efeitos; alteração indevida é rejeitada e auditada |
| Correção de evolução finalizada é solicitada | Original preservado; adendo vinculado e auditado |
| Aplicação tenta alterar logs; administrador tenta remover versão S3 ainda retida | Operações negadas nos respectivos controles; tentativas acompanhadas |
| S3 fica indisponível e depois retorna | Eventos duráveis são reenviados e reconciliados sem duplicação lógica |
| Fila e armazenamento de auditoria ficam indisponíveis | Novas operações protegidas são interrompidas e um alerta é emitido |
| Log de erro é inspecionado após exportação ou falha de login | Não contém senha, token, URL assinada nem conteúdo clínico |
| Perfis e vínculos válidos solicitam operações permitidas na matriz | Operações funcionam no escopo correto, sem ampliar o acesso a outros titulares |

Para a consolidação de Mateus e Carlos, esta entrega fornece três controles centrais: acesso limitado por função e vínculo, autenticação reforçada nas operações sensíveis e evidências preservadas para investigação. Sua eficácia deverá ser demonstrada pelos critérios de aceite após a implementação.


### 6.F. Gestão de Riscos de Segurança

A Vitalis Saúde Digital adota medidas de segurança para proteger os dados pessoais e médicos armazenados e processados pela plataforma, reduzindo riscos de acessos indevidos, vazamentos, perdas e indisponibilidade dos serviços.

| Risco identificado                       | Impacto | Medida de proteção                                                                                     |
| ---------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------ |
| Acesso indevido a prontuários            | Alto    | Controle de acesso por perfil (RBAC), autenticação multifator e validação de permissões pela API.      |
| Vazamento de senhas e credenciais        | Alto    | Hash seguro de senhas, armazenamento protegido de chaves e gestão de credenciais fora do código-fonte. |
| Interceptação de dados                   | Alto    | Criptografia das comunicações por HTTPS/TLS.                                                           |
| Perda ou exclusão de dados               | Alto    | Backups periódicos, versionamento de arquivos e testes de restauração.                                 |
| Exposição de exames e documentos         | Alto    | Armazenamento privado, controle de permissões e links temporários para downloads.                      |
| Exposição de dados no ambiente analítico | Alto    | Minimização de dados, pseudonimização de identificadores e restrição de acesso ao Redshift.            |

## Controles de segurança

Os controles devem abranger todas as etapas de tratamento das informações:

- Acesso aos dados: cada usuário deve visualizar somente as informações necessárias à sua função, com validação de permissões no backend e registro de acessos.
- Credenciais: senhas devem utilizar hash seguro, enquanto chaves e credenciais técnicas devem permanecer em ambientes protegidos.
- Comunicação: as integrações entre aplicativos, APIs e serviços externos devem utilizar conexões criptografadas.
- Armazenamento: bancos de dados e arquivos médicos devem possuir acesso restrito, cópias de segurança e mecanismos de recuperação.
- Ambiente analítico: os dados enviados ao Redshift devem ser limitados ao necessário para relatórios e indicadores, com proteção dos identificadores pessoais.

## Monitoramento e validação

A efetividade dos controles deve ser verificada por meio de testes periódicos, incluindo:

- Tentativas de acesso a prontuários sem autorização.
- Verificação da proteção de senhas e credenciais.
- Análise da criptografia das comunicações.
- Testes de recuperação de backups.
- Verificação das permissões de acesso a documentos.
- Conferência dos dados disponibilizados no ambiente analítico.

Os resultados devem ser documentados, com registro das falhas identificadas e das ações corretivas necessárias.

## Diretriz de segurança

A segurança da informação deve fazer parte da operação da Vitalis, desde o acesso aos sistemas até o armazenamento e a análise dos dados. Os controles precisam ser revisados continuamente, garantindo a confidencialidade, a integridade e a disponibilidade das informações, em conformidade com a LGPD.
