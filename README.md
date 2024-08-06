# MariaDB ColumnStore
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
mariadb -e \
"CREATE TABLE IF NOT EXISTS ecommerce.cliente_cs(
    id_cliente int,
    cpf text,
    nome text
) engine=ColumnStore;"

```

Verificando se deu certo
```
mariadb -e \
"DESCRIBE ecommerce.cliente_cs;"
```

Output esperado:
```
mariadb -e \
"DESCRIBE ecommerce.cliente_cs;"
+------------+---------+------+-----+---------+-------+
| Field      | Type    | Null | Key | Default | Extra |
+------------+---------+------+-----+---------+-------+
| id_cliente | int(11) | YES  |     | NULL    |       |
| cpf        | text    | YES  |     | NULL    |       |
| nome       | text    | YES  |     | NULL    |       |
+------------+---------+------+-----+---------+-------+
```

## Operações em linhas (ou registros)

### 1. 1o. Insert
```
mariadb -e \
"INSERT INTO ecommerce.cliente_cs(id_cliente, cpf, nome)
VALUES (10, '11111111111', 'marcelo barbosa');"

```

Verificando:
```
mariadb -e \
"SELECT * FROM ecommerce.cliente_cs;"
```

Output:
```
mariadb -e \
"SELECT * FROM ecommerce.cliente_cs;"
+------------+-------------+-----------------+
| id_cliente | cpf         | nome            |
+------------+-------------+-----------------+
|         10 | 11111111111 | marcelo barbosa |
+------------+-------------+-----------------+
```

### 3. 2o. Insert
```
mariadb -e \
"INSERT INTO ecommerce.cliente_cs(id_cliente, cpf, nome)
VALUES (11, '22222222222', 'Juscelino Kubitschek');"
```

```
mariadb -e \
"SELECT * FROM ecommerce.cliente_cs;"
```

Output:
```
mariadb -e \
"SELECT * FROM ecommerce.cliente_cs;"
+------------+-------------+----------------------+
| id_cliente | cpf         | nome                 |
+------------+-------------+----------------------+
|         10 | 11111111111 | marcelo barbosa      |
|         11 | 22222222222 | Juscelino Kubitschek |
+------------+-------------+----------------------+
```

### 4. Insert em lote
```
mariadb -Bse \
"INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1001, '98753936060', 'MARIVALDA KANAMARY');
INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1002, '12455426050', 'JUCILENE MOREIRA CRUZ');
INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1003, '32487300051', 'GRACIMAR BRASIL GUERRA');
INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1004, '59813133074', 'ALDENORA VIANA MOREIRA');
INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1005, '79739952003', 'VERA LUCIA RODRIGUES SENA');
INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1006, '66142806000', 'IVONE GLAUCIA VIANA DUTRA');
INSERT INTO ecommerce.cliente_cs (id_cliente, cpf, nome) VALUES (1007, '19052330000', 'LUCILIA ROSA LIMA PEREIRA');"
```

Verificando se os inserts ocorreram como esperado:
```
mariadb -e \
"SELECT * FROM ecommerce.cliente_cs;"
```

Output esperado:
```
mariadb -e \
"SELECT * FROM ecommerce.cliente_cs;"
+------------+-------------+---------------------------+
| id_cliente | cpf         | nome                      |
+------------+-------------+---------------------------+
|         10 | 11111111111 | marcelo barbosa           |
|         11 | 22222222222 | Juscelino Kubitschek      |
|       1001 | 98753936060 | MARIVALDA KANAMARY        |
|       1002 | 12455426050 | JUCILENE MOREIRA CRUZ     |
|       1003 | 32487300051 | GRACIMAR BRASIL GUERRA    |
|       1004 | 59813133074 | ALDENORA VIANA MOREIRA    |
|       1005 | 79739952003 | VERA LUCIA RODRIGUES SENA |
|       1006 | 66142806000 | IVONE GLAUCIA VIANA DUTRA |
|       1007 | 19052330000 | LUCILIA ROSA LIMA PEREIRA |
+------------+-------------+---------------------------+
```

## Datafiles
Vamos checar como é a organização dos arquivos de dados em uma estrutura colunar.<br>
> Você não precisa entender em detalhes a consulta abaixo, basta assumir que o dicionário de dados do servidor mantém metadados que associam o objeto lógico (a tabela) a um objeto físico (arquivo de dados).

```
mariadb -e \
"select cols.table_schema, cols.table_name, cols.column_name, files.filename
from information_schema.columnstore_columns cols 
inner join information_schema.columnstore_files files 
on files.object_id = cols.dictionary_object_id
where cols.table_schema = 'ecommerce'
and cols.table_name = 'cliente_cs';"

