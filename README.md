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
and cols.table_name = 'cliente';"

```

Output esperado:
```
+--------------+------------+-------------+--------------------------------------------------------------------------------+
| table_schema | table_name | column_name | filename                                                                       |
+--------------+------------+-------------+--------------------------------------------------------------------------------+
| ecommerce    | cliente    | nome        | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/200.dir/000.dir/FILE000.cdf |
| ecommerce    | cliente    | cpf         | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/201.dir/000.dir/FILE000.cdf |
| ecommerce    | cliente    | email       | /var/lib/columnstore/data1/000.dir/000.dir/011.dir/202.dir/000.dir/FILE000.cdf |
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