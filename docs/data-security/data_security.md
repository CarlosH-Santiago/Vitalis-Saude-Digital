# Entrega 6 - Segurança de Dados: Classificação, Criptografia e Gestão de Segredos

**Responsável:** Emerson Lucas Sacramento Lima — DPO / Segurança de Dados  
**Escopo:** Classificação das informações, criptografia em trânsito/repouso e gestão de segredos (Secrets Management).

A partir do mapeamento de persistência poliglota definido por Hudnei Sued (composto por PostgreSQL, Redis, Amazon S3 e Amazon Redshift), esta seção estabelece as diretrizes de proteção e privacidade de dados exigidas pela LGPD (Lei nº 13.709/2018) e regulamentações de saúde.

---

## 6.A Classificação dos Dados da Plataforma

Com base nos componentes de armazenamento mapeados, os dados da Vitalis Saúde Digital são classificados em quatro níveis de sensibilidade, definindo o grau de proteção técnica aplicado:

| Categoria do Dado | Nível de Sensibilidade | Onde fica armazenado (Hudnei) | Exemplos de Campos e Recursos | Requisitos de Proteção Exigidos |
| :--- | :--- | :--- | :--- | :--- |
| **Público** | Baixo | PostgreSQL / S3 | Informações institucionais, catálogo de serviços e termos de uso. | Acesso livre, integridade garantida via HTTPS. |
| **Interno / Operacional** | Médio | PostgreSQL / Redis / Redshift | Métricas de uso, logs operacionais, cache de agenda e `FATO_ATENDIMENTO_ANALITICO`. | Acesso restrito à equipe interna; pseudonimização no Data Warehouse. |
| **Pessoal Geral** | Alto | PostgreSQL | Nome, CPF, e-mail, telefone, CRM do médico e dados cadastrais. | Criptografia em repouso, mascaramento em logs e execução via contrato (Art. 7º da LGPD). |
| **Pessoal Sensível (Saúde e Biometria)** | Máximo (LGPD Art. 11) | PostgreSQL / S3 | Prontuários (`PRONTUARIO`), prescrições, laudos, exames em PDF/DICOM, biometria e gravações. | Criptografia forte (AES-256 / KMS), controle rigoroso, logs de acesso e retenção por 20 anos. |

---

## 6.B Mecanismos de Criptografia

A proteção dos dados é dividida entre os momentos em que estão sendo transmitidos e quando estão salvos nos discos da infraestrutura:

### 1. Criptografia em Trânsito (Data in Transit)
* **Aplicações Web e Mobile:** Todo o tráfego de dados entre os clientes e o backend da Vitalis ocorre exclusivamente via **HTTPS com TLS 1.3** (com fallback mínimo para TLS 1.2), impedindo interceptação de dados cadastrais e logins.
* **Telemedicina (Áudio e Vídeo):** As sessões de videoconferência via WebRTC utilizam canal cifrado de ponta a ponta (**End-to-End Encryption - E2EE**). Os dados de mídia fluem de forma efêmera na memória e não ficam gravados, a menos que haja solicitação expressa enviada diretamente ao S3.

### 2. Criptografia em Repouso (Data at Rest)
* **Banco Relacional (PostgreSQL no RDS Multi-AZ):** O volume de dados operacionais e de auditoria (`LOG_ACESSO`) é criptografado utilizando o padrão **AES-256** nativo do Amazon RDS.
* **Arquivos e Anexos (Amazon S3):** Todos os buckets de arquivos (`vitalis-prod-exames`, `vitalis-prod-laudos` e `vitalis-prod-gravacoes`) utilizam criptografia **SSE-KMS** (Server-Side Encryption com AWS Key Management Service). Caso um arquivo seja baixado diretamente do storage sem a chave, ele estará ilegível.
* **Cache em Memória (Redis / ElastiCache):** As sessões e tokens guardados no Redis utilizam criptografia em repouso e trânsito (In-Transit Encryption) para evitar a captura de tokens de autenticação ativos.
* **Data Warehouse (Amazon Redshift):** A base analítica é cifrada por padrão com AES-256 via AWS KMS.

---

