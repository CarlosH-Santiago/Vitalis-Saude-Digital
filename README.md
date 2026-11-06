# Vitalis Saúde Digital — Plataforma VitalisConnect

## Equipe e Divisão de Responsabilidades

| Membro | Cargo na Empresa (Vitalis Saúde Digital) | Atribuição no Projeto GitHub |
| :--- | :--- | :--- |
| **Carlos Henrique Santiago** | CEO (Diretor Executivo) | Líder do Repositório & Identidade Institucional |
| **João Guilherme Pinheiro** | CMO / Diretor Comercial | Produtos, Serviços e Ecossistema de Parceiros |
| **Gustavo Bezerra Nonato** | COO (Diretor de Operações) | Organograma e Processos de Negócio |
| **Hudnei Sued Santana** | Arquiteto de Software | Escopo da Plataforma e Ciclo de Dados |
| **Joalisson Pinto Maia** | Engenheiro / DBA de Dados | Infraestrutura de Banco e Armazenamento em Nuvem |
| **Emerson Lucas Sacramento** | DPO (Encarregado LGPD / Segurança) | Proteção de Dados Sensíveis e Criptografia |
| **Mateus Queiroz** | Analista de Governança e Riscos | Análise de Riscos de Dados e Valor de Negócio |


## Apresentação Institucional

A **Vitalis Saúde Digital** (*Vitalis Tecnologia em Saúde e Serviços S.A.*) é uma HealthTech brasileira focada em soluções em nuvem para clínicas, consultórios médicos e hospitais. Seu principal produto é o **VitalisConnect**, um ecossistema que integra telemedicina de baixa latência (WebRTC), Prontuário Eletrônico do Paciente (PEP), emissão de prescrições médicas digitais com validação ICP-Brasil e liquidação financeira integrada.

Seu modelo operacional é híbrido: no segmento B2B, disponibiliza infraestrutura para clínicas, consultórios e hospitais gerenciarem prontuários eletrônicos (PEP) e emitirem prescrições digitais com certificação ICP-Brasil; no segmento B2C, atende diretamente os pacientes por meio de aplicativo próprio para agendamento e realização de teleconsultas médicas eletivas.

---

**Nome Fantasia:** Vitalis Saúde Digital

**Razão Social:** Vitalis Tecnologia em Saúde e Serviços S.A.

**Justificativa da Marca:** O nome Vitalis foi escolhido por sua associação direta com vida e saúde. A expressão "Saúde Digital" complementa a marca, deixando claro o foco da empresa no desenvolvimento de software para o setor médico.

---

**Missão:** Conectar pacientes e profissionais de saúde por meio de uma plataforma simples e segura de telemedicina e prontuário eletrônico, garantindo a organização e a privacidade dos dados clínicos.

**Visão:** Ser uma plataforma de telemedicina e prontuário eletrônico reconhecida por médicos e clínicas pela facilidade de uso, estabilidade do sistema e segurança no tratamento de dados de saúde.

**Valores:**
  * **Segurança e Cuidado com os Dados:** Tratar informações clínicas e cadastrais com sigilo e responsabilidade, seguindo as diretrizes da LGPD e as normas do CFM.
  * **Simplicidade e Usabilidade:** Desenvolver telas e fluxos intuitivos para que médicos e pacientes utilizem a plataforma sem dificuldades.
  * **Confiabilidade e Estabilidade:** Manter a plataforma disponível e operacional, evitando interrupções durante os atendimentos médicos.
  * **Rastreabilidade e Integridade:** Garantir que prescrições, laudos e prontuários sejam autênticos, imutáveis e auditáveis.
---

## Arquitetura Resumida

A arquitetura da Vitalis opera com **segregação física e lógica estrita** entre a camada transacional operacional (OLTP) e a camada de inteligência de negócios (OLAP):

* **Camada Transacional (OLTP):** Processamento de escrita e leitura de baixa latência para garantir que atendimentos médicos, anamneses e agendamentos operem sem lentidão ou bloqueios (*database locking*).
* **Camada Analítica (OLAP):** Data Warehouse em esquema estrela abastecido via pipeline de extração desacoplado (ETL/CDC) com pseudonimização de dados pessoais para geração de relatórios gerenciais e inteligência clínica sem impacto na base operacional.
* **Segurança e Borda:** Application Load Balancer distribuindo requisições HTTP/S entre instâncias gerenciadas por Auto Scaling, cifragem ponta a ponta (TLS 1.3 e WebRTC DTLS/SRTP) e controle de acesso baseado em papéis (RBAC) com autenticação multifator (MFA).

---

## Stack Tecnológica

Toda a infraestrutura da plataforma é padronizada no ecossistema **Amazon Web Services (AWS)** na região de São Paulo (`sa-east-1`):

