## Introdução

Este repositório contém algumas anotações sobre os serviços do Google Cloud Platform.

## Cloud Storage

O Cloud Storage é o sistema de armazenamento de arquivos do GCP. Ele armazena os dados em 'buckets' (também chamados de 'intervalos'), que funcionam de forma similar a partições de um disco rígido.

### `gsutil` e buckets

Para utilizar o Cloud Storage, é utilizada a ferramenta `gsutil`, da Cloud SDK. Primeiramente, é necessário criar um bucket para armazenar seus arquivos. Para criar um bucket, basta executar o comando:

```sh
gsutil mb gs://hello-bucket
```

Isto criará um novo bucket com o nome `hello-bucket`. Com o bucket criado, podemos transferir arquivos com o comando `gsutil cp`:

```sh
echo "Isto é um arquivo de teste" > teste.txt
gsutil cp teste.txt gs://hello-bucket
```

O `gsutil cp` pode ser utilizado tanto para copiar arquivos para o bucket quando para realizar download de arquivos para sua máquina local.
Além do `gsutil cp`, alguns comandos padrão do Linux também podem ser executados, como mostrado na tabela abaixo:

| Comando | Descrição |
|--------:|:----------|
| `gsutil mv <origem> <destino>` | Move um arquivo |
| `gsutil rm gs://<nome_do_bucket>/<arquivo>` | Remove um arquivo |
| `gsutil ls gs://<nome_do_bucket>` | Lista o conteúdo de um bucket |
| `gsutil cat gs://<nome_do_bucket>/arquivo` | Mostra o conteúdo de um arquivo |

### Acessando de uma aplicação

O Cloud Storage pode ser acessado por aplicações utilizando a biblioteca `google-cloud-storage`. No Python, instale-a utilizando o comando

```sh
pip install google-cloud-storage
```

Para realizar o download de um arquivo do Cloud Storage, pode-se utilizar um código semelhante ao apresentado abaixo:

```python
def download_blob(bucket_name, source_file_name, destination_file_name):
    """Downloads a blob from the bucket."""
    storage_client = storage.Client()
    bucket = storage_client.get_bucket(bucket_name)
    blob = bucket.blob(source_file_name)

    blob.download_to_filename(destination_file_name)

    print('File {} downloaded to {}.'.format(
        source_file_name,
        destination_file_name))
```