## 6.C Gestão de Segredos e Credenciais (Secrets Management)

Para impedir o vazamento de chaves de acesso, senhas de banco e tokens de integração no repositório de código (GitHub), adotam-se as seguintes regras:

1. **Zero Hardcoding:** É proibido inserir senhas do PostgreSQL, credenciais do Redis, chaves da AWS ou segredos de API diretamente no código-fonte das aplicações.
2. **Cofre Centralizado de Senhas:** Todas as credenciais de produção são armazenadas no **AWS Secrets Manager** ou no **AWS Systems Manager Parameter Store**.
3. **Injeção por Variáveis de Ambiente:** Os contêineres e microsserviços consomem as credenciais em tempo de execução através de variáveis de ambiente seguras.
4. **Rotação de Chaves:** As chaves master de criptografia mantidas no AWS KMS passam por rotação automática periódica.

---

## 6.D Alinhamento de Retenção e Anonimização (LGPD)

Em consonância com a arquitetura de persistência:
* **Dados do Redis (Temporários):** Descartados automaticamente via TTL (Time-To-Live) sem deixar rastros permanentes.
* **Prontuários e Exames (PostgreSQL e S3):** Mantidos obrigatoriamente pelo prazo de **20 anos** (Lei nº 13.787/2018). Caso o titular solicite exclusão com base na LGPD, aplica-se a **anonimização/pseudonimização** dos registros clínicos para manter a conformidade legal sem violar a regra do CFM.
* **Logs de Acesso:** Preservados por no mínimo **6 meses** (Marco Civil da Internet) em tabela *append-only* no PostgreSQL e posteriormente arquivados no S3 Glacier.

## Controle de acesso, autenticação e auditoria

**Responsável:** Gustavo Bezerra Nonato – COO / Operações de TI  
**Escopo:** Entregas 6.B, 6.C e 6.E da Atividade 3.

Esta entrega apresenta os controles propostos para proteger os dados do VitalisConnect. O objetivo é garantir que cada usuário acesse apenas as informações necessárias, que a identidade seja confirmada e que ações importantes sejam registradas.

**Importante:** os controles descritos são propostas. Ainda precisam ser implementados e testados.

## 6.B – Controle de acesso por perfil

O sistema utilizará controle de acesso baseado em funções (RBAC). Cada perfil terá permissões específicas, e o acesso também dependerá do vínculo do usuário com o registro e com a instituição.

As permissões serão verificadas pela API em todas as operações. A interface não será responsável por garantir a segurança.

### Matriz resumida de permissões

| Recurso | Administrador | Médico | Paciente | Sistema externo |
|---|---|---|---|---|
| Contas e usuários | Gerenciar conforme permissão | Consultar própria conta | Consultar própria conta | Sem acesso |
| Pacientes | Dados administrativos | Pacientes vinculados | Próprios dados | Sem acesso, salvo integração autorizada |
| Consultas | Gerenciar agenda | Atendimentos atribuídos | Consultas próprias | Operações contratadas |
| Prontuários | Sem acesso clínico | Consultar e atualizar atendimentos autorizados | Consultar próprios registros | Sem acesso |
| Prescrições | Dados operacionais | Emitir e assinar | Consultar próprias prescrições | Serviço de assinatura autorizado |
| Exames e documentos | Sem acesso clínico | Pacientes vinculados | Documentos próprios | Sem acesso por padrão |
| Pagamentos | Permissão financeira específica | Sem acesso | Pagamentos próprios | Transações vinculadas |
| Auditoria | Apenas auditor autorizado | Sem acesso direto | Sem acesso direto | Sem acesso direto |

O administrador não terá acesso irrestrito aos dados clínicos. Permissões administrativas, financeiras e de auditoria serão separadas.

### Regras de acesso

- Pacientes só podem acessar seus próprios dados.
- Médicos acessam apenas pacientes e atendimentos autorizados.
- Instituições não podem acessar dados de outras instituições.
- Sistemas externos terão permissões limitadas à integração contratada.
- Documentos serão disponibilizados por links temporários e autorizados pelo backend.
- Registros clínicos finalizados não serão sobrescritos; correções serão feitas por adendos.