| Camada | Tecnologia / Serviço AWS | Papel Operacional |
| :--- | :--- | :--- |
| **Banco Operacional (OLTP)** | **AWS RDS PostgreSQL 16 Multi-AZ** | Persistência relacional em 3FN de pacientes, médicos, consultas, prontuários, receitas e trilhas de auditoria com garantias ACID e PITR. |
| **Cache & Sessões** | **AWS ElastiCache for Redis** | Armazenamento volátil sub-milissegundo para tokens JWT, locks atômicos de concorrência de agenda e sessões de telemedicina. |
| **Armazenamento de Mídia** | **AWS S3 (Standard & Glacier)** | Object Storage com criptografia SSE-KMS para laudos em PDF, exames DICOM e receitas assinadas, com ciclo de vida de 20 anos (Lei nº 13.787/2018). |
| **Data Warehouse (OLAP)** | **Amazon Redshift (Cluster RA3)** | Armazenamento colunar distribuído sob Esquema Estrela (Star Schema) para relatórios de BI e KPIs gerenciais. |
| **Rede & Balanceamento** | **AWS ALB + Auto Scaling** | O Application Load Balancer distribui requisições HTTP/S e o Auto Scaling ajusta dinamicamente a computação conforme a carga de atendimento. |
| **Criptografia & Segredos** | **AWS KMS + Secrets Manager** | Criptografia em repouso com algoritmo AES-256 e injeção dinâmica de credenciais em tempo de execução (*zero hardcoding*). |

---


## Índice Geral da Documentação Técnica 

Todo o detalhamento técnico, diagramas, esquemas relacionais e políticas de segurança estão organizados na pasta [`docs/`](docs/):

| Módulo | Documento | Descrição Técnica |
| :--- | :--- | :--- |
| **Contexto da Empresa** | [`README.md`](README.md) | Visão institucional, produtos, modelo de negócio B2B/B2C e conformidade legal. |
| **Data Architecture** | [`docs/data-architecture/data_architecture.md`](docs/data-architecture/data_architecture.md) | Mapeamento dos sistemas da plataforma, fluxo de dados de ponta a ponta e ciclo de vida dos dados. |
| **Data Architecture** | [`docs/data-architecture/consolidated_architecture.md`](docs/data-architecture/consolidated_architecture.md) | Diagrama Mermaid/Visio consolidado integrando *Interfaces, Segurança, Storage (com cache offline), OLTP e DW OLAP*. |
| **Data Architecture** | [`docs/data-architecture/business_value.md`](docs/data-architecture/business_value.md) | Síntese executiva demonstrando o valor de negócio, o isolamento OLTP vs. OLAP e a garantia de continuidade operacional. |
| **Data Modeling** | [`docs/data-modeling/data_modeling.md`](docs/data-modeling/data_modeling.md) | Dicionário de dados, modelo lógico relacional em 3FN (PostgreSQL) e esquema estrela dimensional (Amazon Redshift). |
| **Data Modeling** | [`docs/data-modeling/modeling_business_value.md`](docs/data-modeling/modeling_business_value.md) | Análise de valor da modelagem de dados para a integridade clínica, decisões da diretoria e conformidade com a LGPD. |
| **Data Storage** | [`docs/data-storage/data_storage.md`](docs/data-storage/data_storage.md) | Persistência poliglota (AWS RDS, S3, Redis e **IndexedDB/SQLite local para modo offline**), backups PITR, RPO/RTO e retenção S3 Glacier. |
| **Data Storage** | [`docs/data-storage/physical_document_modeling.md`](docs/data-storage/physical_document_modeling.md) | Procedimentos de custódia, digitalização com OCR e descarte seguro de prontuários em papel e acervos legados. |
| **Data Security** | [`docs/data-security/data_security.md`](docs/data-security/data_security.md) | Classificação de dados sensíveis (LGPD), criptografia (AES-256 e TLS 1.3), matriz RBAC de permissões e mitigação dos 5 principais riscos. |
| **Data Integration** | [`docs/data-integration/data_integration.md`](docs/data-integration/data_integration.md) | Especificação de APIs REST/JSON, Webhooks assinados, integração com certificação digital ICP-Brasil e filas de sincronização offline (DLQ). |

---


## Diretrizes de Operação e Resiliência

* **RPO (Recovery Point Objective):** $< 5$ minutos via replicação contínua de WAL no Amazon RDS.
* **RTO (Recovery Time Objective):** $< 1$ hora para restabelecimento total da operação em caso de desastre.
* **Retenção Regulatória:** Prontuários e laudos armazenados por no mínimo **20 anos** via regras de transição automática para o S3 Glacier Deep Archive.
* **Privacidade:** Pseudonimização irreversível aplicada a qualquer dado transferido da base operacional para o Data Warehouse.