# Engenharia Reversa e Views no E-commerce
## 1. Diagnóstico do problema — o debug arquitetural

O problema central da View original é a ausência da cláusula `WHERE`. A consulta relaciona clientes, pedidos e itens, mas não restringe os pedidos ao status `Shipped` nem ao ano de 2004. Portanto, pedidos cancelados, retidos e de outros anos também entram no relatório, desde que tenham registros correspondentes nas tabelas relacionadas.

Na estrutura da base ClassicModels, as três tabelas envolvidas têm as seguintes funções:

| Tabela | Informações utilizadas | Relacionamento |
|---|---|---|
| `customers` | Código e nome do cliente | `customerNumber` identifica o cliente. |
| `orders` | Número, data, status e cliente do pedido | `customerNumber` referencia `customers`. |
| `orderdetails` | Produtos, quantidades e preços de venda | `orderNumber` referencia `orders`; a chave primária combina pedido e produto. |

As condições dos `JOINs` estão corretas: cada pedido é associado ao seu cliente e cada item é associado ao seu pedido. Um cliente pode ter vários pedidos, e um pedido pode conter vários itens. Por isso, `SUM(od.quantityOrdered * od.priceEach)` calcula o total dos itens de cada pedido adequadamente. Entretanto, relacionar as tabelas não equivale a aplicar as regras de negócio: um pedido `Cancelled` também pode ter cliente e itens válidos.

O `GROUP BY` apenas reúne os itens para calcular um total por pedido. Ele não verifica se o pedido foi enviado nem se pertence ao período solicitado. Na refatoração, o agrupamento explicita o identificador e o nome do cliente, além do número do pedido, evitando depender da inferência de dependência funcional pelo SGBD. Agrupar somente por `orderNumber` não é a causa da inclusão dos pedidos indevidos; o MySQL pode aceitar essa forma quando reconhece as dependências entre as chaves.

Também existe um problema de interpretação do indicador: o resultado é o **valor de venda dos pedidos selecionados**. A expressão não desconta custos e despesas nem confirma pagamentos recebidos; portanto, não representa, por si só, lucro ou entrada efetiva de caixa.

## 2. Refatoração — o conserto da View

### 2.1. Critério adotado para o período

Como o enunciado não define um calendário fiscal diferente, considera-se o intervalo de **1º de janeiro a 31 de dezembro de 2004**. Neste relatório, o ano é determinado por `orderDate`: são pedidos realizados em 2004 cujo status atual é `Shipped`.

Essa premissa distingue a data do pedido da data de envio. Se a diretoria desejar especificamente os **envios realizados em 2004**, os dois filtros de data abaixo deverão utilizar `shippedDate`. O status atual também não permite reconstruir, sozinho, qual era a situação de cada pedido no encerramento de 2004.

### 2.2. Comando SQL para MySQL

```sql
USE classicmodels;

CREATE OR REPLACE
SQL SECURITY DEFINER
VIEW view_relatorio_financeiro AS
SELECT
    c.customerName,
    o.orderNumber,
    SUM(od.quantityOrdered * od.priceEach) AS valor_total
FROM customers AS c
INNER JOIN orders AS o
    ON c.customerNumber = o.customerNumber
INNER JOIN orderdetails AS od
    ON o.orderNumber = od.orderNumber
WHERE o.status = 'Shipped'
  AND o.orderDate >= '2004-01-01'
  AND o.orderDate <  '2005-01-01'
GROUP BY
    c.customerNumber,
    c.customerName,
    o.orderNumber;
```

A condição `status = 'Shipped'` admite exclusivamente o estado solicitado. Usar apenas `status <> 'Cancelled'` ainda permitiria, por exemplo, pedidos `On Hold`. O intervalo de datas inclui todo o ano de 2004 e mantém a coluna sem funções no filtro, permitindo que o otimizador considere um índice adequado. O `WHERE` restringe os registros antes da agregação.

O gerente pode consultar o painel com:

```sql
SELECT customerName, orderNumber, valor_total
FROM classicmodels.view_relatorio_financeiro
ORDER BY orderNumber;
```

### 2.3. Validação com a base ClassicModels

