# Entrega 6 – Segurança de Dados

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
