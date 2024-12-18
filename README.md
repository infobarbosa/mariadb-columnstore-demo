# MariaDB - ColumnStore Demo
Author: Prof. Barbosa<br>
Contact: infobarbosa@gmail.com<br>
Github: [infobarbosa](https://github.com/infobarbosa)

## Objetivo
Avaliar de forma rudimentar o comportamento do modelo de armazenamento baseado em **coluna**.<br>
Para isso faremos uso do MariaDb pela sua simplicidade e praticidade.

## Ambiente 
Este laborarório pode ser executado em qualquer estação de trabalho.<br>
Recomendo, porém, a execução em Linux.<br>
Caso não tenha uma estação de trabalho Linux à sua disposição, recomendo utilizar o AWS Cloud9. Para isso siga essas [instruções](Cloud9/README.md).

## Setup
Para começar, faça o clone deste repositório:
```
git clone https://github.com/infobarbosa/mariadb-columnstore-demo.git
```

### Atenção! 
Os comandos desse tutorial presumem que você está no diretório raiz do projeto.<br>
Após clonar o repositório, navegue no terminal para ele:
```
cd mariadb-columnstore-demo
```

## Docker
Por simplicidade, vamos utilizar o MariaDB em um container baseado em *Docker*.<br>
Na raiz do projeto está disponível um arquivo `compose.yaml` que contém os parâmetros de inicialização do container Docker.<br>
Embora não seja escopo deste laboratório o entendimento detalhado do Docker, recomendo o estudo do arquivo `compose.yaml`.

```
ls -la compose.yaml
```

Output esperado:
```
ls -la compose.yaml
-rw-r--r-- 1 barbosa barbosa 144 jul 16 23:20 compose.yaml
```

### Inicialização
```
docker compose up -d
```

Para verificar se está tudo correto:
```
docker compose logs -f
```
> Para sair do comando acima, digite `Control+C`

#### ATENÇÃO!
Aguarde alguns segundos e então execute o comando a seguir:
```
docker exec -it mcs1 provision 127.0.0.1
```

Output esperado:
```
> docker exec -it mcs1 provision 127.0.0.1
Adding PM(s) To Cluster ... done
Restarting Cluster ... done
Validating ColumnStore Engine ... done
```

### Conectando no container
Vamos nos conectar no container 
```
docker exec -it mcs1 /bin/bash
```

## A base de dados

### Database `ecommerce`
```
mariadb -e \
"CREATE DATABASE IF NOT EXISTS ecommerce;"
```

### Tabela `cliente`
```
mariadb -e "
CREATE TABLE ecommerce.cliente (
    id BIGINT,
    nome VARCHAR(100),
    data_nasc DATE,
    cpf VARCHAR(14),
    email VARCHAR(255)
)  engine=ColumnStore;"

```

Verificando se deu certo
```
mariadb -e \
"DESCRIBE ecommerce.cliente;"
```

Output esperado:
```
[root@mcs1 datasets-csv-clientes]# mariadb -e \
> "DESCRIBE ecommerce.cliente;"
+-----------+--------------+------+-----+---------+-------+
| Field     | Type         | Null | Key | Default | Extra |
+-----------+--------------+------+-----+---------+-------+
| id        | bigint(20)   | YES  |     | NULL    |       |
| nome      | varchar(100) | YES  |     | NULL    |       |
| data_nasc | date         | YES  |     | NULL    |       |
| cpf       | varchar(14)  | YES  |     | NULL    |       |
| email     | varchar(255) | YES  |     | NULL    |       |
+-----------+--------------+------+-----+---------+-------+
```

### O dataset `clientes.csv.gz`

Faça o clone do datase de clientes:
```
git clone https://github.com/infobarbosa/datasets-csv-clientes

```

Descompacte o arquivo
```
gunzip -c /datasets-csv-clientes/clientes.csv.gz > /datasets-csv-clientes/clientes.csv

```

Checando o arquivo:
```
head datasets-csv-clientes/clientes.csv
```

Output:
```
[root@mcs1 /]# head datasets-csv-clientes/clientes.csv
id;nome;data_nasc;cpf;email
1;Isabelly Barbosa;1963-08-15;137.064.289-03;isabelly.barbosa@example.com
2;Larissa Fogaça;1933-09-29;703.685.294-10;larissa.fogaca@example.com
3;João Gabriel Silveira;1958-05-27;520.179.643-52;joao.gabriel.silveira@example.com
4;Pedro Lucas Nascimento;1950-08-23;274.351.896-00;pedro.lucas.nascimento@example.com
5;Felipe Azevedo;1986-12-31;759.061.842-01;felipe.azevedo@example.com
6;Ana Laura Lopes;1963-04-27;165.284.390-60;ana.laura.lopes@example.com
7;Ana Beatriz Aragão;1958-04-21;672.135.804-26;ana.beatriz.aragao@example.com
8;Murilo da Rosa;1944-07-13;783.640.251-71;murilo.da.rosa@example.com
9;Alícia Souza;1960-08-26;784.563.029-29;alicia.souza@example.com
[root@mcs1 /]#
```

### Carga de dados
```
mariadb ecommerce -e "
LOAD DATA INFILE '/datasets-csv-clientes/clientes.csv'
INTO TABLE ecommerce.cliente
FIELDS TERMINATED BY ';'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(id, nome, data_nasc, cpf, email);"

```

Checando:
```
mariadb ecommerce -e "SELECT * FROM ecommerce.cliente LIMIT 10;"

```

Output:
```
[root@mcs1 /]# mariadb ecommerce -e "SELECT * FROM ecommerce.cliente LIMIT 10;"
+------+------------------------+------------+----------------+------------------------------------+
| id   | nome                   | data_nasc  | cpf            | email                              |
+------+------------------------+------------+----------------+------------------------------------+
|    1 | Isabelly Barbosa       | 1963-08-15 | 137.064.289-03 | isabelly.barbosa@example.com       |
|    2 | Larissa Fogaça         | 1933-09-29 | 703.685.294-10 | larissa.fogaca@example.com         |
|    3 | João Gabriel Silveira  | 1958-05-27 | 520.179.643-52 | joao.gabriel.silveira@example.com  |
|    4 | Pedro Lucas Nascimento | 1950-08-23 | 274.351.896-00 | pedro.lucas.nascimento@example.com |
|    5 | Felipe Azevedo         | 1986-12-31 | 759.061.842-01 | felipe.azevedo@example.com         |
|    6 | Ana Laura Lopes        | 1963-04-27 | 165.284.390-60 | ana.laura.lopes@example.com        |
|    7 | Ana Beatriz Aragão     | 1958-04-21 | 672.135.804-26 | ana.beatriz.aragao@example.com     |
|    8 | Murilo da Rosa         | 1944-07-13 | 783.640.251-71 | murilo.da.rosa@example.com         |
|    9 | Alícia Souza           | 1960-08-26 | 784.563.029-29 | alicia.souza@example.com           |
|   10 | Milena Silva           | 1937-09-04 | 912.534.067-07 | milena.silva@example.com           |
+------+------------------------+------------+----------------+------------------------------------+
```

## Datafiles
Vamos checar como é a organização dos arquivos de dados em uma estrutura colunar.<br>

Primeiro a estrutura da tabela:
```
mariadb -e "
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, COLUMN_TYPE, COLUMN_KEY, EXTRA, ENGINE
FROM information_schema.columns
JOIN information_schema.tables USING (TABLE_SCHEMA, TABLE_NAME)
WHERE TABLE_SCHEMA = 'ecommerce' AND TABLE_NAME = 'cliente';"
```

Output:
```
+--------------+------------+-------------+--------------+------------+-------+-------------+
| TABLE_SCHEMA | TABLE_NAME | COLUMN_NAME | COLUMN_TYPE  | COLUMN_KEY | EXTRA | ENGINE      |
+--------------+------------+-------------+--------------+------------+-------+-------------+
| ecommerce    | cliente    | id          | bigint(20)   |            |       | Columnstore |
| ecommerce    | cliente    | nome        | varchar(100) |            |       | Columnstore |
| ecommerce    | cliente    | data_nasc   | date         |            |       | Columnstore |
| ecommerce    | cliente    | cpf         | varchar(14)  |            |       | Columnstore |
| ecommerce    | cliente    | email       | varchar(255) |            |       | Columnstore |
+--------------+------------+-------------+--------------+------------+-------+-------------+
```

> Você não precisa entender em detalhes a consulta abaixo, basta assumir que o dicionário de dados do servidor mantém metadados que associam o objeto lógico (a tabela) a um objeto físico (arquivo de dados).

```
mariadb -e \
"select cols.table_schema, cols.table_name, cols.column_name, files.filename
from information_schema.columnstore_columns cols 
inner join information_schema.columnstore_files files 
on files.object_id = cols.dictionary_object_id
where cols.table_schema = 'ecommerce'
and cols.table_name = 'cliente';"

```

Output esperado:
```
+--------------+------------+-------------+--------------------------------------------------------------------------------+
| table_schema | table_name | column_name | filename                                                                       |
+--------------+------------+-------------+--------------------------------------------------------------------------------+
| ecommerce    | cliente    | nome        | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/193.dir/000.dir/FILE000.cdf |
| ecommerce    | cliente    | cpf         | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/194.dir/000.dir/FILE000.cdf |
| ecommerce    | cliente    | email       | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/195.dir/000.dir/FILE000.cdf |
+--------------+------------+-------------+--------------------------------------------------------------------------------+
```

### Inspecionando os datafiles
#### `hexdump`

Utilizando `hexdump` alterne os arquivos e verifique o conteúdo:
```
hexdump -C /var/lib/columnstore/data1/000.dir/000.dir/011.dir/193.dir/000.dir/FILE000.cdf | more

```

# Comparação de performances

## Objetivo
Nesta sessão vamos avaliar de forma rudimentar as performances do modelo de linha versus modelo colunar de armazenamento.

## A base de dados
Vamos criar um database `ecommerce`:
```
sudo mariadb -e "CREATE DATABASE IF NOT EXISTS ecommerce;"
```

## Tabelas
### `ecommerce.invoices` com engine **InnoDB**
```
sudo mariadb -e "
    CREATE TABLE ecommerce.invoices(
        InvoiceDate text,
        Country text,
        InvoiceNo text,
        StockCode text,
        Description text,
        CustomerID text,
        Quantity float,
        UnitPrice float
    ) engine=InnoDB;"
```

Verificando se deu certo:
```
sudo mariadb -e "DESCRIBE ecommerce.invoices;"
```

Output:
```
[root@mcs1 /]# sudo mariadb -e "DESCRIBE ecommerce.invoices;"
+-------------+-------+------+-----+---------+-------+
| Field       | Type  | Null | Key | Default | Extra |
+-------------+-------+------+-----+---------+-------+
| InvoiceDate | text  | YES  |     | NULL    |       |
| Country     | text  | YES  |     | NULL    |       |
| InvoiceNo   | text  | YES  |     | NULL    |       |
| StockCode   | text  | YES  |     | NULL    |       |
| Description | text  | YES  |     | NULL    |       |
| CustomerID  | text  | YES  |     | NULL    |       |
| Quantity    | float | YES  |     | NULL    |       |
| UnitPrice   | float | YES  |     | NULL    |       |
+-------------+-------+------+-----+---------+-------+
```

### `ecommerce.invoices_cs` com engine **ColumnStore**
```
sudo mariadb -e "
    CREATE TABLE ecommerce.invoices_cs(
        InvoiceDate text,
        Country text,
        InvoiceNo text,
        StockCode text,
        Description text,
        CustomerID text,
        Quantity float,
        UnitPrice float
    ) engine=ColumnStore;"
```

Verificando se deu certo:
```
sudo mariadb -e "DESCRIBE ecommerce.invoices_cs;"
```

Output:
```
[root@mcs1 /]# sudo mariadb -e "DESCRIBE ecommerce.invoices_cs;"
+-------------+-------+------+-----+---------+-------+
| Field       | Type  | Null | Key | Default | Extra |
+-------------+-------+------+-----+---------+-------+
| InvoiceDate | text  | YES  |     | NULL    |       |
| Country     | text  | YES  |     | NULL    |       |
| InvoiceNo   | text  | YES  |     | NULL    |       |
| StockCode   | text  | YES  |     | NULL    |       |
| Description | text  | YES  |     | NULL    |       |
| CustomerID  | text  | YES  |     | NULL    |       |
| Quantity    | float | YES  |     | NULL    |       |
| UnitPrice   | float | YES  |     | NULL    |       |
+-------------+-------+------+-----+---------+-------+
```

## Carga das tabelas

##### Voltando ao diretório `home`
```
cd ~
```

##### O arquivo `invoices.csv`
```
ls -latr /tmp/data/invoices.tar.gz
```

Descompacte o arquivo:
```
tar -xzf /tmp/data/invoices.tar.gz -C /tmp/
```

Examinando a estrutura do arquivo
```
head /tmp/invoices.csv
```

Número de linhas
```
wc -l /tmp/invoices.csv
```

### Carga de dados `invoices`
```
sudo mariadb -e "
    LOAD DATA INFILE '/tmp/invoices.csv'
    INTO TABLE ecommerce.invoices
    FIELDS TERMINATED BY ','
    ENCLOSED BY '\"'
    LINES TERMINATED BY '\n'
    IGNORE 1 ROWS;"
```

Verificando a carga:
```
sudo mariadb -e "select * from ecommerce.invoices limit 10;"
```

Output:
```
[root@mcs1 /]# sudo mariadb -e "select * from ecommerce.invoices limit 10;"
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| InvoiceDate          | Country        | InvoiceNo | StockCode | Description                         | CustomerID | Quantity | UnitPrice |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 85123A    | WHITE HANGING HEART T-LIGHT HOLDER  | 17850      |        6 |      2.55 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 71053     | WHITE METAL LANTERN                 | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84406B    | CREAM CUPID HEARTS COAT HANGER      | 17850      |        8 |      2.75 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029G    | KNITTED UNION FLAG HOT WATER BOTTLE | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029E    | RED WOOLLY HOTTIE WHITE HEART.      | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 22752     | SET 7 BABUSHKA NESTING BOXES        | 17850      |        2 |      7.65 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 21730     | GLASS STAR FROSTED T-LIGHT HOLDER   | 17850      |        6 |      4.25 |
| 2010-12-01T08:28:00Z | United Kingdom | 536366    | 22633     | HAND WARMER UNION JACK              | 17850      |        6 |      1.85 |
| 2010-12-01T08:28:00Z | United Kingdom | 536366    | 22632     | HAND WARMER RED POLKA DOT           | 17850      |        6 |      1.85 |
| 2010-12-01T08:34:00Z | United Kingdom | 536367    | 84879     | ASSORTED COLOUR BIRD ORNAMENT       | 13047      |       32 |      1.69 |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
```


### Carga de dados `invoices_cs`
```
sudo mariadb -e "
    LOAD DATA INFILE '/tmp/invoices.csv'
    INTO TABLE ecommerce.invoices_cs
    FIELDS TERMINATED BY ','
    ENCLOSED BY '\"'
    LINES TERMINATED BY '\n'
    IGNORE 1 ROWS;"
```

Verificando a carga:
```
sudo mariadb -e "select * from ecommerce.invoices_cs limit 10;"
```

Output:
```
[root@mcs1 /]# sudo mariadb -e "select * from ecommerce.invoices_cs limit 10;"
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| InvoiceDate          | Country        | InvoiceNo | StockCode | Description                         | CustomerID | Quantity | UnitPrice |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 84692     | BOX OF 24 COCKTAIL PARASOLS         | NULL       |        2 |      0.83 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 84872A    | TEATIME FUNKY FLOWER BACKPACK FOR 2 | NULL       |        1 |     10.79 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 84912B    | GREEN ROSE WASHBAG                  | NULL       |        4 |      3.29 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 84913A    | SOFT PINK ROSE TOWEL                | NULL       |        1 |      3.29 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 84970L    | SINGLE HEART ZINC T-LIGHT HOLDER    | NULL       |        4 |      2.08 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 84978     | HANGING HEART JAR T-LIGHT HOLDER    | NULL       |        4 |      2.46 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 85032B    | BLOSSOM IMAGES GIFT WRAP SET        | NULL       |        1 |      4.13 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 85039A    | SET/4 RED MINI ROSE CANDLE IN BOWL  | NULL       |        1 |      1.63 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 85039B    | S/4 IVORY MINI ROSE CANDLE IN BOWL  | NULL       |        1 |      1.63 |
| 2011-03-17T18:15:00Z | United Kingdom | 546888    | 85078     | SCANDINAVIAN 3 HEARTS NAPKIN RING   | NULL       |        2 |      1.63 |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
```

### Teste 1 
#### Consulta analítica

**`invoices`**
```
sudo mariadb -e "
    select count(distinct StockCode)
          ,max(UnitPrice) mx
          ,min(UnitPrice) mn
          ,avg(UnitPrice) average
    from ecommerce.invoices;"
```

**`invoices_cs`**
```
sudo mariadb -e "
    select count(distinct StockCode)
          ,max(UnitPrice) mx
          ,min(UnitPrice) mn
          ,avg(UnitPrice) average
    from ecommerce.invoices_cs;"
```

#### Medindo o tempo:
**`invoices`**
```
time { 
    sudo mariadb -e "
        select count(distinct StockCode)
            ,max(UnitPrice) mx
            ,min(UnitPrice) mn
            ,avg(UnitPrice) average
        from ecommerce.invoices;"
}
```

Output:
```
+---------------------------+-------+----------+-------------------+
| count(distinct StockCode) | mx    | mn       | average           |
+---------------------------+-------+----------+-------------------+
|                      3958 | 38970 | -11062.1 | 4.611113614622466 |
+---------------------------+-------+----------+-------------------+

real    0m2.401s
user    0m0.013s
sys     0m0.015s
```

**`invoices_cs`**
```
time { 
    sudo mariadb -e "
        select count(distinct StockCode)
            ,max(UnitPrice) mx
            ,min(UnitPrice) mn
            ,avg(UnitPrice) average
        from ecommerce.invoices_cs;"
}
```

Output:
```
+---------------------------+-------+----------+-------------------+
| count(distinct StockCode) | mx    | mn       | average           |
+---------------------------+-------+----------+-------------------+
|                      3958 | 38970 | -11062.1 | 4.611113614622465 |
+---------------------------+-------+----------+-------------------+

real    0m0.301s
user    0m0.016s
sys     0m0.010s
```

### Teste 2 
#### Busca linha completa com restrição de valor

**`invoices`**
```
time { 
    sudo mariadb -e "
        select * 
        from ecommerce.invoices 
        where InvoiceNo='536365';"; 
}
```

Output:
```
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| InvoiceDate          | Country        | InvoiceNo | StockCode | Description                         | CustomerID | Quantity | UnitPrice |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 85123A    | WHITE HANGING HEART T-LIGHT HOLDER  | 17850      |        6 |      2.55 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 71053     | WHITE METAL LANTERN                 | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84406B    | CREAM CUPID HEARTS COAT HANGER      | 17850      |        8 |      2.75 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029G    | KNITTED UNION FLAG HOT WATER BOTTLE | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029E    | RED WOOLLY HOTTIE WHITE HEART.      | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 22752     | SET 7 BABUSHKA NESTING BOXES        | 17850      |        2 |      7.65 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 21730     | GLASS STAR FROSTED T-LIGHT HOLDER   | 17850      |        6 |      4.25 |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+

real    0m0.477s
user    0m0.017s
sys     0m0.019s
```

**`invoices_cs`**
```
time { 
    sudo mariadb -e "
        select * 
        from ecommerce.invoices_cs 
        where InvoiceNo='536365';"; 
}
```

Output:
```
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| InvoiceDate          | Country        | InvoiceNo | StockCode | Description                         | CustomerID | Quantity | UnitPrice |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 85123A    | WHITE HANGING HEART T-LIGHT HOLDER  | 17850      |        6 |      2.55 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 71053     | WHITE METAL LANTERN                 | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84406B    | CREAM CUPID HEARTS COAT HANGER      | 17850      |        8 |      2.75 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029G    | KNITTED UNION FLAG HOT WATER BOTTLE | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029E    | RED WOOLLY HOTTIE WHITE HEART.      | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 22752     | SET 7 BABUSHKA NESTING BOXES        | 17850      |        2 |      7.65 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 21730     | GLASS STAR FROSTED T-LIGHT HOLDER   | 17850      |        6 |      4.25 |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+

real    0m0.154s
user    0m0.015s
sys	    0m0.013s
```

> Ops! O teste não foi necessariamente justo uma vez que a tabela `invoices` não tem um índice definido. 

#### Criando um índice na tabela `ecommerce.invoices`:
```
sudo mariadb -e "ALTER TABLE ecommerce.invoices ADD INDEX invoices_i1 (InvoiceNo);"

```

### Teste 3 

#### Consulta indexada

**`invoices`**
```
time {  
    sudo mariadb -e "
        select * 
        from ecommerce.invoices
        where InvoiceNo='536365';";
}
```

Output:
```
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| InvoiceDate          | Country        | InvoiceNo | StockCode | Description                         | CustomerID | Quantity | UnitPrice |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 85123A    | WHITE HANGING HEART T-LIGHT HOLDER  | 17850      |        6 |      2.55 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 71053     | WHITE METAL LANTERN                 | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84406B    | CREAM CUPID HEARTS COAT HANGER      | 17850      |        8 |      2.75 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029G    | KNITTED UNION FLAG HOT WATER BOTTLE | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029E    | RED WOOLLY HOTTIE WHITE HEART.      | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 22752     | SET 7 BABUSHKA NESTING BOXES        | 17850      |        2 |      7.65 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 21730     | GLASS STAR FROSTED T-LIGHT HOLDER   | 17850      |        6 |      4.25 |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+

real    0m0.073s
user    0m0.036s
sys     0m0.033s
```

**`invoices_cs`**
```
time {  
    sudo mariadb -e "
        select * 
        from ecommerce.invoices_cs
        where InvoiceNo='536365';";
}
```

Output:
```
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| InvoiceDate          | Country        | InvoiceNo | StockCode | Description                         | CustomerID | Quantity | UnitPrice |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 85123A    | WHITE HANGING HEART T-LIGHT HOLDER  | 17850      |        6 |      2.55 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 71053     | WHITE METAL LANTERN                 | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84406B    | CREAM CUPID HEARTS COAT HANGER      | 17850      |        8 |      2.75 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029G    | KNITTED UNION FLAG HOT WATER BOTTLE | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 84029E    | RED WOOLLY HOTTIE WHITE HEART.      | 17850      |        6 |      3.39 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 22752     | SET 7 BABUSHKA NESTING BOXES        | 17850      |        2 |      7.65 |
| 2010-12-01T08:26:00Z | United Kingdom | 536365    | 21730     | GLASS STAR FROSTED T-LIGHT HOLDER   | 17850      |        6 |      4.25 |
+----------------------+----------------+-----------+-----------+-------------------------------------+------------+----------+-----------+

real    0m0.137s
user    0m0.015s
sys     0m0.015s
```

### Teste de carga

Limpeza das tabelas:
**`invoices`**
```
sudo mariadb -e "truncate table ecommerce.invoices;"

```

**`invoices_cs`**
```
sudo mariadb -e "truncate table ecommerce.invoices_cs;"

```

Checagem do conteúdo após a limpeza:
```
sudo mariadb -e "select count(1) from ecommerce.invoices;"

```

```
sudo mariadb -e "select count(1) from ecommerce.invoices_cs;"

```


#### Medindo o tempo de carga:

**`invoices`**
```
time {
sudo mariadb -e "
    LOAD DATA INFILE '/tmp/invoices.csv'
    INTO TABLE ecommerce.invoices
    FIELDS TERMINATED BY ','
    ENCLOSED BY '\"'
    LINES TERMINATED BY '\n'
    IGNORE 1 ROWS;"
}

```

Output:
```
real    0m4.100s
user    0m0.012s
sys     0m0.017s
```

**`invoices_cs`**
```
time {
sudo mariadb -e "
    LOAD DATA INFILE '/tmp/invoices.csv'
    INTO TABLE ecommerce.invoices_cs
    FIELDS TERMINATED BY ','
    ENCLOSED BY '\"'
    LINES TERMINATED BY '\n'
    IGNORE 1 ROWS;"
}

```

Output:
```
real    0m6.728s
user    0m0.015s
sys	    0m0.018s
```

## Parabéns
Neste laboratório nós:
1. Criamos uma instância do MariaDB com armazenamento colunar;
2. Criamos um banco de dados `ecommerce` e uma tabela `cliente` dentro dele;
3. Inserimos alguns registros de exemplo;
4. Consultamos o dicionário de dados para entender a localização física dos arquivos de dados;
5. Inspecionamos os arquivos de dados e comprovamos a distribuição dos atributos da tabela em colunas;
6. Testamos e comparamos a performance dos modelos de armazenamento rowstore e columnstore.