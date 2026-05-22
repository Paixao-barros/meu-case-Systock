#Case Técnico: Analista de Integração de Dados

Bem-vindo ao repositório do meu case técnico de implantação do sistema **Systock**. 
Este projeto contém todo o processo de planejamento e ações realizadas na atividade.

---

## Organização do Projeto

Para facilitar a avaliação, o projeto foi dividido estritamente conforme as atividades solicitadas:

1. [**Atividade 1 – Documentação do Processo de Importação**](documentos/1_Importacao.md): Relatório detalhado descrevendo os erros intencionais mapeados nas planilhas e como corrigi.
2. [**Atividade 2 – Consultas SQL Básicas**](documentos/2_consultas_sql): Queries construídas para responder as dores iniciais de negócio.
3. [**Atividade 3 – Transformação de Dados**](documentos/3_Transformacao_dados.md): Scripts avançados usando (`HAVING`) e a lógica da **Trigger** automatizada.
4. [**Atividade 4 – Estratégia de Validação com o Cliente**](documentos/4_validacao_cliente.md): Roteiro descritivo com hipóteses de negócio, técnicas de precisão e scripts prontos para reunião.

---

## Backup do Banco de Dados

O script completo com a estrutura corrigida, chaves primárias, dados e a trigger funcional encontra-se na pasta `/backup`.
* [**Baixar Backup do Banco (.sql)**](backup/backup_systock_analisado.sql)

---

##Tecnologias Utilizadas
* **Banco de Dados:** PostgreSQL v15+
* **Ferramenta de Acesso:** DBeaver
* **Linguagem Procedural:** PL/pgSQL (para a Trigger)