# Segurança de Dados

A arquitetura da plataforma adota diretrizes de proteção e privacidade alinedas à LGPD (Lei nº 13.709/2018) e às exigências de segurança para dados de saúde.

## Classificação dos Dados

Os dados armazenados no PostgreSQL, Redis, S3 e Redshift são divididos por níveis de sensibilidade para definir o grau de proteção necessário:

| Categoria | Nível | Onde fica | Exemplos | Proteção Aplicada |
| :--- | :--- | :--- | :--- | :--- |
| **Público** | Baixo | PostgreSQL / S3 | Termos de uso, catálogo de serviços. | Acesso livre, integridade garantida via HTTPS. |
| **Interno** | Médio | PostgreSQL / Redis / Redshift | Métricas, logs operacionais, cache de agenda. | Acesso restrito à equipe técnica. |
| **Pessoal** | Alto | PostgreSQL | Nome, CPF, e-mail, CRM do médico. | Criptografia em repouso e mascaramento. |
| **Sensível** | Máximo | PostgreSQL / S3 | Prontuários (PEP), prescrições, laudos, exames. | Criptografia forte (AES-256), acesso restrito e retenção legal de 20 anos. |

## Mecanismos de Criptografia

- **Em trânsito:** Todo o tráfego entre os clientes (web/mobile) e os servidores utiliza **HTTPS com TLS 1.3**. As chamadas de vídeo da telemedicina utilizam criptografia de ponta a ponta (E2EE) de forma efêmera na memória.
- **Em repouso:** O banco de dados PostgreSQL e os arquivos armazenados no Amazon S3 utilizam criptografia padrão **AES-256**, garantindo que os dados fiquem ilegíveis caso haja acesso direto aos discos de armazenamento.

## Gestão de Segredos

Para evitar vazamentos de senhas e chaves no repositório de código:

- **Zero Hardcoding:** É estritamente proibido gravar senhas de banco de dados, chaves de API ou tokens diretamente no código-fonte.
- **Variáveis de Ambiente:** Todas as credenciais sensíveis são injetadas na aplicação em tempo de execução através de **variáveis de ambiente** configuradas de forma segura no servidor, mantendo o código limpo e seguro no GitHub.

## Retenção e Conformidade com a LGPD

- **Dados temporários:** Dados de sessão e cache no Redis possuem tempo de expiração curto (TTL) e são descartados automaticamente.
- **Prontuários e exames:** Armazenados no PostgreSQL e S3 pelo prazo obrigatório de **20 anos** (Lei nº 13.787/2018). Pedidos de exclusão pela LGPD são atendidos por **anonimização**, preservando a obrigação legal. Documentos físicos digitalizados são armazenados no S3 com criptografia, permitindo o descarte do papel após a verificação de integridade.
- **Logs de auditoria:** Mantidos por no mínimo 6 meses conforme o Marco Civil da Internet.


## Controle de acesso por perfil

O controle por perfil, chamado **RBAC**, limita o que cada pessoa pode fazer conforme sua função. Além do perfil, o sistema deve verificar a quem pertence o dado e se existe autorização para aquele atendimento. O que não estiver autorizado deve ser bloqueado.

### Matriz de permissões

Na tabela, **próprio** significa pertencente ao usuário conectado. **Vinculado** indica um paciente cujo atendimento está autorizado para o médico na mesma instituição.

