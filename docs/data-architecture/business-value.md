## *. Segregação OLTP vs. OLAP

Separar a base transacional da base analítica previne indisponibilidade operacional e degradação no atendimento médico:

Se a diretoria executasse consultas pesadas diretamente no banco PostgreSQL RDS, o SGBD realizaria varreduras completas em disco (*table scans*), travando tabelas e saturando memória e conexões. Durante esse período, os médicos enfrentariam lentidão ou *timeouts* na gravação de prontuários e emissão de prescrições.

| Critério | PostgreSQL RDS (OLTP) | Amazon Redshift (OLAP) |
| :--- | :--- | :--- |
| **Foco Operacional** | Sustentação do atendimento e escrita em tempo real. | Decisões estratégicas, projeções e inteligência clínica. |
| **Perfil de Acesso** | Escritas rápidas (`INSERT`, `UPDATE`) e leituras pontuais por chave. | Varredura de milhões de linhas e agregações analíticas complexas. |
| **Modelagem** | Relacional normalizado em 3FN. | Dimensional (*Star Schema*) em armazenamento colunar. |
| **Tempo de Resposta** | Menor que 100 ms. | Segundos a minutos, sobre dados históricos consolidados. |
| **Sincronização** | Instantânea (ACID). | Cargas em lote (*batch*) diárias agendadas. |