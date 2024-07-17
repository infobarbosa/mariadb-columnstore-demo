# MySQL Rowstore
Author: Prof. Barbosa<br>
Contact: infobarbosa@gmail.com<br>
Github: [infobarbosa](https://github.com/infobarbosa)

## Objetivo
Avaliar de forma rudimentar o comportamento do modelo de armazenamento baseado em linha.<br>
Para isso faremos uso do MySQL pela sua simplicidade e praticidade.

## Ambiente 
Este laborarório pode ser executado em qualquer estação de trabalho.<br>
Recomendo, porém, a execução em Linux.<br>
Caso você não tenha um à sua disposição, há duas opções:
1. AWS Cloud9: siga essas [instruções](Cloud9/README.md).
2. Killercoda: disponibiilizei o lab [aqui](https://killercoda.com/infobarbosa/scenario/mysql)

## Setup
Para começar, faça o clone deste repositório:
```
https://github.com/infobarbosa/mysql-rowstore-demo.git
```

>### Atenção! 
> Os comandos desse tutorial presumem que você está no diretório raiz do projeto.

## Docker
Por simplicidade, vamos utilizar o MariaDB em um container baseado em *Docker*.<br>
Na raiz do projeto está disponível um arquivo `compose.yaml` que contém os parâmetros de inicialização do container Docker.<br>
Embora não seja escopo deste laboratório o entendimento detalhado do Docker, recomendo o estudo do arquivo `compose.yaml`.

```
ls -la compose.yaml
```

Output esperado:
```
barbosa@brubeck:~/labs/mysql8$ ls -la compose.yaml
-rw-r--r-- 1 barbosa barbosa 589 jul 16 14:48 compose.yaml
```

### Inicialização
```
docker compose up -d
```

Para verificar se está tudo correto:
```
docker compose logs -f
```

Adicionalmente aguarde alguns segundos e então execute o comando a seguir:
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


## A base de dados

### Database `ecommerce`
```
docker exec -it mcs1 \
    mariadb -e \
    "CREATE DATABASE IF NOT EXISTS ecommerce;"
```

### Tabela `cliente`
```
docker exec -it mcs1 \
    mariadb -e \
    "CREATE TABLE IF NOT EXISTS ecommerce.cliente_cs(
        id_cliente int,
        cpf text,
        nome text
    ) engine=ColumnStore;"

```

Verificando se deu certo
```
docker exec -it mcs1 \
    mariadb -e \
    "DESCRIBE ecommerce.cliente_cs;"
```

Output esperado:
```
docker exec -it mcs1 \
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
docker exec -it mcs1 \
    mariadb -e \
        "INSERT INTO ecommerce.cliente_cs(id_cliente, cpf, nome)
        VALUES (10, '11111111111', 'marcelo barbosa');"

```

Verificando:
```
docker exec -it mcs1 \
    mariadb -e \
        "SELECT * FROM ecommerce.cliente_cs;"
```

Output:
```
docker exec -it mcs1 \
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
docker exec -it mcs1 \
    mariadb -e \
        "INSERT INTO ecommerce.cliente_cs(id_cliente, cpf, nome)
        VALUES (11, '22222222222', 'Juscelino Kubitschek');"
```

```
docker exec -it mcs1 \
    mariadb -e \
        "SELECT * FROM ecommerce.cliente_cs;"
```

Output:
```
docker exec -it mcs1 \
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
docker exec -it mcs1 \
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
docker exec -it mcs1 \
    mariadb -e \
    "SELECT * FROM ecommerce.cliente_cs;"
```

Output esperado:
```
docker exec -it mcs1 \
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

Faça o flush novamente e verifique o arquivo:
```
docker exec -it mysql8 \
    mysql -u root -e \
    "FLUSH LOCAL TABLES ecommerce.cliente FOR EXPORT;"
```

```
docker exec -it mcs1 \
    mariadb -e \
        "select cols.table_schema, cols.table_name, cols.column_name, files.filename
        from information_schema.columnstore_columns cols 
        inner join information_schema.columnstore_files files 
        on files.object_id = cols.dictionary_object_id
        where cols.table_schema = 'ecommerce'
        and cols.table_name = 'cliente_cs';"

```

```

docker exec -it mcs1 \
grep --binary-files=text --include /var/lib/columnstore/data1/000.dir/000.dir/011.dir/192.dir/000.dir/FILE000.cdf -r -l -o "MARIVALDA" ./*

grep --binary-files=text --include \*.cdf -r -l -o "MARIVALDA" ./*
grep --binary-files=text --include \*.cdf -r -l -o "MARIVALDA" ./*

```

Output:
```

```


### 5. Delete
```
docker exec -it mysql8 \
    mysql -u root -e \
    "DELETE FROM ecommerce.cliente WHERE id = 11;"
```

Faça o flush novamente e verifique o arquivo:
```
docker exec -it mysql8 \
    mysql -u root -e \
    "FLUSH LOCAL TABLES ecommerce.cliente FOR EXPORT;"
```

```
docker exec -it mysql8 \
    cat /var/lib/mysql/ecommerce/cliente.ibd
```

> Perceba o espaço vazio entre o registro `marcelo` e `MARIVALDA`.

### 6. Update
```
docker exec -it mysql8 \
    mysql -u root -e \
    "UPDATE ecommerce.cliente SET nome='MARI K.' WHERE id = 1001"

```

```
docker exec -it mysql8 \
    mysql -u root -e \
    "FLUSH LOCAL TABLES ecommerce.cliente FOR EXPORT;"
```

```
docker exec -it mysql8 \
    cat /var/lib/mysql/ecommerce/cliente.ibd
```

> Perceba que o update praticamente não alterou o layout do arquivo.

```
docker exec -it mysql8 \
    mysql -u root -e \
    "UPDATE ecommerce.cliente SET nome='MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY' WHERE id = 1001;"
```

```
docker exec -it mysql8 \
    mysql -u root -e \
    "FLUSH LOCAL TABLES ecommerce.cliente FOR EXPORT;"
```

```
docker exec -it mysql8 \
    cat /var/lib/mysql/ecommerce/cliente.ibd
```

> Perceba agora que, em razão do tamanho do nome, o banco de dados realocou o registro para um novo bloco (ou, possivelmente, outra posição no mesmo bloco)

## Parabéns