| Recurso | Administrador | Médico | Paciente | Sistema externo |
| --- | --- | --- | --- | --- |
| Contas e cadastros | Gerenciar contas e dados administrativos autorizados | Consultar dados dos pacientes vinculados | Consultar dados próprios e atualizar contatos após confirmação | Sem acesso por padrão |
| Agendamentos | Organizar agendas e registrar cancelamentos | Consultar a própria agenda e atualizar seus atendimentos | Agendar, consultar e cancelar consultas próprias | Apenas operações autorizadas para a integração |
| Prontuários | Sem acesso ao conteúdo clínico | Ler histórico autorizado e registrar seus atendimentos | Consultar registros próprios disponibilizados | Sem acesso |
| Prescrições | Consultar apenas a situação de emissão | Emitir e assinar nos atendimentos sob sua responsabilidade | Consultar e baixar prescrições próprias | Serviço de assinatura recebe apenas o documento autorizado |
| Exames e laudos | Sem acesso ao conteúdo clínico | Consultar documentos dos pacientes vinculados | Enviar e consultar documentos próprios | Sem acesso por padrão |
| Planos e pagamentos | Gerenciar com permissão financeira específica | Consultar a cobertura necessária ao atendimento | Consultar plano e pagamentos próprios | Empresa de pagamento atualiza apenas suas transações autorizadas |
| Chamados de suporte | Tratar chamados sem consultar prontuários | Sem acesso por padrão | Abrir e acompanhar chamados próprios | Sem acesso por padrão |
| Relatórios | Consultar resultados resumidos com permissão de gestão | Consultar resultados da própria atividade, se autorizado | Sem acesso direto | Sem acesso direto |
| Registros de auditoria | Consultar somente com autorização específica de auditor | Sem acesso direto | Sem acesso direto | Sem acesso direto |
| Exclusão de prontuários, prescrições e auditoria | Não permitida pelo perfil | Não permitida | Não permitida | Não permitida |

### Regras de uso

- A permissão deve ser conferida no servidor a cada acesso, inclusive em buscas e downloads. Esconder um botão não impede uma tentativa de acesso indevido.
- Pacientes não podem acessar dados de outras pessoas. Médicos precisam de autorização para o atendimento, e uma consulta antiga não garante acesso permanente ao histórico.
- Os dados de uma clínica não ficam disponíveis para outra. Arquivos médicos são privados e seus links de download devem ter prazo curto de validade.
- O administrador não pode aumentar as próprias permissões. Acesso financeiro e acesso de auditoria exigem autorizações separadas.
- Cada sistema externo utiliza uma identificação própria e recebe somente os dados necessários ao serviço, sem acesso direto ao banco.
- Correções em prontuários finalizados devem ser acrescentadas ao registro, com autor, data e motivo, preservando o texto original. Pedidos de exclusão passam por análise específica.

O cadastro público cria apenas contas de pacientes. Médicos passam por validação profissional, e permissões administrativas dependem da aprovação de outro responsável. Os acessos devem ser revisados a cada três meses e retirados imediatamente em caso de desligamento ou mudança de função.

## Autenticação

**Autenticação** confirma quem está entrando no sistema. **Autorização** define o que essa pessoa pode fazer depois de entrar.

A autenticação em dois fatores, chamada **MFA ou 2FA**, combina duas formas de comprovar a identidade. Um exemplo é usar a senha e um código temporário gerado por aplicativo autenticador no celular.

| Perfil | Forma de acesso proposta |
| --- | --- |
| Administrador e auditor | Senha e segundo fator obrigatórios |
| Médico | Senha e segundo fator obrigatórios |
| Paciente | Senha, com segundo fator obrigatório para baixar o histórico completo ou alterar dados de recuperação da conta |
| Sistema externo | Identificação e credencial próprias, limitadas ao serviço autorizado |

Quando o segundo fator for exigido, o acesso só será liberado após sua confirmação. O código não pode ser aceito novamente depois de usado. Entrar como médico não substitui a assinatura digital necessária para emitir uma prescrição.

### Proteção e recuperação da conta

Como regras iniciais do projeto:

- Limitar tentativas de acesso, com bloqueio temporário após cinco falhas em quinze minutos. A mensagem de erro não deve revelar se a conta existe.
- Encerrar a sessão após trinta minutos sem uso ou oito horas de duração, com aviso prévio ao usuário. Sessão é o período em que a pessoa permanece conectada.
- Pedir uma nova confirmação do segundo fator para conceder permissões, baixar o histórico completo ou trocar dados de recuperação.
- Encerrar os acessos anteriores quando houver bloqueio, troca de senha ou recuperação da conta. Uma conta bloqueada não pode continuar acessando dados por já estar conectada.

