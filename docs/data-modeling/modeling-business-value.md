# 2.4.4 Alinhamento Estratégico e Valor de Negócio

A arquitetura de dados da Vitalis Saúde Digital foi pensada para manter os atendimentos funcionando, facilitar as decisões da diretoria e proteger as informações dos pacientes. Essas medidas são importantes porque a empresa trabalha com dados de saúde, que exigem cuidado e controle em todas as etapas.

### 1. Como a arquitetura evita travamentos no atendimento médico

A separação entre o banco operacional (OLTP) e o ambiente analítico (OLAP) permite que cada um tenha uma função específica. O PostgreSQL, utilizado no ambiente operacional, registra agendamentos, prontuários e prescrições, priorizando respostas rápidas e a consistência das informações.

Já os relatórios e cálculos mais pesados ficam no Data Warehouse, utilizando o Amazon Redshift. Assim, quando a diretoria precisa analisar dados de vários anos, essa consulta não disputa diretamente os recursos do banco utilizado pelos médicos. Isso reduz o risco de lentidão e ajuda a manter a continuidade dos atendimentos.

### 2. Como a modelagem analítica ajuda nas decisões estratégicas

O Data Warehouse reúne dados históricos da operação em uma estrutura preparada para análises. Com o apoio das ferramentas de BI, como Metabase e Power BI, a diretoria pode acompanhar indicadores, comparar períodos, identificar as especialidades mais procuradas e avaliar o volume de consultas e o tempo médio de atendimento.

Essas informações ajudam a identificar problemas, planejar melhorias e direcionar investimentos com mais segurança. Em vez de depender apenas de impressões, os gestores passam a tomar decisões com base em dados organizados. A frequência de atualização dos relatórios também deve ser considerada, pois os indicadores refletem os dados disponíveis até a última carga realizada.

### 3. Como a gestão de dados reduz riscos de multas e vazamentos

A proteção das informações precisa abranger tanto os ambientes digitais quanto os documentos físicos. No sistema digital, a Vitalis prevê criptografia, controle de acesso por perfil, autenticação em múltiplos fatores, backups e registros de auditoria. Essas medidas ajudam a impedir acessos indevidos, recuperar informações em caso de falhas e identificar ações realizadas no sistema.

Os documentos físicos, como prontuários e termos assinados, devem permanecer em local restrito, com acesso controlado e registro de retirada e devolução. A digitalização e a indexação também facilitam a localização dos documentos e reduzem o risco de perda.

Esses cuidados contribuem para o cumprimento da LGPD, das regras de sigilo profissional e das normas aplicáveis do CFM. Com controles adequados, a empresa reduz a possibilidade de vazamentos, perda de informações e problemas legais que podem gerar.