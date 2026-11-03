# F. Riscos

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