Se o usuário perder o celular, poderá usar um código de recuperação previamente cadastrado, válido uma única vez. Sem esse código, o suporte deve verificar sua identidade; para médicos e administradores, outro responsável também aprova a recuperação. Depois, as sessões antigas são encerradas e o usuário recebe um aviso no contato já cadastrado.

Mensagens recebidas de sistemas externos também precisam ter sua origem confirmada. Uma informação de pagamento repetida não deve gerar nova cobrança nem alterar transações de outro parceiro.

## Auditoria

A auditoria mantém um histórico das ações no sistema. Esses registros, chamados **logs**, ajudam a identificar quem acessou um dado, quando isso aconteceu e se a operação foi permitida ou negada.

### O que registrar

| Operação | Informações principais |
| --- | --- |
| Entrada, saída e tentativas de login | Usuário identificado, quando possível, horário, origem e resultado, incluindo falhas no segundo fator |
| Troca de senha ou recuperação de conta | Conta afetada, ação realizada e responsável pela aprovação, quando houver |
| Alteração de permissões | Quem solicitou, quem aprovou e o que mudou |
| Leitura ou alteração de prontuário | Profissional ou paciente, registro acessado, horário e resultado; nas correções, motivo e versão do registro |
| Emissão ou cancelamento de prescrição | Responsável, documento e motivo do cancelamento |
| Download ou exportação | Solicitante, documentos envolvidos e resultado |
| Solicitação ou tentativa de exclusão | Responsável, registro, justificativa, aprovação e resultado |
| Comunicação com sistemas externos | Serviço envolvido, operação e resultado, incluindo mensagens rejeitadas ou repetidas |
| Acesso aos próprios logs | Quem consultou ou tentou alterar os registros e o resultado |

A tabela `LOG_ACESSO` já descrita no projeto é a base desse histórico. Cada evento deve identificar a ação, o recurso, a data, a origem e o resultado. Também será necessário identificar a instituição, os serviços externos e as aprovações. Senhas, códigos de autenticação e conteúdo clínico não devem ser copiados para os logs.

**Exemplo:** um médico tenta abrir um prontuário sem autorização. O sistema bloqueia a consulta e registra o médico, o identificador do prontuário, o horário e o motivo da negativa, sem copiar o diagnóstico.

### Como proteger os registros

A aplicação pode acrescentar logs, mas não alterar ou apagar os anteriores. A consulta fica limitada a pessoas autorizadas para auditoria. Essa restrição no banco, sozinha, não impede alterações por um administrador do banco.

Por isso, a proposta também mantém uma cópia separada no Amazon S3, serviço de armazenamento em nuvem, com **Object Lock em modo Compliance**. Esse recurso impede a alteração ou exclusão das versões protegidas durante o prazo definido. São os chamados registros imutáveis: o histórico preservado não pode ser reescrito nesse período.

A proposta inicial é manter noventa dias disponíveis para consulta rápida e doze meses de arquivo protegido. Esses prazos precisam ser alinhados com a equipe responsável pela guarda dos dados antes da implantação; não são apresentados como uma obrigação legal.

Se o envio para o arquivo falhar, os eventos ficam salvos em uma fila persistente, que mantém os dados mesmo após reiniciar o serviço, até o reenvio. Uma alteração importante e seu registro de auditoria devem ser salvos juntos, evitando mudanças sem histórico. Se não for possível guardar o registro com segurança, novas operações protegidas devem ser interrompidas e a equipe avisada. Após a recuperação, deve-se conferir se houve perda ou duplicação de eventos.



### Gestão de Riscos de Segurança

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
