# Armazenamento e Persistência

A Vitalis Saúde Digital utiliza serviços da AWS para armazenar e gerenciar os dados da plataforma. Cada serviço possui uma finalidade específica, separando os dados operacionais, arquivos, informações temporárias e dados utilizados em relatórios.

## 1. Tecnologias utilizadas

| Tecnologia                | Serviço AWS                          | Dados armazenados                                                                             |
| ------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------- |
| PostgreSQL                | Amazon RDS                           | Cadastros, consultas, prontuários, prescrições, pagamentos e registros de auditoria.          |
| Armazenamento de arquivos | Amazon S3                            | Exames, laudos, documentos, imagens e gravações autorizadas de teleconsultas.                 |
| Redis                     | Amazon ElastiCache for Redis         | Sessões, cache e reservas temporárias de horários.                                            |
| Data Warehouse            | Amazon Redshift                      | Dados organizados para relatórios e análises da plataforma.                                   |
| Replicação de dados       | AWS Database Migration Service (DMS) | Envio das alterações do banco operacional para o ambiente analítico.                          |
| ETL                       | AWS Glue                             | Preparação e transformação dos dados armazenados no S3 para utilização no ambiente analítico. |

O Amazon RDS é o banco de dados principal da aplicação. O Amazon S3 é responsável pelos arquivos, enquanto o Redis atende às necessidades temporárias da plataforma.

O Amazon Redshift mantém uma estrutura separada para consultas analíticas, evitando que relatórios sobrecarreguem o banco utilizado nas operações diárias.

## 2. Organização dos dados

### 2.1 Banco de dados — Amazon RDS

O PostgreSQL hospedado no Amazon RDS armazena as informações necessárias ao funcionamento da plataforma.

Os principais dados são:

- Dados cadastrais de pacientes e médicos.
- Agendamentos e informações das consultas.
- Prontuários e prescrições.
- Planos de saúde e pagamentos.
- Registros de acesso e alterações realizadas na plataforma.

O banco utiliza relacionamentos entre tabelas e restrições para manter a integridade das informações.

### 2.2 Arquivos — Amazon S3

O Amazon S3 armazena os arquivos enviados à plataforma, como exames, laudos, documentos e imagens.

Os arquivos são organizados por tipo e data. O banco de dados mantém as referências necessárias para localizar cada arquivo.

O acesso aos documentos é controlado pelo backend, evitando que arquivos clínicos fiquem disponíveis publicamente.

### 2.3 Cache e sessões — Amazon ElastiCache for Redis

O Redis é utilizado para armazenar informações temporárias, como:

- Sessões de usuários.
- Dados utilizados frequentemente pela aplicação.
- Reservas temporárias de horários.

Essas informações possuem prazo de validade e podem ser recriadas quando necessário. O Redis não é utilizado como banco principal da plataforma.

### 2.4 Data Warehouse — Amazon Redshift

O Amazon Redshift armazena os dados utilizados em relatórios e análises.

As informações são organizadas em tabelas de fatos e dimensões, permitindo analisar atendimentos, consultas e indicadores da clínica.

Antes de serem disponibilizados para análise, os dados pessoais devem passar por medidas de proteção, como pseudonimização, sempre que aplicável.

Os arquivos originais, como exames e laudos, permanecem no Amazon S3. O Data Warehouse recebe apenas os dados e metadados necessários às análises.

## 3. Persistência dos dados

Os dados são classificados conforme sua finalidade e necessidade de armazenamento.

| Categoria   | Exemplos                                          | Armazenamento   |
| ----------- | ------------------------------------------------- | --------------- |
| Cadastrais  | Pacientes, médicos e planos de saúde              | Amazon RDS      |
| Clínicos    | Prontuários, diagnósticos e prescrições           | Amazon RDS      |
| Financeiros | Pagamentos e transações                           | Amazon RDS      |
| Auditoria   | Acessos e alterações realizadas                   | Amazon RDS      |
| Arquivos    | Exames, laudos e documentos                       | Amazon S3       |
| Temporários | Sessões, cache e reservas de horários             | Redis           |
| Analíticos  | Indicadores e históricos utilizados em relatórios | Amazon Redshift |

Os dados clínicos e cadastrais permanecem armazenados conforme as obrigações legais e as políticas de retenção da plataforma.

As informações temporárias armazenadas no Redis possuem prazo de validade e são removidas automaticamente após esse período.

## 4. Inserção, atualização e exclusão

