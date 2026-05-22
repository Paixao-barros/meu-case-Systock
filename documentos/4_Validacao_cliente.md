## Etapa 4: Estratégia e Validação com o Cliente

Para a reunião de validação dos dados referentes a Fevereiro de 2025, o objetivo não é apenas apresentar tabelas, mas garantir que a base de dados reflita com exatidão a realidade da operação. Estruturei a validação em três pilares principais, focando em regras de negócio, integridade técnica e auditoria de dados.

### 1. Principais Pontos de Validação (Regras de Negócio)

* **Tratamento de Entregas Parciais:** Nossa regra atual para identificar "produtos pendentes" assume que qualquer pedido com quantidade solicitada maior que a entregue gera uma pendência. No entanto, é necessário validar como a operação do cliente lida com entregas parciais: o saldo restante deve manter a ordem de compra em aberto (como um pedido pendente) ou a ordem é encerrada? Essa definição é crítica, pois impacta diretamente os KPIs dos fornecedores.
* **Inconsistência de Unidades de Medida:** Identificamos na tabela de vendas tem registros com quantidades fracionadas (38,6), embora a unidade de medida padrão (UN) exija números inteiros. Precisamos alinhar se o negócio permite vendas fracionadas (venda por peso registrada como unidade) ou se houve erro na captura dos dados, o que exigiria o arredondamento (uso de `TRUNC` ou `ROUND`) para garantir a integridade dos relatórios.

### 2. Técnicas para Garantir Exatidão e Precisão

Para atestar a confiabilidade da carga de dados de Fevereiro, utilizaremos duas frentes de auditoria:
* **Linha a Linha:** Faremos uma amostragem extraindo cenários muito específicos no banco (pedidos de fevereiro com quantidade menor que 10). Ao isolar esse cenário, o banco retornará um número reduzido de linhas. Em seguida, aplicaremos o mesmo filtro na planilha original do Excel. Se o resultado bater linha a linha, provamos a exatidão da importação e da lógica de filtragem.
* **Total Financeiro:** Para assegurar a precisão de valores, utilizaremos funções de agregação (`SUM`) no banco de dados para somar o faturamento total e o custo total do mês, cruzando esse valor exato com o somatório da planilha. Em validações que envolvem fluxo de caixa, a totalização financeira é a prova real.

### 3. Consultas Estratégicas Preparadas para a Reunião

Deixarei consultas focadas em auditoria (dados fantasmas) para demonstrarmos o nível aplicado aos dados.

**A. Pedidos com Baixa Requisição em Fevereiro:**
Utilizada para o cruzamento "linha a linha" com o Excel e para investigar o motivo do item ser pouco requisitado.
```sql
SELECT 
    pc.pedido_id,
    pc.ordem_compra,
    pc.produto_id,
    pc.qtde_pedida,
    pc.data_pedido
FROM public.pedido_compra pc
WHERE pc.qtde_pedida < 10
  AND pc.data_pedido BETWEEN '2025-02-01' AND '2025-02-28'
ORDER BY pc.data_pedido ASC;
```

**B. Inconsistência de Cadastro:**
Identificar produtos que constam na base, mas que perderam o vínculo com a tabela de fornecedores, diminuindo a assertividade da operação.
```sql
SELECT 
    p.produto_id,
    p.descricao AS descricao,
    p.fornecedor_id AS id_produto_fornecedor,
    f.fornecedor_id AS id_na_tabela_fornecedor
FROM public.produtos_filial p
LEFT JOIN public.fornecedor f ON p.fornecedor_id = f.fornecedor_id_num
WHERE f.fornecedor_id_num IS NULL 
  AND p.fornecedor_id IS NOT NULL;
```

**C. Auditoria de Vendas Fantasmas:**
Mapeia o histórico de vendas para encontrar ocorrências de transações registradas com IDs de produtos que simplesmente não existem no cadastro do sistema.
```sql
SELECT 
    v.venda_id,
    v.data_emissao,
    v.produto_id AS produto_na_venda,
    p.produto_id AS produto_no_cadastro
FROM public.venda v
LEFT JOIN public.produtos_filial p ON v.produto_id = p.produto_id
WHERE p.produto_id IS NULL;
```