### Gestão de permissões

- O cadastro público cria apenas contas de paciente.
- Médicos precisam ter seu credenciamento validado.
- Permissões administrativas exigem solicitação e aprovação.
- As permissões serão revisadas periodicamente e revogadas quando necessário.
- A revogação deve bloquear novos acessos e invalidar sessões.

## 6.C – Autenticação e sessões

### Autenticação

O sistema utilizará um provedor de identidade compatível com OpenID Connect. A autenticação confirma a identidade do usuário; as permissões definem o que ele pode acessar.

| Perfil | Autenticação proposta |
|---|---|
| Administrador e auditor | MFA obrigatório |
| Médico | MFA obrigatório |
| Paciente | Senha; MFA para operações sensíveis |
| Sistema externo | Credenciais próprias de integração |

MFA significa autenticação por mais de um fator. A proposta prioriza passkeys ou aplicativos autenticadores. Códigos de recuperação devem ser individuais e usados uma única vez.

O sistema limitará tentativas de login e utilizará mensagens genéricas para evitar revelar se uma conta existe.

### Sessões

- Tokens de acesso terão validade curta.
- Sessões serão encerradas após inatividade ou ao atingir o tempo máximo.
- Logout, bloqueio e troca de senha invalidarão sessões.
- Operações sensíveis exigirão autenticação recente.
- Tokens serão armazenados de forma segura, conforme o tipo de dispositivo.

### Recuperação de acesso

A recuperação exigirá um fator alternativo ou validação de identidade. Para médicos e administradores, será necessária aprovação adicional.

Após a recuperação, sessões e fatores antigos serão revogados, e o usuário será notificado.

### Integrações externas

Sistemas externos terão identidade própria e permissões específicas. Webhooks serão validados para evitar eventos falsos ou repetidos.

## 6.E – Auditoria e rastreabilidade

O sistema registrará ações importantes para permitir investigação e acompanhamento de segurança.

### Eventos registrados

- Login, logout e falhas de autenticação.
- Recuperação de conta e alteração de fatores.
- Concessão e revogação de permissões.
- Acesso e alteração de prontuários e prescrições.
- Downloads e exportações de documentos.
- Operações de integração e pagamentos.
- Acesso ou tentativa de alteração dos logs.

Os registros devem identificar quem realizou a ação, quando ocorreu, qual recurso foi afetado e se a operação foi permitida.

### Proteção dos logs

Os logs não devem conter senhas, tokens, códigos MFA ou conteúdo clínico desnecessário.

A aplicação poderá inserir eventos, mas não alterá-los ou apagá-los. Os registros também serão enviados para armazenamento separado e protegido contra alterações.

### Retenção e monitoramento

A proposta inicial é manter os logs pesquisáveis por 90 dias e preservá-los por 12 meses. Esses prazos precisam ser confirmados com a política de retenção do projeto.

O sistema deverá alertar sobre falhas de coleta, tentativas de acesso indevido e atividades incomuns. Se a auditoria estiver indisponível, operações críticas deverão ser interrompidas até que a segurança seja restabelecida.

## Dependências para implementação

- Definir a entidade de usuário e sua relação com pacientes e médicos.
- Criar vínculos entre instituições e atendimentos.
- Incluir adendos clínicos e campos adicionais de auditoria na modelagem.
- Definir armazenamento e retenção dos logs.
- Configurar identidades e permissões das integrações externas.

## Critérios de aceite

A implementação deverá comprovar que:

- Pacientes não acessam dados de outras pessoas.
- Médicos não acessam pacientes sem vínculo autorizado.
- Administradores não acessam prontuários clínicos sem permissão específica.
- MFA é exigido nos perfis e operações definidos.
- Sessões são invalidadas após bloqueio ou revogação.
- Webhooks repetidos não duplicam operações.
- Logs não podem ser alterados pela aplicação.
- Falhas na auditoria geram alertas e bloqueiam operações críticas.

**Conclusão:** a proposta estabelece três medidas principais: acesso restrito por perfil e vínculo, autenticação reforçada e registro protegido das ações. A eficácia dos controles deverá ser validada após a implementação.