**Inserção:** os dados enviados pelo aplicativo passam pela API do backend, que verifica as informações antes de encaminhá-las ao banco de dados. Os arquivos são armazenados no S3, e sua referência é registrada no PostgreSQL.

**Atualização:** alterações cadastrais são registradas no banco. Modificações em informações clínicas devem preservar o histórico, identificando o responsável e o momento da alteração.

**Exclusão:** os dados são excluídos conforme as regras da plataforma e as obrigações legais de armazenamento. Quando necessário, o registro pode ser desativado sem apagar imediatamente seu histórico.

A exclusão de arquivos e registros deve respeitar as políticas de retenção aplicáveis aos dados de saúde.

## 5. Integridade e prevenção de duplicidade

A plataforma utiliza mecanismos no banco de dados e no backend para evitar informações inconsistentes e registros duplicados.

Os principais mecanismos são:

| Mecanismo              | Finalidade                                                                         |
| ---------------------- | ---------------------------------------------------------------------------------- |
| Chave primária (PK)    | Identificar cada registro de forma única.                                          |
| Chave estrangeira (FK) | Garantir relacionamentos entre tabelas existentes.                                 |
| Restrição UNIQUE       | Impedir duplicidade em campos que exigem valores únicos.                           |
| NOT NULL e CHECK       | Impedir valores obrigatórios ausentes e informações fora das regras estabelecidas. |
| Transações             | Garantir que operações relacionadas sejam concluídas ou revertidas.                |
| Validação no backend   | Verificar os dados antes de armazená-los.                                          |

Exemplos de aplicação:

- CPF único para cada cadastro de paciente.
- CRM e UF únicos para cada médico.
- Identificação única do prontuário associado à consulta.
- Código único para identificar uma transação de pagamento.

Esses mecanismos reduzem erros de cadastro, duplicidade de registros e inconsistências nas informações clínicas e financeiras.

## 6. Backup e recuperação

A Vitalis utiliza mecanismos de backup para reduzir o risco de perda de informações e permitir a recuperação dos dados em caso de falhas.

| Recurso              | Estratégia de backup                                                                                      |
| -------------------- | --------------------------------------------------------------------------------------------------------- |
| Amazon RDS           | Backups automáticos, snapshots e recuperação para um momento específico (Point-in-Time Recovery).         |
| Amazon S3            | Versionamento dos arquivos e cópias de segurança conforme a política de retenção.                         |
| Amazon Redshift      | Snapshots para recuperação do ambiente analítico.                                                         |
| Redis                | Os dados temporários podem ser recriados a partir das fontes originais.                                   |
| Equipamentos físicos | Cópias das configurações de rede e dos arquivos operacionais importantes em armazenamento protegido no S3. |

### 6.1 Backup da infraestrutura física

Além dos serviços em nuvem, a clínica possui equipamentos físicos necessários ao funcionamento da operação, como computadores, roteadores, switches e outros dispositivos de rede.

Para reduzir os impactos de falhas nesses equipamentos, serão adotadas as seguintes medidas:

- **Computadores:** arquivos importantes para a operação devem possuir cópias de segurança no S3, evitando que permaneçam armazenados exclusivamente nas máquinas locais.
- **Equipamentos de rede:** as configurações de roteadores, switches e firewalls devem ser exportadas após alterações e armazenadas em um bucket protegido no S3.
- **Servidores locais:** caso existam, seus dados e configurações devem possuir uma rotina de cópias de segurança para o S3.
- **Recuperação dos equipamentos:** manter um inventário dos equipamentos e procedimentos para reinstalação e substituição em caso de defeitos.

Os backups protegem os dados e as configurações dos equipamentos. A recuperação do próprio hardware depende de manutenção ou substituição do equipamento.

### 6.2 Recuperação dos dados

Em caso de perda ou falha:

1. O banco PostgreSQL pode ser recuperado utilizando os backups e mecanismos de restauração do RDS.
2. Os arquivos do S3 podem ser recuperados utilizando o versionamento e as cópias de segurança.
3. O ambiente analítico pode ser restaurado a partir dos snapshots do Redshift.
4. As informações temporárias do Redis podem ser recriadas conforme a necessidade da aplicação.
5. Os equipamentos físicos podem ser reconfigurados utilizando as cópias de suas configurações e os procedimentos de recuperação.

Os procedimentos de restauração devem ser testados periodicamente para verificar se os dados podem ser recuperados corretamente.
