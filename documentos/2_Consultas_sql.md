## Parte 2 - Consultas SQL básicas:

A consulta que traz o total de vendas:

```sql
SELECT
	v.produto_id,
	v.data_emissao,
	SUM(trunc(v.qtde_vendida)) AS Quantidade_vendida,
	v.valor_unitario AS valor_uni,
	SUM(trunc(v.qtde_vendida) * v.valor_unitario) AS Valor_Real
FROM public.venda v 
WHERE v.data_emissao BETWEEN '2025-02-01' AND '2025-02-28'
GROUP BY v.produto_id, v.data_emissao, v.valor_unitario 
ORDER BY Valor_Real DESC;
```

* Usei o `trunc` pra quebrar o valor deixando inteiro, visto que a unidade de medida de todos os produtos são Unidades. O valor em real não é uma tabela e sim um cálculo de quantidade vendida * valor unitário e ordenei o valor como sendo o mais vendido no topo.

---

Produtos requisitados que não foram recebidos:

```sql
SELECT 
	pf.descricao,
	pc.produto_id,
	SUM(pc.qtde_pedida) AS QT_PEDIDO,
	COALESCE(SUM(em.qtde_recebida), 0) AS Total_recebido,
	(SUM(pc.qtde_pedida) - COALESCE(SUM(em.qtde_recebida), 0)) AS NAO_RECEBIDO
FROM public.pedido_compra pc 
LEFT JOIN public.entradas_mercadoria em ON (pc.ordem_compra = em.ordem_compra) AND (pc.produto_id = em.produto_id)
LEFT JOIN public.produtos_filial pf ON (pc.produto_id = pf.produto_id)
GROUP BY pc.produto_id, pf.descricao 
HAVING SUM(pc.qtde_pedida) > COALESCE(SUM(em.qtde_recebida), 0)
ORDER BY NAO_RECEBIDO DESC;
```

* Usei o `LEFT JOIN` para garantir que linhas duplicadas não se somassem e que produtos que não foram entregues sumam, assim eles ficam com valor 0.
* Mantive regra de negócio onde um pedido de compra é vinculado à entrada da mercadoria pela coluna `ordem_compra` e `produto_id` para ter uma trava caso o pedido de uma ordem cruze com a entrada de outra ordem.