A documentação completa pode ser encontrada [aqui](https://cloud.google.com/storage/docs/reference/libraries).

## Cloud Functions

Cloud Functions são funções que são executadas 'serverless', sem a necessidade de configuração de um servidor, e são ativadas via requisições HTTP. Elas também podem ser executadas como resposta a certos eventos (upload no Cloud Storage, inserção no Cloud SQL, etc.).

Cloud Functions podem utilizar bibliotecas e acessar o sistema de arquivos normalmente, porém possuem permissão de escrita apenas no diretório `/tmp`.

### Criando uma Function

As Cloud Functions utilizam as mesmas estruturas de dados do servidor [Flask](http://flask.pocoo.org/). Uma Function possui estrutura semelhante a apresentada abaixo:

```python
def hello_get(request):
    return 'Hello World!'
```

A função recebe um parâmetro do tipo [Request](http://flask.pocoo.org/docs/1.0/api/#flask.Request), e pode retornar qualquer objeto compatível com o método [make_response](http://flask.pocoo.org/docs/1.0/api/#flask.Flask.make_response).

Para implementar esta função e fazê-la responder a requests HTTP, salve seu código com o nome `main.py` execute o comando:

```sh
gcloud functions deploy hello_get --runtime python37 --trigger-http
```

Após a implantação, podemos descobrir a URL de acesso a ela com o comando:

```sh
gcloud functions describe hello_get
```

A URL terá o formato `
https://REGIAO_GCP-ID_DO_PROJETO.cloudfunctions.net/hello_get`. Para verificar o funcionamento da função, basta acessar esta URL em um navegador. Caso ela esteja funcionando, uma mensagem de "Hello World!" deverá aparecer em seu browser.

Comandos como o `print` serão exibidos nos registros do Stackdriver, localizados [neste link](https://console.cloud.google.com/logs/viewer).

### Executar uma função ao realizar ações no Cloud Storage

Para fazer uma Function responder a eventos do Cloud Storage, é necessária a criação de uma Background Cloud Function. Ela tem a seguinte estrutura:

```python
def hello_gcs_generic(data, context):
    print('Event ID: {}'.format(context.event_id))
    print('Event type: {}'.format(context.event_type))
    print('Bucket: {}'.format(data['bucket']))
    print('File: {}'.format(data['name']))
    print('Metageneration: {}'.format(data['metageneration']))
    print('Created: {}'.format(data['timeCreated']))
    print('Updated: {}'.format(data['updated']))
```

O parâmetro `data` conterá os dados dos arquivos inseridos no Storage, como nome, bucket e data de criação. O arquivo atualizado pode ser acessado utilizando as bibliotecas do Cloud Storage e as funções padrão de acesso a arquivos do Python.

Para realizar o deploy da função, execute o seguinte comando, substituindo NOME_DO_BUCKET pelo nome do seu bucket no Cloud Storage:

```sh
gcloud functions deploy hello_gcs_generic --runtime python37 --trigger-resource NOME_DO_BUCKET --trigger-event google.storage.object.finalize
```

O comando acima fará com que a função seja executada toda vez que um novo arquivo seja enviado para o Storage (trigger `google.storage.object.finalize`). Todos os possíveis triggers estão listados na tabela abaixo:

| Trigger | Ação |
|--------:|:-----|
|`google.storage.object.finalize`| Acionado quando um arquivo é criado|
|`google.storage.object.delete`|Acionado quando um arquivo é removido|
|`google.storage.object.archive`|Acionado quando um arquivo é arquivado|
|`google.storage.object.metadataUpdate`|Acionado quando os metadados de um arquivo são alterados|

### Variáveis de Ambiente

Para definir as variáveis de ambiente para sua Function, crie um arquivo chamado `.env.yaml` na mesma pasta da sua function, com o seguinte formato:

```yaml
DB_USER: usuario
DB_PASS: minhasenhasecreta
DB_NAME: livros
```

Para utilizar o arquivo, durante o deploy da função, passe o parâmetro `--env-vars-file` com o nome do arquivo criado:

```sh
gcloud functions deploy minha_funcao --env-vars-file .env.yaml
```

As variáveis podem também ser definidas diretamente pela linha de comando:

```sh
gcloud functions deploy minha_funcao --set-env-vars DB_USER=usuario,DB_PASS=minhasenhasecreta DB_NAME=livros
```

Para acessá-las de dentro da Function, pode se utilizar a função `os.environ.get`:

```python
db_user = os.environ.get("DB_USER")
db_pass = os.environ.get("DB_PASS")
db_name = os.environ.get("DB_NAME")
```

As variáveis de ambiente podem ser utilizadas para armazenar dados sensíveis, como usuários e senhas de máquinas e bancos de dados. Como são voláteis (permanecem apenas na memória RAM), elas são consideradas mais seguras do que armazenamento em arquivos de configuração ou no código de sua aplicação.

## Cloud Source Repositories

O Cloud Source Repositories é o serviço de hospedagem de repositórios Git do GCP (como uma espécie de GitHub ou BitBucket privado).

**APIs necessárias:** Cloud Source Repositories

### Criando um novo repositório

Para criar um novo repositório chamado `hello-world`, basta rodar o comando 

```sh
gcloud source repos create hello-world
```

Após isto, clone o repositório para sua máquina com o comando

```sh
gcloud source repos clone hello-world
```

Com o repositório configurado, é possível utilizá-lo como um repositório git comum:

```sh
echo "Ola Mundo!" >> arquivo.txt
git add arquivo.txt
git commit -m "Upload inicial"
git push origin master
```

Os arquivos do repositório podem ser visualizados acessando o [Painel do Cloud Source Repositories](https://source.cloud.google.com/repos?hl=pt-br).

### Implantação automática no Google App Engine

É possível configurar o Cloud Source Repositories para automaticamente atualizar aplicações do App Engine ao receber alterações em repositórios do Cloud Source.

**APIs necessárias:** Administrador do App Engine, Cloud Build

Primeiramente, é necessário adicionar as permissões necessárias a conta de serviço do Cloud Build na [Página de Gerenciamento de IAM](https://console.cloud.google.com/projectselector/iam-admin/iam). A conta de serviço do Cloud Build tem o nome `[NUMERO_DO_PROJETO]@cloudbuild.gserviceaccount.com`, aonde `[NUMERO_DO_PROJETO]` é o número do projeto no GCP. No menu Editar (ícone de Lápis), adicione o papel **Administrador do App Engine** para a conta.

Em seu projeto do App Engine, crie um arquivo chamado `cloudbuild.yaml`, com o seguinte conteúdo:

```yaml
steps:
- name: "gcr.io/cloud-builders/gcloud"
  args: ["app", "deploy"]
timeout: "1600s"
```

Adicione o arquivo ao repositório:

```sh
git add cloudbuild.yaml
git commit -m "Adicionado cloudbuild.yaml"
git push origin master
```

A última etapa é criar um acionador de versão, que invocará o Cloud Build a cada mudança no repositório do Cloud Source. Para isso, no [Painel de Controle do Cloud Build](https://console.cloud.google.com/cloud-build/triggers), crie um novo Acionador com as seguintes configurações:

| Configuração | Valor |
|------------:|:-----|
| Origem       | Repositório de origem do Google Cloud |
| Repositório  | Nome do seu repositório no Cloud Source |
| Nome | Nome do Acionador |
| Configuração de Compilação | cloudbuild.yaml |
| Local do cloudbuild.yaml | /cloudbuild.yaml |

Com o acionador criado, as alterações no repositório serão automaticamente propagadas para o App Engine. O progresso das alterações pode ser visto [nesta página](https://console.cloud.google.com/cloud-build/builds)

## Cloud SQL

O Cloud SQL é o serviço de Banco de Dados SQL do GCP, permitindo a criação de bancos MySQL e PostgreSQL.

**APIs necessárias:** Cloud SQL Admin

### Cloud SQL Proxy 

O Cloud SQL Proxy permite o acesso a instâncias do Cloud SQL via um cliente local (como o Workbench) e também permite testar de forma local aplicações do Google App Engine. 

#### Instalação

Para instalar o SQL Proxy, rode os comandos abaixo:

```sh
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy
```

#### Executando

Para executar o Cloud SQL Proxy, é necessário a string de nome da instância do Cloud SQL, localizada ao clicar em uma instância na [Lista de Instâncias do Cloud SQL](https://console.cloud.google.com/sql/instances/). O formato dessa string é `<NOME DO PROJETO>:<REGIÃO DO PROJETO>:<NOME DA INSTÂNCIA>`. Após isso, existem 2 modos de funcionamento do proxy:

**Socket TCP:** Este método pode ser utilizado para conectar ao banco na nuvem utilizando o MySQL Workbench. Ele cria um Socket TCP que pode ser usado para conectar nas instâncias do Cloud SQL de forma 'tradicional'. Para utilizar este modo, basta rodar:

```sh
./cloud_sql_proxy -instances=<INSTANCE_CONNECTION_NAME>=tcp:3306
```

O banco de dados poderá então ser acessado utilizando o IP local (127.0.0.1 ou localhost), utilizando os mesmos usuário e senha definidos na configuração do banco na GCP. 

**Socket Unix:** Este método cria um socket UNIX em um diretório especificado pelo usuário. Este é o mesmo método utilizado internamente pelo App Engine, permitindo que as aplicações do App Engine sejam testadas em um ambiente local com mínimas alterações. Para utilizar este modo, primeiro crie um diretório que conterá o arquivo do socket (necessita de acesso root):

```sh
sudo mkdir /cloudsql
sudo chmod 777 /cloudsql
```

Após isso, execute o Proxy com:

```sh
./cloud_sql_proxy -dir=/cloudsql &
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