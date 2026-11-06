# Entrega 6 - Segurança de Dados: Classificação, Criptografia e Gestão de Segredos

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




# 6 F. Riscos

A Vitalis Saúde Digital trabalha com dados pessoais e informações de saúde, como prontuários, exames e prescrições. Por isso, é necessário prever situações que possam expor esses dados, impedir o acesso aos sistemas ou causar perda de informações.

A tabela apresenta seis riscos relacionados à arquitetura proposta para o projeto e algumas medidas que podem reduzir seus impactos.

| Risco | Impacto | Estratégia de proteção |
|---|---|---|
| Acesso indevido a prontuários | Alto | Controlar o acesso por perfil (RBAC), exigir autenticação multifator para profissionais e conferir, na API, se o usuário tem vínculo com o paciente. |
| Vazamento de senhas e credenciais | Alto | Armazenar senhas com hash seguro, manter chaves fora do código-fonte e utilizar variáveis de ambiente protegidas ou um gerenciador de segredos. |
| Interceptação de dados durante a transmissão | Alto | Utilizar HTTPS/TLS nas comunicações entre a aplicação, as APIs e os serviços externos. |
| Perda ou exclusão acidental de dados | Alto | Configurar backups do PostgreSQL, manter cópias dos arquivos e testar a restauração dos dados. |
| Exposição de exames e documentos no armazenamento | Alto | Manter os buckets privados, restringir permissões e liberar downloads somente após validar o acesso do usuário. |
| Exposição de dados pessoais no ambiente analítico | Alto | Enviar ao Redshift somente os campos necessários, remover ou pseudonimizar identificadores e limitar o acesso aos relatórios. |

## Detalhamento dos riscos

### 1. Acesso indevido a prontuários

Um usuário pode tentar consultar informações de um paciente sem ter autorização para isso. Na Vitalis, esse problema pode ocorrer se a API aceitar apenas o identificador enviado pelo aplicativo, sem verificar se o profissional tem acesso àquele atendimento.

O controle de acesso deve considerar o perfil do usuário e seu vínculo com o paciente. As permissões precisam ser verificadas no backend, inclusive para consultas, alterações e downloads. As tentativas de acesso também devem ser registradas para auditoria.

### 2. Vazamento de senhas e credenciais

Senhas ou chaves de acesso podem ser expostas se forem salvas diretamente no código, compartilhadas de forma inadequada ou armazenadas sem proteção. Uma credencial comprometida pode permitir acesso aos bancos de dados e aos serviços da plataforma.

As senhas dos usuários devem ser armazenadas por meio de hash seguro, como Argon2id ou bcrypt. As credenciais técnicas devem ficar fora do repositório, em variáveis de ambiente protegidas ou em um gerenciador de segredos. Também é necessário permitir a revogação e a troca de credenciais comprometidas.

### 3. Interceptação de dados durante a transmissão

Os dados enviados entre o aplicativo, o backend e os serviços externos podem ser interceptados quando a comunicação não utiliza criptografia. Isso pode expor informações como dados cadastrais, exames e prescrições.

A comunicação deve utilizar HTTPS/TLS, com certificados válidos. O sistema também deve evitar o envio de informações sensíveis em endereços de URL e exigir autenticação nas APIs.

### 4. Perda ou exclusão acidental de dados

Falhas técnicas, erros de operação ou exclusões indevidas podem causar a perda de prontuários, registros de atendimento e arquivos médicos. Esse problema pode prejudicar o acompanhamento dos pacientes e interromper parte do atendimento.

A arquitetura prevê o uso de PostgreSQL em Amazon RDS, backups e mecanismos de recuperação. Para os arquivos armazenados no S3 ou R2, é importante manter cópias e, quando disponível, utilizar versionamento. Os backups precisam ser testados para confirmar que os dados podem ser recuperados.

### 5. Exposição de exames e documentos armazenados

Exames, laudos e outros documentos podem ficar acessíveis a pessoas não autorizadas se o armazenamento permitir acesso público ou se os links de download forem compartilhados sem controle.

Os arquivos devem ficar em buckets privados. Antes de liberar um documento, a API deve verificar a autorização do usuário. Quando forem utilizadas URLs temporárias, elas devem ter prazo de validade limitado e permitir acesso somente ao arquivo solicitado.

### 6. Exposição de dados no ambiente analítico

O Redshift será utilizado para consultas e relatórios. Se forem enviados dados pessoais além do necessário, ou se muitos usuários tiverem acesso às tabelas, informações dos pacientes poderão ser expostas fora do contexto do atendimento.

O processo de ETL deve selecionar apenas os campos necessários para os indicadores. Sempre que possível, os identificadores diretos devem ser removidos ou pseudonimizados antes da carga. O acesso ao ambiente analítico também deve ser limitado conforme a função de cada usuário.

## Validação no protótipo

As medidas descritas estão relacionadas à arquitetura planejada para a Vitalis. Para demonstrar seu funcionamento, podem ser realizados testes em um protótipo com dados fictícios:

- tentar acessar um prontuário sem vínculo com o paciente e verificar se o sistema bloqueia a consulta;
- conferir se a senha cadastrada não fica armazenada em texto puro e se as credenciais técnicas não aparecem no código;
- verificar se a API utiliza HTTPS/TLS;
- excluir dados de teste e realizar a restauração a partir de um backup;
- tentar acessar um arquivo privado sem autorização e testar a validade de uma URL temporária;
- conferir quais campos são enviados ao ambiente analítico e testar as permissões de consulta.

Os testes devem ser registrados com o procedimento e o resultado obtido. Enquanto não forem executados, esses controles devem ser apresentados como medidas propostas, e não como funcionalidades já implantadas.

## Considerações finais

Os riscos levantados estão ligados aos dados e aos serviços previstos na arquitetura da Vitalis. O controle de acesso, a proteção das credenciais, a criptografia, os backups e a restrição dos arquivos ajudam a reduzir a chance de exposição ou perda de informações.

A análise deverá ser revisada quando o protótipo estiver disponível, para que os controles possam ser testados e as medidas ainda pendentes sejam identificadas.
