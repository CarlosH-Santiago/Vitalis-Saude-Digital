# Vitalis Saúde Digital — Plataforma VitalisConnect

> Documentação técnica oficial de engenharia de software, arquitetura de dados e infraestrutura em nuvem da Vitalis Saúde Digital.

---

## 🏥 Apresentação Institucional

A **Vitalis Saúde Digital** (*Vitalis Tecnologia em Saúde e Serviços S.A.*) é uma HealthTech brasileira especializada em infraestrutura em nuvem para o setor de saúde. A empresa desenvolve e opera a plataforma **VitalisConnect**, um ecossistema integrado que combina telemedicina de baixa latência (WebRTC), Prontuário Eletrônico do Paciente (PEP), emissão de prescrições médicas digitais válidas via certificação ICP-Brasil e gestão financeira unificada.

A companhia atua sob modelo híbrido:
* **B2B (SaaS):** Fornecimento da plataforma para clínicas particulares, consultórios e hospitais de médio porte gerenciarem prontuários eletrônicos com conformidade com a LGPD e resoluções do CFM.
* **B2C:** Atendimento médico direto a pacientes por meio de aplicativo próprio para agendamento, liquidação e realização de teleconsultas eletivas.

---

## 🏛️ Arquitetura Resumida

A arquitetura de software e dados opera sob **segregação física e lógica estrita** entre a camada transacional operacional (OLTP) e a camada analítica (OLAP):

* **Camada Transacional (OLTP):** Processamento de escrita e leitura de baixa latência no **AWS RDS PostgreSQL 16 Multi-AZ** e **AWS ElastiCache for Redis**, blindando o atendimento médico contra lentidão ou bloqueios de concorrência (*database locking*).
* **Camada Analítica (OLAP):** Data Warehouse em **Esquema Estrela (Star Schema)** no **Amazon Redshift (Cluster RA3)**, alimentado via pipeline assíncrono (ETL/CDC) com pseudonimização obrigatória (LGPD) para relatórios gerenciais sem impacto na base clínica.
* **Resiliência e Borda:** Application Load Balancer distribuindo requisições HTTP/S sob proteção do AWS WAF & Shield, terminação segura em **TLS 1.3** e suporte a modo de contingência *Local-First* (IndexedDB) para continuidade do PEP médico mesmo em instabilidades de rede.

---

## 🛠️ Stack Tecnológica Oficial (100% AWS)

Toda a infraestrutura em produção é padronizada no ecossistema da **Amazon Web Services (AWS)** na região de São Paulo (`sa-east-1`):

| Camada | Tecnologia / Serviço AWS | Papel Operacional |
| :--- | :--- | :--- |
| **Banco Operacional (OLTP)** | **AWS RDS PostgreSQL 16 Multi-AZ** | Persistência relacional em 3FN de pacientes, médicos, consultas, prontuários e auditoria com garantias ACID e PITR. |
| **Cache & Sessões** | **AWS ElastiCache for Redis** | Armazenamento volátil sub-milissegundo para tokens JWT, locks atômicos de agenda (`SET NX`) e sessões WebRTC. |
| **Armazenamento de Mídia** | **AWS S3 (Standard & Glacier)** | Object Storage com criptografia SSE-KMS para laudos em PDF, exames DICOM e receitas com ciclo de vida de 20 anos (Lei nº 13.787/2018). |
| **Data Warehouse (OLAP)** | **Amazon Redshift (Cluster RA3)** | Armazenamento colunar distribuído sob Esquema Estrela (Star Schema) para relatórios executivos e BI. |
| **Rede & Balanceamento** | **AWS ALB + Auto Scaling** | O Application Load Balancer distribui requisições HTTP/S e o Auto Scaling gerencia nós de computação sob demanda. |
| **Criptografia & Segredos** | **AWS KMS + Secrets Manager** | Criptografia AES-256 gerenciada e injeção dinâmica de credenciais em tempo de execução (*zero hardcoding*). |

---

## 📚 Índice Geral da Documentação Técnica

O detalhamento técnico completo, dicionários de dados, esquemas relacionais e políticas de segurança estão organizados na pasta [`docs/`](docs/):

| Módulo | Documento | Descrição Técnica |
| :--- | :--- | :--- |
| **Data Architecture** | [`docs/data-architecture/data-architecture.md`](docs/data-architecture/data-architecture.md) | Mapeamento dos sistemas da plataforma, fluxo de dados ponta a ponta e arquitetura *Local-First*. |
| **Data Architecture** | [`docs/data-architecture/consolidated-architecture.md`](docs/data-architecture/consolidated-architecture.md) | Diagrama de Arquitetura Consolidada integrando *Interfaces, Segurança, Storage, OLTP e DW OLAP*. |
| **Data Architecture** | [`docs/data-architecture/business-value.md`](docs/data-architecture/business-value.md) | Síntese executiva demonstrando o valor de negócio e a defesa do isolamento OLTP vs. OLAP. |
| **Data Modeling** | [`docs/data-modeling/data-modeling.md`](docs/data-modeling/data-modeling.md) | Dicionário de dados, modelo relacional 3FN (PostgreSQL) e esquema estrela dimensional (Amazon Redshift). |
| **Data Modeling** | [`docs/data-modeling/modeling-business-value.md`](docs/data-modeling/modeling-business-value.md) | Impacto da modelagem relacional e dimensional na integridade clínica, KPIs gerenciais e conformidade com a LGPD. |
| **Data Storage** | [`docs/data-storage/data-storage.md`](docs/data-storage/data-storage.md) | Persistência poliglota 100% AWS, rotinas de backup (RDS PITR), metas de RPO/RTO e retenção no S3 Glacier. |
| **Data Storage** | [`docs/data-storage/physical-document-modeling.md`](docs/data-storage/physical-document-modeling.md) | Gestão, digitalização com OCR e custódia segura de prontuários em papel e acervos legados. |
| **Data Security** | [`docs/data-security/data-security.md`](docs/data-security/data-security.md) | Classificação LGPD, criptografia (AES-256 e TLS 1.3), matriz RBAC de acessos e mitigação de riscos. |
| **Data Integration** | [`docs/data-integration/data-integration.md`](docs/data-integration/data-integration.md) | Especificação de APIs REST/JSON, Webhooks assinados, integração com ICP-Brasil e tolerância a falhas. |

---

## ⚙️ Diretrizes Operacionais e SLAs de Resiliência

* **RPO (Recovery Point Objective):** $< 5$ minutos via arquivamento contínuo de WAL no Amazon RDS.
* **RTO (Recovery Time Objective):** $< 1$ hora para restabelecimento total da operação em caso de desastre.
* **Retenção Regulatória:** Prontuários e laudos armazenados por no mínimo **20 anos** via regras automáticas de transição para o S3 Glacier Deep Archive.
* **Privacidade:** Pseudonimização irreversível aplicada a qualquer dado transferido da base operacional para o Data Warehouse.
