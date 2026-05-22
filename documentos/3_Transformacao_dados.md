## 3. Transformação de Dados e Frequência de Requisições

Para atender a essas demandas, desenvolvi duas consultas estruturadas (uma para Vendas e outra para Pedidos de Compra). O objetivo aqui foi aplicar uma regra de negócio baseada na **Quantidade de requisições**.

Em vez de somar o volume físico total dos itens (o que seria feito através da coluna `qtde_pedida`), a lógica aplicada conta quantas vezes um determinado produto apareceu nas transações (`COUNT`) através de ID únicos. Isso nos permite medir a recorrência real de saída ou solicitação de um item em uma determinada faixa de tempo.

### Consulta para a Tabela de Vendas
```sql
SELECT 
    CONCAT(v.produto_id, ' - ', pf.descricao) AS PRODUTO,
    COUNT(v.venda_id) AS QTDE_REQUISITADO,
    TO_CHAR(MAX(v.data_emissao), 'DD/MM/YYYY') AS DATA_ULTIMA_VENDA_FORMAT
FROM public.venda v 
JOIN public.produtos_filial pf ON (v.produto_id = pf.produto_id)
WHERE v.data_emissao BETWEEN '2025-01-01' AND '2025-03-31' 
GROUP BY v.produto_id, pf.descricao
HAVING COUNT(v.venda_id) > 10
ORDER BY data_ultima_venda_format  DESC;
```

### Consulta para a Tabela de Pedido de Compra
```sql
SELECT 
    CONCAT(pc.produto_id, '-', pf.descricao) AS PRODUTO,
    COUNT(pc.produto_id) AS Qt_REQUISITADA,
    TO_CHAR(MAX(pc.data_pedido), 'DD/MM/YYYY') AS DATA_SOLICITACAO
FROM public.pedido_compra pc 
JOIN public.produtos_filial pf ON (pc.produto_id = pf.produto_id)
WHERE pc.data_pedido BETWEEN '2025-02-01' AND '2025-02-28'
GROUP BY pc.produto_id, pf.descricao 
HAVING COUNT(pc.produto_id) > 10 
ORDER BY data_solicitacao DESC;
```
* Mantem a solicitação de concatenar produto_id + descicao e convertendo o formato da data para DD/MM/YYYY e retornando produtos com mais de 10 requisições no período.
---

## 4. Normalização de IDs e Criação da Trigger Automática

O escopo do esperado exigia a geração de um novo ID numérico para os fornecedores. No entanto, os dados importados da planilha utilizavam um formato de texto (`F8`). Para resolver esse conflito de TYPE sem perder o histórico, a estratégia adotada foi criar um sistema de "de-para" automatizado usando uma Trigger.

A lógica é as seguinte: quando a planilha injetar um dado como `F8`, a Trigger vai interceptar a inserção, consultar a tabela de fornecedores, verificar que o `F8` agora corresponde ao ID numérico `8`, e gravar apenas o número inteiro no banco.

### Passo 4.1: Preparação da Tabela de Fornecedores
Primeiro, foi criada uma nova coluna na tabela de fornecedores para receber o identificador numérico.

```sql
ALTER TABLE public.fornecedor ADD COLUMN IF NOT EXISTS fornecedor_id_num int4;
```

Em seguida, criei uma *Sequence* para garantir que a geração desses números seja sequencial e padronizada, atualizando os fornecedores já existentes na base:

```sql
CREATE SEQUENCE IF NOT EXISTS public.seque_fornecedor_id START WITH 1;

UPDATE public.fornecedor
SET fornecedor_id_num = nextval('public.seque_fornecedor_id')
WHERE fornecedor_id_num IS NULL;
```

### Passo 4.2: Tabela de Produtos
Havia um obstáculo técnico na tabela de produtos: a coluna `fornecedor_id` precisava ser convertida para `int4` (inteiro), mas como ela já estava preenchida com strings da importação (`F8`), o banco bloqueava a alteração de tipo.

A solução foi criar uma coluna temporária de transição (`idfornecedor_temp`). Os dados textuais antigos foram movidos para ela, esvaziando a coluna principal. Com a coluna principal limpa, foi possível alterar seu tipo para `int4` com segurança. A coluna temporária passa a servir como porta para os dados da planilha, enquanto a principal recebe o número processado pela Trigger.

```sql
UPDATE public.produtos_filial SET idfornecedor_temp = fornecedor_id;
UPDATE public.produtos_filial SET fornecedor_id = NULL;
ALTER TABLE public.produtos_filial ALTER COLUMN fornecedor_id TYPE int4;
```

### Passo 4.3: Criação da Função e da Trigger
Por fim, a Trigger e o  gatilho foram criadas para fazer o cruzamento dos dados antes de qualquer nova inserção. Ela lê o valor temporário em texto, busca o número correspondente e injeta no campo correto.

```sql
CREATE OR REPLACE FUNCTION public.func_gerar_id_prod()
RETURNS TRIGGER AS $$
DECLARE
    v_id_numerico int4;
BEGIN
    SELECT fornecedor_id_num INTO v_id_numerico
    FROM public.fornecedor
    WHERE fornecedor_id = NEW.idfornecedor_temp;
    
    IF v_id_numerico IS NOT NULL THEN
        NEW.fornecedor_id := v_id_numerico;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```
* **Gatilho:**
```sql
CREATE OR REPLACE TRIGGER trg_idfornecedor_num
BEFORE INSERT ON public.produtos_filial
FOR EACH ROW
EXECUTE FUNCTION public.func_gerar_id_prod();
```