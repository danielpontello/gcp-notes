## Introdução

Este repositório contém algumas anotações sobre os serviços do Google Cloud Platform, escritas a medida que vou descobrindo os recursos da plataforma.

## Cloud SQL

O Cloud SQL é o serviço de Banco de Dados SQL do GCP, permitindo a criação de bancos MySQL e PostgreSQL.

## Confguração

**APIs necessárias:** Cloud SQL Admin

### Cloud SQL Proxy 

O Cloud SQL Proxy permite o acesso a instâncias do Cloud SQL via um cliente local (como o Workbench) e também permite testar de forma local aplicações do Google App Engine. 

#### Instalação

Para instalar o SQL Proxy, rode os comandos abaixo:

```shell
$ wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
$ chmod +x cloud_sql_proxy
```

#### Executando

Para executar o Cloud SQL Proxy, é necessário a string de nome da instância do Cloud SQL, localizada ao clicar em uma instância na [Lista de Instâncias do Cloud SQL](https://console.cloud.google.com/sql/instances/). O formato dessa string é `<NOME DO PROJETO>:<REGIÃO DO PROJETO>:<NOME DA INSTÂNCIA>`. Após isso, existem 2 modos de funcionamento do proxy:

**Socket TCP:** Este método pode ser utilizado para conectar ao banco na nuvem utilizando o MySQL Workbench. Ele cria um Socket TCP que pode ser usado para conectar nas instâncias do Cloud SQL de forma 'tradicional'. Para utilizar este modo, basta rodar:

```
$ ./cloud_sql_proxy -instances=<INSTANCE_CONNECTION_NAME>=tcp:3306
```

O banco de dados poderá então ser acessado utilizando o IP local (127.0.0.1 ou localhost), utilizando os mesmos usuário e senha definidos na configuração do banco na GCP. 

**Socket Unix:** Este método cria um socket UNIX em um diretório especificado pelo usuário. Este é o mesmo método utilizado internamente pelo App Engine, permitindo que as aplicações do App Engine sejam testadas em um ambiente local com mínimas alterações. Para utilizar este modo, primeiro crie um diretório que conterá o arquivo do socket (necessita de acesso root):

```
$ sudo mkdir /cloudsql
$ sudo chmod 777 /cloudsql
```

Após isso, execute o Proxy com:

```
$ ./cloud_sql_proxy -dir=/cloudsql &
```

O socket será criado na pasta /cloudsql/ com o nome da sua instância do Cloud SQL. Este socket então pode ser usado por sua aplicação App Engine para obter acesso ao Cloud SQL, como mostrado no exemplo abaixo. Exemplo completo disponível nos [samples oficiais do GCP no GitHub](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/master/cloud-sql/mysql/sqlalchemy)

```python
# The SQLAlchemy engine will help manage interactions, including automatically
# managing a pool of connections to your database
db = sqlalchemy.create_engine(
    # Equivalent URL:
    # mysql+pymysql://<db_user>:<db_pass>@/<db_name>?unix_socket=/cloudsql/<cloud_sql_instance_name>
    sqlalchemy.engine.url.URL(
        drivername='mysql+pymysql',
        username=db_user,
        password=db_pass,
        database=db_name,
        query={
            'unix_socket': '/cloudsql/{}'.format(cloud_sql_instance_name)
        }
    ),
    # ... Specify additional properties here.
    # ...
)
```
