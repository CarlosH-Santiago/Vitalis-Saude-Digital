
# **Entrega 1 — Data Architecture (Visão Macro)**
*Responsável: Carlos Henrique De Souza Santana Santiago*

## **1.A. Sistemas**
- Qual é o papel principal deste sistema na operação?
- Quem é o usuário direto dele (médico, paciente, equipe financeira, diretoria)?
- Ele é um sistema de entrada de dados, de processamento ou de análise?

* **Aplicativo Mobile/Web do Paciente**
  * **Papel:** Agendamento de consultas, acesso à sala virtual de telemedicina e consulta de receitas e exames.
  * **Usuário Direto:** Pacientes (B2C).
  * **Tipo:** Entrada e consulta de dados.

* **Sistema Administrativo (VitalisConnect)**
  * **Papel:** Gestão de agendas, preenchimento do Prontuário Eletrônico do Paciente (PEP) e emissão de prescrições.
  * **Usuário Direto:** Médicos, enfermeiros e recepcionistas (B2B).
  * **Tipo:** Entrada e processamento operacional.

* **Sistema de Pagamentos (Gateway)**
  * **Papel:** Processamento de pagamentos de consultas e cobrança das assinaturas das clínicas.
  * **Usuário Direto:** Pacientes, gestores de clínicas e equipe financeira.
  * **Tipo:** Processamento transacional financeiro.

* **CRM de Suporte**
  * **Papel:** Registro e acompanhamento de chamados técnicos e dúvidas operacionais.
  * **Usuário Direto:** Equipe de atendimento, clínicas parceiras e pacientes.
  * **Tipo:** Gestão de atendimento.

* **Banco Operacional (OLTP - PostgreSQL)**
  * **Papel:** Armazenamento relacional de dados transacionais com garantia de consistência e baixa latência.
  * **Usuário Direto:** Serviços de backend da aplicação e administradores de banco de dados.
  * **Tipo:** Armazenamento transacional (OLTP).

* **Data Warehouse (OLAP - Redshift)**
  * **Papel:** Armazenamento consolidado do histórico operacional para análise de dados e geração de relatórios.
  * **Usuário Direto:** Engenheiros de dados e analistas de BI.
  * **Tipo:** Armazenamento e processamento analítico (OLAP).

* **Sistema de BI (Metabase / Power BI)**
  * **Papel:** Exibição de painéis gerenciais e indicadores de desempenho da operação.
  * **Usuário Direto:** Diretoria executiva e gestores de clínicas.
  * **Tipo:** Visualização e análise de dados.

* **Servidor de Mídia (WebRTC)**
  * **Papel:** Transmissão de áudio e vídeo em tempo real durante as teleconsultas.
  * **Usuário Direto:** Médicos e pacientes em atendimento.
  * **Tipo:** Infraestrutura de comunicação em tempo real.

* **Repositório de Arquivos (AWS S3)**
  * **Papel:** Armazenamento de arquivos pesados, como exames em PDF, laudos e imagens no padrão DICOM.
  * **Usuário Direto:** Aplicação backend, médicos e pacientes.
  * **Tipo:** Armazenamento não estruturado (Object Storage).

## **1.B. Fluxo de Dados**
*Responsável: João Guilherme Pinheiro (CMO / Diretor Comercial)*

Na Vitalis, os dados começam a ser gerados quando o paciente usa o aplicativo para marcar uma consulta. Depois, passam pelo atendimento e podem ser usados pela diretoria para acompanhar os resultados da empresa.

A ideia é separar o que usamos na operação do que precisamos analisar. Assim, um relatório pesado não atrapalha o médico durante uma consulta.

O fluxo funciona assim:

### 1. Agendamento no aplicativo
O paciente entra no aplicativo (B2C), escolhe um horário e confirma a consulta. O backend registra o agendamento no banco operacional, chamado OLTP. Nesse momento, o sistema precisa garantir que o mesmo horário não seja reservado por dois pacientes.