Foi utilizada a base pública disponibilizada pelo [MySQLTutorial.org](https://www.mysqltutorial.org/getting-started-with-mysql/mysql-sample-database/), contendo **122 clientes, 326 pedidos e 2.996 itens de pedido**. A validação foi executada em **SQLite 3.53.3**, com os dados importados do arquivo original. Foram removidos os comandos específicos de seleção/criação do banco e adaptados os escapes de apóstrofos; a consulta da View preservou os mesmos `JOINs`, filtros e agrupamento apresentados acima.

**Limite da validação:** os resultados abaixo comprovam a seleção dos dados nessa cópia da ClassicModels. Não foi executado um servidor MySQL neste ambiente; a sintaxe `CREATE OR REPLACE`, o contexto `SQL SECURITY` e as permissões devem ser verificados no MySQL da disciplina. No SQLite, foi utilizado `CREATE VIEW` para executar a consulta equivalente.

| Verificação executada | Resultado observado |
|---|---:|
| Pedidos de 2004 com status `Shipped` na tabela `orders` | 145 |
| Pedidos de 2004 com status `Cancelled` na tabela `orders` | 4 |
| Pedidos de 2004 com status `On Hold` na tabela `orders` | 1 |
| Pedidos de 2004 com status `Resolved` na tabela `orders` | 1 |
| Pedidos retornados pela View corrigida | 145 |
| Pedidos `Cancelled` retornados pela View | 0 |
| Pedidos com status diferente de `Shipped` ou fora do período na View | 0 |
| Pedidos elegíveis ausentes na View | 0 |
| Números de pedido duplicados na View | 0 |

Os testes evitam uma falsa aprovação por resultado vazio: a View excluiu os pedidos indevidos e manteve os 145 pedidos elegíveis existentes nessa base.

Exemplo dos três primeiros registros, ordenados pelo número do pedido:

| customerName | orderNumber | valor_total |
|---|---:|---:|
| Saveley & Henriot, Co. | 10208 | 49614.72 |
| Men 'R' US Retailers, Ltd. | 10209 | 21053.69 |
| Osaka Souveniers Co. | 10210 | 47177.59 |

### 2.4. Consultas para reproduzir a auditoria no MySQL

As consultas seguintes devem ser executadas por uma conta de auditoria com acesso à View e às tabelas base.

```sql
USE classicmodels;

-- Teste 1: deve retornar 0.
SELECT COUNT(*) AS pedidos_cancelados
FROM view_relatorio_financeiro AS v
JOIN orders AS o ON o.orderNumber = v.orderNumber
WHERE o.status = 'Cancelled';

-- Teste 2: deve retornar apenas Shipped; na base consultada, 145.
SELECT o.status, COUNT(*) AS quantidade
FROM view_relatorio_financeiro AS v
JOIN orders AS o ON o.orderNumber = v.orderNumber
GROUP BY o.status;

-- Teste 3: deve retornar 0.
SELECT COUNT(*) AS pedidos_invalidos
FROM view_relatorio_financeiro AS v
JOIN orders AS o ON o.orderNumber = v.orderNumber
WHERE o.status <> 'Shipped'
   OR o.orderDate <  '2004-01-01'
   OR o.orderDate >= '2005-01-01';

-- Teste 4: deve retornar 0; verifica a completude do relatório.
SELECT COUNT(*) AS pedidos_elegiveis_ausentes
FROM orders AS o
WHERE o.status = 'Shipped'
  AND o.orderDate >= '2004-01-01'
  AND o.orderDate <  '2005-01-01'
  AND NOT EXISTS (
      SELECT 1
      FROM view_relatorio_financeiro AS v
      WHERE v.orderNumber = o.orderNumber
  );

-- Teste 5: deve retornar nenhuma linha.
SELECT orderNumber, COUNT(*) AS ocorrencias
FROM view_relatorio_financeiro
GROUP BY orderNumber
HAVING COUNT(*) > 1;
```

## 3. Síntese teórica — segurança e abstração

O gerente pediu uma View porque precisa consultar o relatório com frequência. Se tivesse que executar o SQL completo toda vez, teria que lidar com as junções, os filtros e o cálculo dos valores. A View deixa essa consulta definida no banco e facilita o uso no dia a dia. A aula da Bóson Treinamentos e seu material de apoio mostram essa simplificação com um exemplo que reúne livros e autores.

### 3.1. Segurança de dados

Uma vantagem das Views é permitir que cada usuário consulte apenas os dados necessários para sua atividade. O artigo da DevMedia explica que elas podem limitar as informações exibidas, inclusive escolhendo quais colunas ficam disponíveis para cada usuário (Wagner, 2008).

No caso da ClassicModels, o gerente precisa do nome do cliente, do número do pedido e do valor total. Para esse relatório, não é necessário mostrar telefone, endereço ou limite de crédito. A View pode apresentar somente os três campos solicitados e os pedidos que atendem aos filtros.

Mas criar a View, sozinho, não garante essa proteção. Também é preciso configurar as permissões de acesso. Se o gerente continuar com acesso livre às tabelas originais, poderá consultar os dados que ficaram de fora do relatório. Portanto, o uso da View deve acompanhar o controle das permissões no banco.

### 3.2. Abstração da complexidade

Abstrair a complexidade significa permitir o uso de uma consulta sem exigir que a pessoa conheça todos os detalhes de como ela foi construída. Na aula da Bóson, a View evita que seja necessário escrever novamente a junção entre as tabelas a cada consulta (Bóson Treinamentos, [s.d.]).

No nosso caso, os relacionamentos entre clientes, pedidos e itens, junto com os filtros de status e data, ficam na definição da View. O gerente pode consultar o resultado com:

```sql
SELECT * FROM view_relatorio_financeiro;
```

Isso também facilita a manutenção. Se a regra do relatório mudar, a equipe pode ajustar a definição da View. Assim, as pessoas que utilizam esse relatório passam a consultar a mesma regra atualizada. Ainda assim, a consulta precisa ser revisada: como o caso mostrou, uma View também pode repetir um erro quando sua lógica está incorreta.

## Referências

BÓSON TREINAMENTOS. **MySQL - VIEWS - Criando Tabelas Virtuais (Visões) - 28**. [S. l.]: Bóson Treinamentos, [s.d.]. 1 vídeo. Apresentação: Fábio dos Reis. Disponível em: [https://www.youtube.com/watch?v=m1C8c8QRz4s](https://www.youtube.com/watch?v=m1C8c8QRz4s). Acesso em: 3 nov. 2026. Material de apoio da aula, publicado em 13 fev. 2014: [MySQL – VIEWS – Criando Tabelas Virtuais (Visões)](https://www.bosontreinamentos.com.br/mysql/mysql-views-criando-tabelas-virtuais-visoes-28/).

WAGNER. **MYSQL – Trabalhando com Views**. [S. l.]: DevMedia, 2008. Disponível em: [https://www.devmedia.com.br/mysql-trabalhando-com-views/8724](https://www.devmedia.com.br/mysql-trabalhando-com-views/8724). Acesso em: 3 nov. 2026.
