# 🔪 JSON_TABLE no MySQL 8

A partir da versão 8.0, a table function JSON_TABLE permite transformar JSON em tabelas SQL

## Exemplos

### 💀 Números e Letras
Transformando um array simples de números ou letras em tabela:
```sql
SELECT * FROM JSON_TABLE(
    '[1,2,3]', # json file 
    '$[*]' # referencia ao próprio json file
    COLUMNS (
        XPTO VARCHAR(100) PATH '$' #referencia ao proprio elemento
    )
) AS tbl 

# exemplo simples 2
SELECT * FROM JSON_TABLE(
    '["A","B","C"]', # json file 
    '$[*]' # referencia ao próprio json file
    COLUMNS (
        XPTO VARCHAR(100) PATH '$' #referencia ao proprio elemento
    )
) AS tbl 

```
### 💀 Exemplo com Produtos
Neste exemplo, temos um JSON com dados de produtos.
Ao utilizar a JSON_TABLE, conseguimos transpor esses dados para uma tabela SQL e assim podemos consultar ou filtrar.

```sql
CREATE TABLE t1 (json_col JSON);

INSERT INTO t1 VALUES (
    '{ "products": [
        { "code":"P001", "name":"Teclado Mecânico", "price": 350.90 },
        { "code":"P002", "name":"Mouse Gamer",      "price": 180.00 },
        { "code":"P003", "name":"Monitor 24 Pol",   "price": 1299.99 }
    ] }'
);

SELECT tbl.*
FROM t1
JOIN JSON_TABLE(
    json_col,                 # json principal
    '$.products[*]'           # array de produtos
    COLUMNS (
        product_code VARCHAR(10)  PATH '$.code',   # campo do produto
        product_name VARCHAR(100) PATH '$.name',
        price         DECIMAL(10,2) PATH '$.price'
    )
) AS tbl;

/*
# tbl é uma tabela virtual, então aceita filtros normalmente

WHERE tbl.product_name LIKE 'Mouse%'
OR tbl.price > 500;
*/

```

### 💀 Exemplo com Pedidos Aninhados
Aqui temos um JSON mais complexo, com um array de pedidos e cada pedido contendo itens (JSON aninhado).

```sql
CREATE TABLE t2 (json_col JSON);

INSERT INTO t2
VALUES (
'[
  {
    "order_id": "ORD001",
    "customer": "Cliente A",
    "order_date": "2024-10-15",
    "items": [
      { "sku": "P100", "name": "Teclado", "qty": 1, "price": 350.90 },
      { "sku": "P200", "name": "Mouse",   "qty": 2, "price": 180.00 }
    ]
  },
  {
    "order_id": "ORD002",
    "customer": "Cliente B",
    "items": [
      { "sku": "P300", "name": "Monitor", "qty": 1, "price": 1299.99 },
      { "sku": "P400", "name": "Headset", "qty": 1, "price": 499.90 },
      { "sku": "P500", "name": "Webcam",  "qty": 1, "price": 320.00 }
    ]
  }
]'
);

SELECT * FROM t2;

SELECT pedidos.*
FROM t2
JOIN JSON_TABLE(
    json_col,
    '$[*]'
    COLUMNS (
        id_pedido FOR ORDINALITY,
        codigo_pedido VARCHAR(10) PATH '$.order_id',
        cliente        VARCHAR(20) PATH '$.customer',
        possui_data    INT EXISTS PATH '$.order_date', -- 1 se existir, 0 se não
        NESTED PATH '$.items[*]' COLUMNS (
            id_item FOR ORDINALITY,
            sku      VARCHAR(10)  PATH '$.sku',
            produto  VARCHAR(50)  PATH '$.name',
            quantidade INT        PATH '$.qty',
            preco     DECIMAL(10,2) PATH '$.price'
        )
    )
) AS pedidos;

/*
-- pedidos é uma tabela virtual, então aceita filtros:
WHERE pedidos.preco > 500
AND pedidos.cliente = 'Cliente B';
*/

```
### 🪓 Exemplo do dia a dia: normalizando planilhas
Recebi uma planilha, importei para o banco, e precisava transformar uma coluna com vários códigos em várias linhas, repetindo as informações das outras colunas.
Utilizei a JSON_TABLE para facilitar o processo

```sql
-- Criação da tabela
CREATE TABLE TBL_XPTO (
    COL_A VARCHAR(10),
    COL_B VARCHAR(50),
    COL_C VARCHAR(255)
);

-- Inserção dos 10 registros aleatórios
INSERT INTO TBL_XPTO (COL_A, COL_B, COL_C) VALUES
('00123401', 'AAA-DESCRICAO PRODUTO A', '"100001","100002","100003","100004","100005"'),
('00123402', 'AAB-DESCRICAO PRODUTO B', '"200001","200002","200003","200004","200005"'),
('00123403', 'AAC-DESCRICAO PRODUTO C', '"300001","300002","300003","300004","300005"'),
('00123404', 'AAD-DESCRICAO PRODUTO D', '"400001","400002","400003","400004","400005"'),
('00123405', 'AAE-DESCRICAO PRODUTO E', '"500001","500002","500003","500004","500005"'),
('00123406', 'AAF-DESCRICAO PRODUTO F', '"600001","600002","600003","600004","600005"'),
('00123407', 'AAG-DESCRICAO PRODUTO G', '"700001","700002","700003","700004","700005"'),
('00123408', 'AAH-DESCRICAO PRODUTO H', '"800001","800002","800003","800004","800005"'),
('00123409', 'AAI-DESCRICAO PRODUTO I', '"900001","900002","900003","900004","900005"'),
('00123410', 'AAJ-DESCRICAO PRODUTO J', '"010001","010002","010003","010004","010005"');

SELECT COL_A, LEFT(COL_B,3), NEW_COL_C 
FROM TBL_XPTO
JOIN JSON_TABLE(
	CONCAT('[',COL_C,']')  , # tratamento dos dados para ficar no mesmo padrão que um arquivo JSON
	'$[*]'
	COLUMNS (
	    NEW_COL_C VARCHAR(6) PATH '$'
	)
) AS TBL_XPTO_II

;

/*
# DE TBL_XPTO
COL_A        COL_B                          COL_C
-----------  ----------------------------- ----------------------------------------------
00123456     ABC-DESCRICAO QUALQUER        "111111","222222","333333","444444","555555"
*/

/*
# PARA TBL_XPTO_II
COL_A        LEFT(COL_B,3)                  COL_D
-----------  ----------------------------- ----------
00123456     ABC                           111111
00123456     ABC                           222222
00123456     ABC                           333333
00123456     ABC                           444444
00123456     ABC                           555555
*/

SELECT COL_A, LEFT(COL_B,3), NEW_COL_C 
FROM TBL_XPTO
JOIN JSON_TABLE(
	CONCAT('[',COL_C,']')  , -- tratamento da colunas para ficar no padrão JSON
	'$[*]'
	COLUMNS (
	    NEW_COL_C VARCHAR(6) PATH '$'
	)
) AS TBL_XPTO_II

;

```
--- 
🔪 Referências:

- [MySQL Docs - JSON_TABLE](https://dev.mysql.com/doc/refman/8.4/en/json-table-functions.html)
- [Blog MySQL sobre JSON_TABLE](https://dev.mysql.com/blog-archive/json_table-the-best-of-both-worlds/)

🤡🎃💀🔪
