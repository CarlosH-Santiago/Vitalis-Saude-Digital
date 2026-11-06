# Arquitetura de Dados Corporativa

## 1. Diretrizes de Arquitetura

A arquitetura da **Vitalis Saúde Digital** sustenta serviços críticos de telemedicina e Prontuário Eletrônico do Paciente (PEP) operando simultaneamente em modelos B2B e B2C. Três decisões técnicas guiam o ecossistema:

* **Baixa latência clínica:** A consulta e a escrita do prontuário exigem tempo de resposta imediato, sem contenção de recursos computacionais.
* **Conformidade regulatória:** Isolamento de dados clínicos sensíveis (LGPD, Art. 11) e retenção legal obrigatória de 20 anos para prontuários (Lei nº 13.787/2018 e CFM).
* **Isolamento de cargas:** Segregação estrita entre o processamento transacional de atendimento (OLTP) e o processamento analítico de inteligência de negócios (OLAP).

---

## 2. Ecossistema Tecnológico

Cada tecnologia atende a uma demanda computacional específica, evitando centralização excessiva e gargalos de I/O:

| Componente | Tecnologia | Papel Prático na Operação |
| :--- | :--- | :--- |
| **App do Paciente (B2C)** | Web / Mobile | Autoagendamento, sala de espera virtual e acesso a documentos. |
| **Portal Médico (B2B)** | Web SaaS | Anamnese, histórico clínico e prescrições com assinatura digital ICP-Brasil. |
| **Banco Operacional (OLTP)** | PostgreSQL RDS Multi-AZ | Persistência relacional ACID de baixa latência para prontuários, cadastros e agendas. |
| **Armazenamento de Arquivos** | AWS S3 | Guarda de objetos pesados e estáticos (laudos em PDF e imagens médicas DICOM). |
| **Cache em Memória** | Redis | Gestão de sessões WebRTC, tokens de autenticação e trava temporária de agenda contra duplo agendamento. |
| **Servidores de Mídia** | WebRTC dedicado | Streaming de áudio e vídeo da teleconsulta ponto a ponto, sem sobrecarregar a base de dados. |
| **Integração & Pipeline** | Airflow + CDC (Debezium) | Captura de alterações do banco relacional com sanitização e mascaramento prévio de dados sensíveis. |
| **Data Warehouse (OLAP)** | Amazon Redshift | Banco colunar com modelagem dimensional (*Star Schema*) para consultas analíticas corporativas. |
| **Camada de BI** | Metabase / Power BI | Painéis executivos com indicadores de ocupação, faturamento SaaS e tempos médios de consulta. |

---

## 3. Fluxo de Dados

O dado percorre um ciclo unificado desde a entrada na interface até a camada analítica:

```
[Paciente / Médico] 
       │ (HTTPS / TLS 1.3)
       ▼
[API Gateway / Backend] ──► [Redis] (Controle de sessão e trava de horário)
       │
       ├──► [PostgreSQL RDS] (Registros clínicos ACID)
       └──► [AWS S3] (Armazenamento de PDFs e exames pesados)
                 │
                 ▼ (Carga incremental diária via CDC / Airflow)
       [Amazon Redshift DW]
                 │
                 ▼
       [Dashboards de BI]
```

1. **Ingestão e Agendamento:** O paciente agenda a consulta; a aplicação valida a disponibilidade no Redis e persiste o agendamento no PostgreSQL RDS.
2. **Atendimento Clínico:** A teleconsulta ocorre via WebRTC. O médico digita anotações clínicas diretamente no prontuário e assina receitas digitais; laudos e exames anexados são enviados ao AWS S3 via URLs pré-assinadas.
3. **Extração e Tratamento:** Cargas incrementais via Change Data Capture (CDC) extraem alterações da base operacional sem bloqueios de tabela, aplicando anonimização nos dados protegidos pela LGPD.
4. **Carga e Visualização:** O Amazon Redshift consolida os dados na tabela `Fato_Atendimento` associada a dimensões temporais e médicas, alimentando os relatórios da diretoria sem onerar o atendimento médico.

---

## 4. Resiliência: Modo Offline

Para evitar interrupção no atendimento médico em caso de queda de conexão na clínica, o Portal VitalisConnect adota arquitetura *Local-First* com sincronização assíncrona:

* **Persistência Local:** O rascunho da anamnese e itens da prescrição são gravados em tempo real no banco local do navegador (IndexedDB) com criptografia de sessão.
* **Fila em Background:** A aplicação gera identificadores globais (`UUIDv4`) para cada transação e retém as ações em fila local gerenciada por um *Service Worker*.
* **Sincronização Idempotente:** Assim que a conexão HTTPS é restabelecida, o *Service Worker* despacha a fila para a API REST, que utiliza o `UUID` para descartar duplicidades e consolida os registros no PostgreSQL central.

---

