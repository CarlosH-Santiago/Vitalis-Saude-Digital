# Impacto da Modelagem de Dados no Negócio

## 1. Confiabilidade Clínica e Integridade Transacional (OLTP em 3FN)

A escolha da 3ª Forma Normal (3FN) no **AWS RDS PostgreSQL** foi orientada diretamente às exigências operacionais e regulatórias da clínica médica:

* **Prevenção de Duplo Agendamento (*Overbooking*):** A confirmação de consulta, a reserva do slot de horário e a transação financeira ocorrem em transação atômica (ACID). As restrições de unicidade no PostgreSQL e travas atômicas no Redis (`SET NX`) garantem que dois pacientes nunca reservem o mesmo médico no mesmo minuto.
* **Segurança Jurídica e Imutabilidade do Prontuário:** O relacionamento 1:1 entre `CONSULTA` e `PRONTUARIO` é selado após a finalização do atendimento. Alterações posteriores são bloqueadas no banco; retificações clínicas exigem adendos auditáveis, protegendo o médico e a clínica em perícias ético-legais perante o Conselho Federal de Medicina (CFM).
* **Segregação de Dados para RBAC Granular:** A separação física entre `PACIENTE` (dados cadastrais) e `PRONTUARIO` (dados clínicos) permite que perfis administrativos da recepção e faturamento acessem apenas telefones e planos, sem qualquer visibilidade sobre sintomas, hipóteses diagnósticas (CID-10) ou receitas médicas.

---

## 2. Visibilidade Gerencial e Eficiência Operacional (OLAP Star Schema)

A modelagem dimensional em **Esquema Estrela** no **Amazon Redshift** transforma os dados brutos da operação em métricas acionáveis para a diretoria executiva e gestores de clínicas parceiras:

| Indicador Crítico (KPI) | Fonte no Data Warehouse | Aplicação Prática no Negócio |
| :--- | :--- | :--- |
| **Taxa de Absenteísmo (*No-Show*)** | `FATO_ATENDIMENTO_ANALITICO.fl_atendimento_concluido` | Identificação de turnos e dias com maior índice de faltas, permitindo disparar lembretes automatizados via WhatsApp e otimizar horários de teleconsulta. |
| **Tempo Médio de Espera e Atendimento** | `tempo_espera_min` e `duracao_atendimento_min` | Avaliação da pontualidade clínica e calibragem do tempo padrão de duração de slots por especialidade médica. |
| **Rentabilidade por Operadora vs. Particular** | `DIM_PLANO_SAUDE.categoria_faturamento` e `valor_consulta` | Análise de margem líquida para apoiar negociações contratuais de reajuste de tabelas com planos de saúde parceiros. |
| **Sazonalidade e Demanda Regional** | `DIM_TEMPO` e `DIM_PACIENTE.regiao_brasil` | Dimensionamento de escalas de plantonistas de telemedicina de acordo com picos epidemiológicos sazonais por macrorregião. |
| **Volume de Prescrições Emitidas** | `qtd_prescricoes_emitidas` | Monitoramento da adesão à ferramenta de assinatura digital ICP-Brasil e controle de custos de certificação. |

### Por que o Esquema Estrela traz retorno financeiro?
Em vez de sobrecarregar o banco de produção com consultas analíticas pesadas com múltiplos *JOINs*, o Redshift processa dados desnormalizados em formato colunar. Um relatório de 3 anos de histórico é executado em menos de 2 segundos, sem demandar mais hardware e sem gerar lentidão nas telas dos médicos.

---

## 3. Governança LGPD e Mitigação de Passivos Legais

A modelagem de dados foi desenhada sob o princípio de *Privacy by Design*:

1. **Pseudonimização Automática no Data Warehouse:** A dimensão `DIM_PACIENTE` armazena exclusivamente o campo `id_paciente_anonimizado` (hash HMAC SHA-256). Analistas de negócios e ferramentas de BI geram relatórios estatísticos e epidemiológicos completos sem ter acesso a CPF ou nomes reais de pacientes.
2. **Harmonização LGPD vs. Lei do Prontuário:** Em caso de pedido de exclusão formulado pelo paciente (Art. 18 da LGPD), o cadastro em `PACIENTE` é desativado (*soft delete* com `fl_ativo = false`), enquanto o registro em `PRONTUARIO` é anonimizado irreversivelmente e preservado pelo prazo compulsório de 20 anos estabelecido pelo Art. 6º da Lei nº 13.787/2018.
3. **Auditabilidade Irrefutável (`LOG_ACESSO`):** A estrutura *append-only* (sem comandos `UPDATE` ou `DELETE`) garante que qualquer visualização ou download de prontuário fique registrado com IP, usuário e data/hora exatos, fornecendo prova técnica imediata em eventuais fiscalizações da ANPD.