### 2. Atendimento médico
O médico acessa seu painel (B2B), abre o Prontuário Eletrônico do Paciente (PEP) e inicia a teleconsulta. Durante o atendimento, registra a anamnese e emite a prescrição com assinatura ICP-Brasil. Esses registros ficam no banco operacional, que precisa responder rápido para não atrasar o trabalho.

### 3. Preparação dos dados
Depois, um processo de integração extrai os dados necessários para análise. Isso pode acontecer de madrugada ou em pequenos lotes ao longo do dia, dependendo da necessidade de atualização.

Nessa etapa, os dados são organizados e tratados. As informações sensíveis precisam de cuidados específicos, como anonimização ou pseudonimização, conforme o uso previsto e os requisitos de proteção de dados.

### 4. Armazenamento para análise
Os dados preparados seguem para o Data Warehouse da Vitalis. Nesse ambiente, podemos organizar os atendimentos em uma tabela de fatos, como Fato_Atendimento, ligada a dimensões como Dim_Tempo, Dim_Medico e Dim_Clinica. Isso facilita as consultas por período, médico ou clínica.

### 5. Painel da diretoria
Uma ferramenta de BI usa os dados do Data Warehouse para montar os painéis. A diretoria consegue acompanhar o volume de teleconsultas no mês, as especialidades mais procuradas e o tempo médio de atendimento.

A atualização desses painéis depende da frequência de carga dos dados. Se a carga acontecer só de madrugada, por exemplo, o painel vai mostrar os dados disponíveis até a última atualização.

## **1.C. Transacional vs. Analítico (OLTP e OLAP)**
*Responsável: João Guilherme Pinheiro (CMO / Diretor Comercial)*

Os dois ambientes têm funções diferentes. O OLTP atende às atividades do dia a dia. Já o OLAP permite analisar o histórico da operação.

### Banco transacional (OLTP)

É o banco usado pelo aplicativo e pelo painel do médico. Ele registra agendamentos, salva informações do prontuário e atende às solicitações que chegam durante o uso do sistema.

O foco é executar operações rápidas, mesmo com várias pessoas usando a plataforma ao mesmo tempo. Cada operação costuma envolver poucos dados, como o agendamento de uma consulta ou a atualização de um prontuário.

As propriedades ACID (Atomicidade, Consistência, Isolamento e Durabilidade) ajudam a manter as transações corretas. Uma gravação deve ser concluída por inteiro ou desfeita, evitando que uma falha deixe parte da informação salva.

### Ambiente analítico (OLAP / Data Warehouse)

Esse ambiente serve para consultar grandes volumes de dados e acompanhar o histórico da Vitalis. Por exemplo, podemos comparar o número de consultas entre meses ou calcular o tempo médio de atendimento por especialidade.

A estrutura é preparada para somas, médias, contagens e cruzamentos de informações. O Data Warehouse reúne os dados usados nessas análises, enquanto o OLAP se refere ao processamento analítico.

Como esse trabalho acontece em um ambiente separado, os relatórios não disputam os mesmos recursos do banco usado no atendimento.

### O que acontece se a diretoria rodar um relatório pesado no banco usado pelo médico?

O atendimento pode ficar mais lento, porque o relatório passa a disputar recursos com as operações do médico.

Imagine que a diretoria queira calcular a média de duração das consultas de todas as clínicas nos últimos cinco anos, agrupada por mês e especialidade. Para gerar esse relatório, o banco pode precisar ler milhões de registros e fazer vários cálculos.

Enquanto isso, o médico está tentando salvar uma evolução no prontuário. Essa gravação é pequena, mas pode demorar se o banco estiver sobrecarregado ou se houver bloqueios entre as consultas. Na prática, o médico pode ficar esperando a tela responder, e a emissão da prescrição também pode atrasar.

O impacto depende da capacidade do banco e de como as consultas são executadas. Mesmo assim, não faz sentido colocar o atendimento nessa disputa.

Por isso, a proposta para a Vitalis é manter os ambientes separados: o OLTP cuida dos agendamentos e dos registros clínicos, e o ambiente analítico fica com os relatórios da diretoria. Assim, conseguimos analisar os resultados sem prejudicar a rotina de atendimento.
