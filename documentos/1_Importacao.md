# Documentação do Processo de importação de dados (Cliente > Systock)
Processo de carga de dados realizados para implantação do sistema Systock, com base nos arquivos brutos fornecidos pelo cliente.

---

## 1. Correção das estrutura e erros ortográficos nas tabelas descritas.
Antes de iniciar a criação das tabelas no banco, foram identificadas algumas correções que deveriam ser feitas, correções estruturias, de concordância de chaves e de digitação que impediriam a criação da estrutura no momento.

### Tabela: *public.entradas_mercadoria*
* **Ajuste:** Durante a criação, o banco apresentou erro informando que a coluna `ordem_compra` não havia sido declarada
* **Ação:** Foi adicionada a coluna, com o type float8 NOT NULL visando a mesma estar sendo definico como Primary Key

### Tabela: *public.produtos_filial*
* **Erro de Sintaxe:** Durante a criação, foi notado que  na coluna IDFONECEDOR além de estar incorreta (IDFORNECEDOR) tabmém não contava com a vírgula, sepração obrigatória no final da linha.
* **Chave Primária:** A coluna criada foi `produto_id`, mas a restrinção da chave primária referenciava um campo chamadao `idproduto`.
Para mannter a pradronização, foi refrenciada na primary key `produto_id` visando a concordancia com a tabela de Entradas_mercadoria.
* **Segurança:** A coluna `filial-id` estava definida apenas como `NULL`, o que em significa que em alguns cénarios passar parâmetros vazios para insert/updates causaria erros. A coluna então foi alterada para `NOT NULL` previnindo possiveis erros futuros.
* **Ortográfica:**  o campo originalmente foi mapeado como `decricao`, que ocasiono ter que executar o comando de renomear a coluna para o nome correto:
        ``` sql
        ALTER TABLE public.produtos_filial RENAME COLUMUN decricao TO descricao;
        ```

### Tabela: public.fornecedor
* **Correção Ortográfica:** A coluna estava descrita como `idforncedor` e foi ajustada para `fornecedor_id` mantendo os padrões de produto_id e filial_id.

## 2. Processo de importação de dados.
A importação dos dados para o ambiente foram realizadas pelo assistente de importação do DBeaver, as seguintes alterações foram necessárias:

### Tabela: public.fornecedor
* **Tratamento do arquivo** O assistente de DBeaver utiliza a vírgula como separador padrão de arquivos csv, no entando o arquivo enviado pelo cliente continha o ponto e vírgula como separador, foi necessário realizar o ajuste na configuração do painel do DBeaver para realizar a importação.
* **Tipo de dados:** Na planilha do cliente, continha valores como "F*", mas a coluna estava definiada com o TYPE (`int4`), foi necessário alterar o tipo da coluna para suporte texto `Varchar(25)` para garantir o funcionamento das consultas.
        ```sql
        ALTER TABLE public.fornecedor ALTER COLUMN fornecedor_id TYPE varchar(25);
        ´´´

### Tabela: public.produtos_filial
* **Chave estrangeira:** A importação inicial falhou pois a planilha continha o código de texto "F8" para fornecedor-id, mas a coluna no banco estava tipada como numérica, para não barrar o processo da carga dos dados, a coluna foi ajustada temporariamente para `varchar(25)` para herdar os dados do cliente.
* **Divergências:** No arquivo excel a coluna identificadora do produto estava descrita como `idproduto`, enquanto a tabela do banco esperava `produto_id`, acontece que pelo fato desse campo ser uma chave primária, optei por alterar na planilha e assim seguir com a importação.

### Tabela: public.pedido_compra
* **Padronização de Chaves e Nomes:** Na tabela de fornecedores tinhamos `idfornecedor`, mas na tabela de pedidos estava mapeado como `fornecedor_id`. Para manter o padrão do projeto (seguindo o exemplo de `produto_id` e `filial_id`), os nomes de colunas e suas chaves primárias foram ajustados.
* **Formato de Datas:** O arquivo bruto continha datas no padrão brasileiro. O PostgreSQL exige o padrão internacional (`YYYY-MM-DD`). O formato foi convertido diretamente nas células da planilha antes do envio para evitar rejeição do banco.
* **Correção de Dados ('F'):** No arquivo do cliente, a coluna do fornecedor continha apenas o número limpo (`8`), mas no cadastro de fornecedores o código era uma string contendo a letra `F8`. Se deixarmos assim, os futuros `JOINs` de relatórios resultariam em branco. Por isso, removemos o valor padrão da coluna, alteramos seu tipo para texto e aplicamos uma query de atualização  para concatenar o prefixo ´F´:
    ```sql
    ALTER TABLE public.pedido_compra ALTER COLUMN fornecedor_id DROP DEFAULT;
    ALTER TABLE public.pedido_compra ALTER COLUMN fornecedor_id TYPE varchar(25);

    UPDATE public.pedido_compra 
    SET fornecedor_id = 'F' || fornecedor_id
    WHERE fornecedor_id NOT LIKE 'F%';
    ```

### Tabelas: public.entradas_mercadoria e public.venda
* Ambas as estruturas e arquivos de dados estavam certos e foram importados com sucesso pelo DBeaver sem a necessidade de alterações ou ajustes.

---

## 3. Qualidade da Base Histórica

Durante a análise visual das linhas nas planilhas, foram aplicados dois critérios de governança de dados para garantir a qualidade do ambiente de produção do Systock:

1.  **Eliminação de Linhas Corrompidas:** Na tabela de pedidos de compras, o arquivo original apresentava registros com informações totalmente fora de suas colunas correspondentes. Como essas linhas continham dados incompletos e corrompidos, elas foram excluídas na origem, garantindo que apenas registros certos entrassem no banco.
2.  **Ferramentas Manuais de Transição:** As correções de layout, cabeçalhos e máscaras de data foram feitas no Excel. A carga foi consolidada manualmente no DBeaver, gerando ao final um ambiente limpo, mapeado e pronto para a próxima etapa do projeto.