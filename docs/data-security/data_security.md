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