```

Output esperado:
```
[root@mcs1 /]#     mariadb -e \
>         "select cols.table_schema, cols.table_name, cols.column_name, files.filename
>         from information_schema.columnstore_columns cols
>         inner join information_schema.columnstore_files files
>         on files.object_id = cols.dictionary_object_id
>         where cols.table_schema = 'ecommerce'
>         and cols.table_name = 'cliente_cs';"
+--------------+------------+-------------+--------------------------------------------------------------------------------+
| table_schema | table_name | column_name | filename                                                                       |
+--------------+------------+-------------+--------------------------------------------------------------------------------+
| ecommerce    | cliente_cs | cpf         | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/191.dir/000.dir/FILE000.cdf |
| ecommerce    | cliente_cs | nome        | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/192.dir/000.dir/FILE000.cdf |
+--------------+------------+-------------+--------------------------------------------------------------------------------+
```

### Inspecionando os datafiles
```
grep --binary-files=text --include \*.cdf -r -l -o "MARIVALDA" ./var/lib/columnstore/data1/*
```

Output esperado:
```
[root@mcs1 /]# grep --binary-files=text --include \*.cdf -r -l -o "MARIVALDA" ./var/lib/columnstore/data1/*
./var/lib/columnstore/data1/000.dir/000.dir/011.dir/192.dir/000.dir/FILE000.cdf
./var/lib/columnstore/data1/versionbuffer.cdf
[root@mcs1 /]#
```

```
grep --binary-files=text --include \*.cdf -r -l -o "98753936060" ./var/lib/columnstore/data1/*
```

Output esperado:
```
[root@mcs1 /]# grep --binary-files=text --include \*.cdf -r -l -o "98753936060" ./var/lib/columnstore/data1/*
./var/lib/columnstore/data1/000.dir/000.dir/011.dir/191.dir/000.dir/FILE000.cdf
./var/lib/columnstore/data1/versionbuffer.cdf
[root@mcs1 /]#
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

### Teste 1 
#### Consulta analítica
```
sudo mariadb -e "
    select count(distinct StockCode)
          ,max(UnitPrice) mx
          ,min(UnitPrice) mn
          ,avg(UnitPrice) average
    from ecommerce.invoices;"
```

```
sudo mariadb -e "
    select count(distinct StockCode)
          ,max(UnitPrice) mx
          ,min(UnitPrice) mn
          ,avg(UnitPrice) average
    from ecommerce.invoices_cs;"
```

#### Medindo o tempo:
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

### Teste 2 
#### Busca linha completa com restrição de valor
```
sudo mariadb -e "
    select * 
    from ecommerce.invoices 
    where InvoiceNo='536365';"
```

```
sudo mariadb -e "
    select * 
    from ecommerce.invoices_cs 
    where InvoiceNo='536365';"
```

#### Medindo o tempo:
```
time { 
    sudo mariadb -e "
        select * 
        from ecommerce.invoices 
        where InvoiceNo='536365';"; 
}
```

```
time { 
    sudo mariadb -e "
        select * 
        from ecommerce.invoices_cs 
        where InvoiceNo='536365';"; 
}
```

> Ops! O teste não foi necessariamente justo uma vez que a tabela `invoices` não tem um índice definido. 

#### Criando um índice na tabela `ecommerce.invoices`:
```
sudo mariadb -e "ALTER TABLE ecommerce.invoices ADD INDEX invoices_i1 (InvoiceNo);"
```

### Teste 3 

#### Consulta indexada
Repetindo o teste:
```
sudo mariadb -e "
    select * 
    from ecommerce.invoices
    where InvoiceNo='536365';"
```

```
sudo mariadb -e "
    select * 
    from ecommerce.invoices_cs 
    where InvoiceNo='536365';"
```

#### Medindo o tempo:
```
time {  
    sudo mariadb -e "
        select * 
        from ecommerce.invoices
        where InvoiceNo='536365';";
}
```

```
time {  
    sudo mariadb -e "
        select * 
        from ecommerce.invoices_cs
        where InvoiceNo='536365';";
}
```

### Teste de carga

Limpeza das tabelas:
```
sudo mariadb -e "truncate table ecommerce.invoices;";
```
```
sudo mariadb -e "truncate table ecommerce.invoices_cs;";
```

Checagem do conteúdo após a limpeza:
```
sudo mariadb -e "select count(1) from ecommerce.invoices;";
```
```
sudo mariadb -e "select count(1) from ecommerce.invoices_cs;";
```
	

#### Medindo o tempo de carga:
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

## Parabéns
Neste laboratório nós:
1. Criamos uma instância do MariaDB com armazenamento colunar;
2. Criamos um banco de dados `ecommerce` e uma tabela `cliente` dentro dele;
3. Inserimos alguns registros de exemplo;
4. Consultamos o dicionário de dados para entender a localização física dos arquivos de dados;
5. Inspecionamos os arquivos de dados e comprovamos a distribuição dos atributos da tabela em colunas;
6. Testamos e comparamos a performance dos modelos de armazenamento rowstore e columnstore.