[Anterior](./01-ping-pong.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](./03-get.md)

# Criando tarefas

O primeiro passo para criar nossa tarefa será entender o que é uma tarefa. A ideia de uma tarefa é conter informações sobre ela e que outras pessoas do time tenham visibilidade do que se trata a tarefa. Assim vamos começar por modelar o domínio de entrada e saída. Usaremos a `struct` do Rust para modelar:

```rust
struct Task {
    is_done: bool,
    title: String
}

enum State {
    Todo,
    Doing,
    Done,
}

struct TodoCard {
    title: String,
    description: String,
    owner: Uuid,
    tasks: Vec<Task>,
    state: State
}
```

Assim, nossa struct principal é a `TodoCard`, que possui os campos `String` `title` e `description`, correspondentes ao título da tarefa e a sua descrição. Depois disso, podemos ver que existe um campo do tipo `Uuid` (inclua a crate `uuid` com as `features` `serde` e `v4` ativadas em seu `Cargo.toml`), que é um `owner`, ou seja, a pessoa dona da tarefa. Cada tarefa possui um conjunto de subtarefas a fazer, que podem estar completas ou não. Essas subtarefas são chamadas de `Task`, e é uma struct que possui um título, `title`, e um estado booleano que chamamos de `is_done`. Em seguida temos o estado da tarefa no fluxo de cards, `state`, que corresponde ao enum `State`, com os campos `Todo`, `Doing` e `Done`. O primeiro passo do serviço será algo bastante simples, receber um `POST` JSON com o `TodoCard` e respondermos um `TodoCardId`:

```rust
struct TodoCardId {
    id: Uuid
}
```

> **Uuid**
>
> A crate Uuid possui várias configurações, mas para o que vamos utilizar precisamos de compatibilidade com `Serde` e a versão 4. Serde para garantir que ela é serializável e desserializável para JSON, e versão 4, pois é o formato que vamos utilizar. Assim, essas configurações são adicionadas ao `[dependencies]` do `Cargo.toml` como `uuid = { version = "0.7", features = ["serde", "v4"] }`.

Um exemplo de `POST` em json de uma `TodoCard` seria:

```json
{
    "title": "This is a card",
    "description": "This is the description of the card",
    "owner": "ae75c4d8-5241-4f1c-8e85-ff380c041442",
    "tasks": [
        {
            "title": "title 1",
            "is_done": true
        },
        {
            "title": "title 2",
            "is_done": true
        },
        {
            "title": "title 3",
            "is_done": false
        }
    ],
    "state": "Doing"
}
```

Agora que modelamos nosso `TodoCard` podemos começar sua implementação com testes.

## Criando o primeiro teste de `TodoCard`

O novo teste envolve uma série de alterações no código, como criar um novo `scope` para rotas de `api`, enviar `payloads` e responder objetos JSON. Assim, a estratégia desse teste vai envolver enviar um `TodoCard` para ser criado e termos como resposta um JSON contendo o id desse `TodoCard`. Lembrando que agora que vamos manipular JSON, precisamos poder serializar e desserializar eles, e para isso devemos incluir a biblioteca `serde` no Cargo.toml:

```toml
// ...
[dependencies]
actix-web = "2.0"
actix-rt = "1.0"
uuid = { version = "0.7", features = ["serde", "v4"] }
serde = { version = "1.0.104", features = ["derive"] }
serde_json = "1.0.44"
serde_derive = "1.0.104"
num_cpus = "1.0"

[dev-dependencies]
bytes = "0.5.3"
actix-service = "1.0.5"
```

Assim, podemos escrever nosso teste sem conflito de dependências em `tests/todo_api_web/controller.rs`:

```rust
mod create_todo {
    use todo_server::todo_api_web::{controller::todo::create_todo, model::todo::TodoIdResponse};

    use actix_web::{http::header::CONTENT_TYPE, test, web, App, body};
    use serde_json::from_str;

    fn post_todo() -> String {
        String::from(
            "{
                \"title\": \"This is a card\",
                \"description\": \"This is the description of the card\",
                \"owner\": \"ae75c4d8-5241-4f1c-8e85-ff380c041442\",
                \"tasks\": [
                    {
                        \"title\": \"title 1\",
                        \"is_done\": true
                    },
                    {
                        \"title\": \"title 2\",
                        \"is_done\": true
                    },
                    {
                        \"title\": \"title 3\",
                        \"is_done\": false
                    }
                ],
                \"state\": \"Doing\"
            }",
        )
    }

    #[actix_web::test]
    async fn valid_todo_post() {
        let mut app = test::init_service(App::new().service(create_todo)).await;

        let req = test::TestRequest::post()
            .uri("/api/create")
            .insert_header((CONTENT_TYPE, ContentType::json()))
            .set_payload(post_todo().as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        let body = resp.into_body();
        let bytes = body::to_bytes(body).await.unwrap();
        let id = from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();
        assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());
    }
}
```

Duas características já se destacam, `controller::todo::create_todo` e `model::TodoIdResponse`, que correspondem ao recursos que de fato estão sendo testados. `TodoIdResponse` corresponde ao Json com o id de criação da `TodoCard`, sua definição é a seguinte:

```rust
#[derive(Serialize, Deserialize)]
pub struct TodoIdResponse {
    id: Uuid,
}
```

Perceba que utilizamos as macros de `Serialize, Deserialize` para sua fácil conversão entre JSON e String. Além disso, `TodoIdResponse` possui um campo `id` que é do tipo `Uuid`, um `Uuid` do tipo `v4`, conforme definimos no `Cargo.toml`. Agora temos também o controller `create_todo`, que receberá um `POST` do tipo JSON, fará sua inserção no banco de dados e retornará seu id. Felizmente, para este primeiro momento, não precisamos fazer a inserção no banco, pois o teste espera somente um tipo de retorno `id`. 

Outro ponto importante é o uso da biblioteca `use serde_json::from_str;`. Essa função em especial serve para converter uma `&str` em uma das structs serializáveis, conforme a linha `let id: TodoIdResponse = from_str(&String::from_utf8(resp.to_vec()).unwrap()).unwrap();`. Note que como a função não sabe para qual struct deve converter a resposta `resp`, tivemos de definir seu tipo na declaração do valor id, `id: TodoIdResponse`. O JSON do `payload` do `POST` está definido como uma string na função auxiliar de teste `post_todo`.

A seguir possuímos a definição do teste e o uso do runtime do actix, seguidos da definição do `App`, que vamos utilizar para mockar o serviço e suas rotas:

```rust
//definição do teste
#[actix_web::test]
async fn valid_todo_post() {
```

```rust
// Definição do App
let mut app = test::init_service(
    App::new().service(create_todo)
).await;
```

Note duas mudanças na definição do `App`: nossa rota possui um padrão diferente `/api/create` e o controller `create_todo` está sendo passado para um método `service()`. Outro detalhe é que estamos utilizando mais recursos na criação do request:

```rust
let req = test::TestRequest::post()
    .uri("/api/create")
    .insert_header((CONTENT_TYPE, ContentType::json()))
    .set_payload(post_todo().as_bytes().to_owned())
    .to_request();
```

Veja que `TestRequest` agora instancia um tipo `POST` antes de adicionar informações ao seu builder, `TestRequest::post()`. As duas outras mudanças são a adição das funções `header` e `set_payload`, `.header("Content-Type", ContentType::json()).set_payload(post_todo().as_bytes().to_owned())`. `header` define o tipo de conteúdo que estamos enviando e sua ausência nesse caso pode implicar em uma resposta com o status `400`. `set_payload` recebe um array de bytes com o conteúdo do `payload`, ou seja `post_todo`.

```rust
let resp = test::call_service(&mut app, req).await;
    let body = resp.into_body();
    let bytes = body::to_bytes(body).await.unwrap();
    let id = from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();
    assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());
```
Depois podemos ler a resposta normalmente, `let resp = test::call_service(&mut app, req).await;`, obter o body da response em bytes `let body = resp.into_body();let bytes = body::to_bytes(body).await.unwrap();` e transformar essa resposta em uma struct conhecida pelo serviço, `from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();`. O último passo é garantir que a resposta contendo o `TodoIdResponse` seja de fato um id válido e para isso utilizamos a macro `assert!` em `assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());`. Note a função auxiliar `get_id`, se nosso teste estivesse dentro do nosso módulo em vez de na pasta de testes de integração, seria possível anotar ela com `#[cfg(test)]` e economizar espaço no executável e tempo de compilação. Eu optei por deixá-la visível e testar o controller nos testes de integração, mas a escolha é sua:

```rust
impl TodoIdResponse {
    // #[cfg(test)]
    pub fn get_id(self) -> String {
        format!("{}", self.id)
    }
}
```

### Implementando o controller do teste anterior

Agora, com o teste implementado, precisamos entender quais são as coisas que necessitamos implementar:

1. Controller `create_todo`.
2. O controller recebe um Json do tipo `TodoCard` , que precisa ser deserializável com a macro `#[derive(Deserialize)]`.
3. Um struct `TodoIdResponse` que precisa ser serializável com `#[derive(Serialize)]`.

Como os itens 2 e 3 já foram mencionados na seção anterior, vou mostrar como eles ficaram com as macros de serialização e desserialização. Além disso, inclui a macro de `Debug`, pois pode ser útil durante o desenvolvimento, se você achar necessário retirá-la no futuro pode ajudar a economizar espaço do binário.

* Para utilizar as macros de serde, lembre-se de incluir `#[macro_use] extern crate serde;` em `lib.rs` e em `main.rs` (versões mais antigas do Rust).

```rust
// src/todo_api_web/model/mod.rs
use uuid::Uuid;

#[derive(Serialize, Deserialize, Debug)]
struct Task {
    is_done: bool,
    title: String,
}

#[derive(Serialize, Deserialize, Debug)]
enum State {
    Todo,
    Doing,
    Done,
}

#[derive(Serialize, Deserialize, Debug)]
pub struct TodoCard {
    title: String,
    description: String,
    owner: Uuid,
    tasks: Vec<Task>,
    state: State,
}

#[derive(Serialize, Deserialize)]
pub struct TodoIdResponse {
    id: Uuid,
}

impl TodoIdResponse {
    // #[cfg(test)]
    pub fn get_id(self) -> String {
        format!("{}", self.id)
    }
}
```

Para o item 1, `create_todo` controller, devemos novamente criar uma função `async`, que tem como tipo de resposta uma implementação da trait `Responder`, a `impl Responder`, como fizemos com `pong` e `readiness`:

```rust
// src/todo_api_web/controller/todo.rs
use crate::todo_api_web::model::todo::TodoIdResponse;
use actix_web::{ http::header::ContentType, post, web, HttpResponse, Responder};
use uuid::Uuid;

#[post("/api/create")]
pub async fn create_todo(_payload: web::Payload) -> impl Responder {
    let new_id = Uuid::new_v4();
    let str = serde_json::to_string(&TodoIdResponse::new(new_id)).unwrap();
    HttpResponse::Created()
        .content_type(ContentType::json())
        .body(str)
}
```

As primeiras coisas que podemos perceber são a create de `Uuid` para gerar novos `uuids` com `Uuid::new_v4()`, e os tipos de entrada e de saída, `TodoCard, TodoIdResponse`, respectivamente. O actix possui uma forma interna de desserializar objetos JSON que é definido no módulo `web` com `web::Json<T>` e é em `T` que vamos incluir nossa struct `TodoCard`. Veja que o tipo de retorno `TodoIdResponse` está sendo serializado pelo `serde_json` e retornado ao `body`. Note também que adicionamos o header `Content-type` através da função `.content_type(ContentType::json())`. Assim já seria suficiente para nosso teste passar, mas se quisermos testar essa rota com um `curl` é preciso adicionar ao `App` de `main.rs`:

```rust
HttpServer::new(|| {
    App::new()
        .service(readiness)
        .service(ping)
        .service(create_todo)
        .default_service(web::to(|| HttpResponse::NotFound()))
})
```

### Refatorando as rotas

No nosso teste anterior, percebemos que nossas rotas da `main.rs` são desconectadas das rotas do teste (`tests/todo_api_web/controller`), pois iniciamos um servidor de teste (`test::init_service`) que pode possuir uma rota aleatória, já que iniciamos um novo `App` dentro dele. Assim, basta direcionarmos a rota a um controller correto e fazer o request ser direcionado para essa rota que tudo ocorrerá bem. Para resolver isso, a sugestão é refatorarmos o `App` de forma que suas rotas sejam configuradas em um único lugar e possam ser utilizadas tanto na `main.rs` quanto nos testes. Para isso, vamos refatorar nosso `main` para extrair todo o `web::scope` de forma que as configurações venham de um módulo de rotas. Assim, devemos criar um módulo de rotas em `src/todo_api_web/routes.rs` e adicionar o seguinte código:

```rust
use actix_web::{web, HttpResponse};
use crate::todo_api_web::controller::{
    ping, readiness,
    todo::create_todo
};

pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
                .service(
                    web::scope("api/")
                        .route("create", web::post().to(create_todo))
                )
                .route("ping", web::get().to(pong))
                .route("~/ready", web::get().to(readiness))
                .route("", web::get().to(|| HttpResponse::NotFound()))
    );
}
```

Basicamente estamos extraindo todas as rotas para uma nova função que alterará o está do configuração do serviço dentro de  `App` com a função `config.service`. Isso impacta também nosso `main`, pois agora somente vamos precisar declarar a função `app_routes`:

```rust
// main.rs
mod todo_api_web;
use todo_api_web::routes::app_routes;

use actix_web::{App, HttpServer};
use num_cpus;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().configure(app_routes) 
    })
    .workers(num_cpus::get() + 2)
    .bind(("localhost", 4004))
    .unwrap()
    .run()
    .await
}
```

Agora podemos fazer a mesma refatoração nos testes, `tests/todo_api_web/controller`:

```rust
mod ping_readiness {
    use todo_server::todo_api_web::routes::app_routes;

    use actix_web::{body, http::StatusCode, test, web, App};

    #[actix_web::test]
    async fn test_ping_pong() {
        let mut app = test::init_service(App::new().configure(app_routes)).await;

        let req = test::TestRequest::get().uri("/ping").to_request();
        let resp = test::call_service(&mut app, req).await;
        let body = resp.into_body();
        let bytes = body::to_bytes(body).await.unwrap();

        assert_eq!(bytes, web::Bytes::from_static(b"pong"));
    }

    #[actix_web::test]
    async fn test_readiness() {
        let mut app = test::init_service(App::new().configure(app_routes)).await;
        let req = test::TestRequest::get().uri("/~/ready").to_request();
        let resp = test::call_service(&mut app, req).await;

        assert_eq!(resp.status(), StatusCode::ACCEPTED);
    }
}

mod create_todo {
    use todo_server::todo_api_web::{controller::todo::create_todo, model::todo::TodoIdResponse};

    use actix_web::{body, http::header::CONTENT_TYPE, test, web, App};
    use serde_json::from_str;

    fn post_todo() -> String {
        String::from(
            "{
                \"title\": \"This is a card\",
                \"description\": \"This is the description of the card\",
                \"owner\": \"ae75c4d8-5241-4f1c-8e85-ff380c041442\",
                \"tasks\": [
                    {
                        \"title\": \"title 1\",
                        \"is_done\": true
                    },
                    {
                        \"title\": \"title 2\",
                        \"is_done\": true
                    },
                    {
                        \"title\": \"title 3\",
                        \"is_done\": false
                    }
                ],
                \"state\": \"Doing\"
            }",
        )
    }

    #[actix_web::test]
    async fn valid_todo_post() {
        let mut app = test::init_service(App::new().service(create_todo)).await;

        let req = test::TestRequest::post()
            .uri("/api/create")
            .insert_header((CONTENT_TYPE, ContentType::json()))
            .set_payload(post_todo().as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        let body = resp.into_body();
        let bytes = body::to_bytes(body).await.unwrap();
        let id = from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();
        assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());
    }
}

```

Com estas alterações podemos perceber que um teste falha ao executarmos `cargo test`, esse é o teste `test todo_api_web::controller::ping_readiness::test_readiness_ok`, que falha com um `404`. Isso se deve ao fato de que a rota que estamos enviando o request de `readiness` estava errada esse tempo todo, pois escrevemos `/readiness`, enquanto a rota real é `/~/ready`:

```rust
#[get("/~/ready")]
pub async fn readiness() -> impl Responder {
    let process = std::process::Command::new("sh")
        .arg("-c")
        .arg("echo hello")
        .output();
    match process {
        Ok(_) => HttpResponse::Accepted(),
        Err(_) => HttpResponse::InternalServerError(),
    }
}
```

Nosso próximo passo é incluir nosso `TodoCard` em nossa base de dados.

## Configurando a base de dados

A base de dados que vamos utilizar agora é o DyanmoDB. O objetivo de utilizar essa base de dados é salvar as `TodoCards` para podermos buscá-las no futuro, assim o primeiro passo é configurar e modelar a base de dados para que nosso servidor a reconheça. A instância que vamos utilizar é derivada de um contêiner docker cuja imagem é `amazon/dynamodb-local` e pode ser executada com `docker run -p 8000:8000 amazon/dynamodb-local`. Observe que a porta que o DynamoDB está expondo é a `8000`. Eu gosto muito de utilizar Makefiles, pois eles facilitam a vida quando precisamos rodar vários comandos, especialmente em serviços diferentes. Assim, criei o seguinte Makefile para executar o DynamoDB:

```sh
db:
	docker run -p 8000:8000 amazon/dynamodb-local

```

### Escrevendo no banco de dados

Como essa primeira feature envolve exploração, primeiro vou apresentar a lógica de como fazemos para depois escrever os testes e generalizações. O próximo passo para termos a lógica do banco de dados é criar um novo módulo em `lib.rs` (e no `main.rs`) chamado `todo_api`, que por sua vez possuirá o módulo `db`, que vai gerenciar todas as relações com o DynamoDB. Antes de seguir com o servidor em si, vou comentar a atual função `main` e substituir por outra simples que sera descrita posteriormente, que utiliza somente o módulo `todo_api` para executar a criação de uma `TodoCard` no banco de dados, depois disso podemos conectar as partes novamente.

Para podermos nos comunicar facilmente com o DynamoDB em Rust, existem as bibliotecas oferecidas pela AWS, chamadas `aws-sdk-dynamodb` e `aws-config`. Basta adicioná-las às dependências no `Cargo.toml`. (Atualmente a sdk Rust da AWS está em Developer Preview e não deve ser usada em produção).

```toml
[dependencies]
actix-web = "2.0"
actix-rt = "1.0"
actix-http = "1.0.1"
uuid = { version = "0.7", features = ["serde", "v4"] }
serde = { version = "1.0.104", features = ["derive"] }
serde_json = "1.0.44"
serde_derive = "1.0.104"
num_cpus = "1.0"
aws-config = "0.49.0"
aws-sdk-dynamodb = "0.19.0"

[dev-dependencies]
bytes = "0.5.3"
actix-service = "1.0.5"
```

Com a biblioteca `aws-sdk-dynamodb` disponível, podemos começar a pensar em como nos comunicar com o DynamoDB. Podemos fazer isso adicionando um módulo `helpers` dentro de `todo_api/db` e criando uma função que retorna o cliente:

Nota: Para utilizar o dynamodb localmente, deve ser criado um arquivo de configuração contendo uma região e credenciais (que não precisam ser validas) da AWS em `~/.aws/config` contendo:

```code
[profile localstack]
region=us-east-1
aws_access_key_id=AKIDLOCALSTACK
aws_secret_access_key=localstacksecret
```

Agora precisamos criar uma tabela, para nosso caso não vou utilizar uma migracão pois acredito que em um cenário real este banco de dados será configurado por outro serviço, algo mais próximo a um ambiente cloud. Assim, vamos criar a função `create_table` em `todo_api/db/helpers.rs`, que fará a configuração da tabela para nós:

```rust
use actix_web::http::Uri;
use aws_sdk_dynamodb::{
    model::{
        AttributeDefinition, KeySchemaElement, KeyType, ProvisionedThroughput, ScalarAttributeType,
    },
    Client, Endpoint,
};

pub static TODO_CARD_TABLE: &str = "TODO_CARDS";

pub async fn create_table() {
    let config = aws_config::load_from_env().await;
    let dynamodb_local_config = aws_sdk_dynamodb::config::Builder::from(&config)
        .endpoint_resolver(
            Endpoint::immutable(Uri::from_static("http://localhost:8000")),
        )
        .build();

    let client = Client::from_conf(dynamodb_local_config);

    let table_name = TODO_CARD_TABLE.to_string();
    let ad = AttributeDefinition::builder()
        .attribute_name("id")
        .attribute_type(ScalarAttributeType::S)
        .build();

    let ks = KeySchemaElement::builder()
        .attribute_name("id")
        .key_type(KeyType::Hash)
        .build();

    let pt = ProvisionedThroughput::builder()
        .read_capacity_units(1)
        .write_capacity_units(1)
        .build();

    match client
        .create_table()
        .table_name(table_name)
        .key_schema(ks)
        .attribute_definitions(ad)
        .provisioned_throughput(pt)
        .send()
        .await
    {
        Ok(output) => {
            println!("Output: {:?}", output);    
        }
        Err(error) => {
            println!("Error: {:?}", error);
        }
    }
}
```
Para testar precisamos executar o comando `make db`. Iremos seguir a [documentação](https://docs.aws.amazon.com/sdk-for-rust/latest/dg/dynamodb-local.html) do aws-sdk-rust para configurar o DynamoDB local. Em outro terminal, precisamos setar uma variavel de ambiente para a `aws-config` utilizar o profile `localstack` que adicionamos em `~/.aws/config`, para isso usamos `export AWS_PROFILE=localstack` (no osx ou linux). Depois atualizamos a main com o cdigo abaixo e executamos em seguida, no mesmo terminal aonde setamos a variavel de ambiente `AWS_PROFILE` executamos `cargo build && cargo run`. 

```rust
// main.rs
use todo_api::db::helpers::create_table;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    create_table();
}
```
Executando esta sequência de comandos, recebemos o seguinte output:

```rust
Output: CreateTableOutput CreateTableOutput {
	table_description: Some(TableDescription {
		attribute_definitions: Some([AttributeDefinition {
			attribute_name: Some("id"),
			attribute_type: Some(S)
		}]),
		table_name: Some("TODO_CARDS"),
		key_schema: Some([KeySchemaElement {
			attribute_name: Some("id"),
			key_type: Some(Hash)
		}]),
		table_status: Some(Active),
		creation_date_time: Some(DateTime {
			seconds: 1665206254,
			subsecond_nanos: 461999893
		}),
		provisioned_throughput: Some(ProvisionedThroughputDescription {
			last_increase_date_time: Some(DateTime {
				seconds: 0,
				subsecond_nanos: 0
			}),
			last_decrease_date_time: Some(DateTime {
				seconds: 0,
				subsecond_nanos: 0
			}),
			number_of_decreases_today: Some(0),
			read_capacity_units: Some(1),
			write_capacity_units: Some(1)
		}),
		table_size_bytes: 0,
		item_count: 0,
		table_arn: Some("arn:aws:dynamodb:ddblocal:000000000000:table/TODO_CARDS"),
		table_id: None,
		billing_mode_summary: None,
		local_secondary_indexes: None,
		global_secondary_indexes: None,
		stream_specification: None,
		latest_stream_label: None,
		latest_stream_arn: None,
		global_table_version: None,
		replicas: None,
		restore_summary: None,
		sse_description: None,
		archival_summary: None,
		table_class_summary: None
	})
}
```

Tabela criada! Mas se executarmos `cargo run` de novo, receberemos um erro dizendo que não é possível criar uma tabela que já existe:

```rust
Error: ServiceError {
	err: CreateTableError {
		kind: ResourceInUseException(ResourceInUseException {
			message: Some("Cannot create preexisting table")
		}),
		meta: Error {
			code: Some("ResourceInUseException"),
			message: Some("Cannot create preexisting table"),
			request_id: Some("543a624e-7f21-4dd2-80f2-520ae078152b"),
			extras: {}
		}
	},
	raw: Response {
		inner: Response {
			status: 400,
			version: HTTP / 1.1,
			headers: {
				"date": "Sat, 08 Oct 2022 05:33:45 GMT",
				"content-type": "application/x-amz-json-1.0",
				"x-amzn-requestid": "543a624e-7f21-4dd2-80f2-520ae078152b",
				"content-length": "112",
				"server": "Jetty(9.4.48.v20220622)"
			},
			body: SdkBody {
				inner: Once(Some(b "{\"__type\":\"com.amazonaws.dynamodb.v20120810#ResourceInUseException\",\"Message\":\"Cannot create preexisting table\"}")),
				retryable: true
			}
		},
		properties: SharedPropertyBag(Mutex {
			data: PropertyBag,
			poisoned: false,
			..
		})
	}
}
```

Para corrigir esse erro, sugiro modificar o método `create_table` para verificar se existem tabelas com a função `client.list_tables().send()`. Para isso, fazemos a seguinte modificação:

```rust
use actix_web::http::Uri;
use aws_sdk_dynamodb::{
    model::{
        AttributeDefinition, KeySchemaElement, KeyType, ProvisionedThroughput, ScalarAttributeType,
    },
    Client, Endpoint
};

pub static TODO_CARD_TABLE: &str = "TODO_CARDS";

pub async fn create_table() {
    let config = aws_config::load_from_env().await;
    let dynamodb_local_config = aws_sdk_dynamodb::config::Builder::from(&config)
        .endpoint_resolver(
            Endpoint::immutable(Uri::from_static("http://localhost:8000")),
        )
        .build();

    let client = Client::from_conf(dynamodb_local_config);

    match client.list_tables().send().await {
        Ok(list) => {
            match list.table_names {
                Some(table_vec) => {
                    if table_vec.len() > 0 {
                        println!("Error: {:?}", "Table already exists");
                    } else {
                        create_table_input(&client).await
                    }
                }
                None => create_table_input(&client).await,
            };
        }
        Err(_) => {
            create_table_input(&client).await;
        }
    }
}

fn build_key_schema() -> KeySchemaElement {
    KeySchemaElement::builder()
        .attribute_name("id")
        .key_type(KeyType::Hash)
        .build()
}

fn build_provisioned_throughput() -> ProvisionedThroughput {
    ProvisionedThroughput::builder()
        .read_capacity_units(1)
        .write_capacity_units(1)
        .build()
}

fn build_attribute_definition() -> AttributeDefinition {
    AttributeDefinition::builder()
        .attribute_name("id")
        .attribute_type(ScalarAttributeType::S)
        .build()
}

async fn create_table_input(client: &Client) {
    let table_name = TODO_CARD_TABLE.to_string();
    let ad = build_attribute_definition();
    let ks = build_key_schema();
    let pt = build_provisioned_throughput();

    match client
        .create_table()
        .table_name(table_name)
        .key_schema(ks)
        .attribute_definitions(ad)
        .provisioned_throughput(pt)
        .send()
        .await
    {
        Ok(output) => {
            println!("Output: {:?}", output);
        }
        Err(error) => {
            println!("Error: {:?}", error);
        }
    }
}
```

Note que, quando verificamos as listas existentes na tabela, surgiram várias situações possíveis e para facilitar a criação da tabela, extraímos sua lógica para `create_table_input`. A primeira situação é `Err`, que possivelmente representa algum problema de listagem de tabelas na base, indicando ausência de tabelas, que nos permite criar tabelas. O segundo caso, dentro do `Ok` é um `None`, que pode significar os mais diversos problemas. Depois disso obtemos a listagem em `Some`, mas esta listagem pode estar vazia, sendo um caso para criar tabela, o `else`, e se a listagem for maior que zero, não criamos a tabela.

### Inserindo conteúdo na tabela

Para inserirmos a tabela, vamos precisar de uma struct de `rusoto_dynamo` chamada `PutItemInput`, que nos permitirá inserir o JSON que recebemos na tabela, porém o JSON que recebemos em `TodoCard` não possui o id do card. Para podermos utilizar o `PutItemInput` como definimos na tabela, vamos criar um `model` que possua um id.

```rust
// src/todo_api/model/mod.rs
use rusoto_dynamodb::AttributeValue;
use std::collections::HashMap;
use uuid::Uuid;

#[derive(Debug, Clone)]
struct TaskDb {
    is_done: bool,
    title: String,
}

#[derive(Debug, Clone)]
enum StateDb {
    Todo,
    Doing,
    Done,
}

#[derive(Debug, Clone)]
pub struct TodoCardDb {
    id: Uuid,
    title: String,
    description: String,
    owner: Uuid,
    tasks: Vec<TaskDb>,
    state: StateDb,
}
```

Vamos criar uma função que permita transformar um `TodoCard` em um `TodoCardDb`, em `src/todo_api/model/mod.rs`:

```rust
use crate::todo_api_web::model::{State, TodoCard};
use actix_web::web;

impl TodoCardDb {
    pub fn new(card: web::Json<TodoCard>) -> Self {
        Self {
            id: Uuid::new_v4(),
            title: card.title.clone(),
            description: card.description.clone(),
            owner: card.owner,
            tasks: card
                .tasks
                .iter()
                .map(|t| TaskDb {
                    is_done: t.is_done,
                    title: t.title.clone(),
                })
                .collect::<Vec<TaskDb>>(),
            state: match card.state {
                State::Doing => StateDb::Doing,
                State::Done => StateDb::Done,
                State::Todo => StateDb::Todo,
            },
        }
    }
}
```

Agora, podemos fazer nosso `controller` momentaneamente gerenciar todas as ações com o banco de dados:

```rust
use actix_web::{HttpResponse, web, Responder};
use uuid::Uuid;
use crate::todo_api_web::model::{TodoCard, TodoIdResponse};
use crate::todo_api::model::{TodoCardDb};

pub async fn create_todo(info: web::Json<TodoCard>) -> impl Responder {
    let todo_card = TodoCardDb::new(info);
    let client = get_client().await;
    match put_todo(&client, todo_card).await {
        None => HttpResponse::BadRequest().body("Failed to create todo card"),
        Some(id) => HttpResponse::Created()
            .content_type(ContentType::json())
            .body(serde_json::to_string(&TodoIdResponse::new(id)).expect("Failed to serialize todo card"))
    }
}

/// A partir daqui vamos extrair logo mais
use aws_sdk_dynamodb::{Client};
use crate::{
    todo_api::db::helpers::{TODO_CARD_TABLE},
};
use super::helpers::get_client;

pub async fn put_todo(client: &Client, todo_card: TodoCardDb) ->  Option<uuid::Uuid> {
    match client.put_item()
    .table_name(TODO_CARD_TABLE.to_string())
    .set_item(Some(todo_card.clone().into()))
    .send()
    .await {
        Ok(_) => {
            Some(todo_card.id)
        },
        Err(_) => {
            None
        }
    }
}
```

Veja que nosso controller ficou muito mais funcional agora. Ele recebe um JSON do tipo `TodoCard`, transforma esse JSON em um `TodoCardDb` e envia para a função `put_todo` inserir no banco de dados. Caso ocorra algum problema com a inserção fazemos pattern matching com o `None` e retornamos algo como `HttpResponse::BadRequest()` ou `HttpResponse::InternalServerError()`, mas caso o retorno seja um id em `Some`, retornamos um JSON contendo `TodoIdResponse`. Note que foi necessário adicionar a função `body` ao `HttpResponse::BadRequest()` para garantir que os dois pattern matchings tivessem o mesmo tipo de retorno `Response`, em vez de `ResponseBuilder`. 

Se você estiver utilizando o `rust-analyzer` do rust, vai perceber que o `into` de `item: todo_card.clone().into(),` está destacado, isso se deve ao fato de que precisamos implementar a função `into` para o tipo `TodoCardDB` de forma que retorne `HashMap<String, AttributeValue>`. Para isso, utilizamos a declaração `impl Into<HashMap<String, AttributeValue>> for TodoCardDb` com a seguinte implementação:

```rust
// src/todo_api/model/mod.rs
use std::collections::HashMap;

impl Into<HashMap<String, AttributeValue>> for TodoCardDb {
    fn into(self) -> HashMap<String, AttributeValue> {
        let mut todo_card = HashMap::new();
        todo_card.insert("id".to_string(), val!(S => self.id.to_string()));
        todo_card.insert("title".to_string(), val!(S => self.title));
        todo_card.insert("description".to_string(), val!(S => self.description));
        todo_card.insert("owner".to_string(), val!(S => self.owner.to_string()));
        todo_card.insert("state".to_string(), val!(S => self.state.to_string()));
        todo_card.insert("tasks".to_string(), val!(L => task_to_db_val(self.tasks)));
        todo_card
    }
}
```

Se você está utilizando `rls` vai perceber que o `state.to_string()` e o `task_to_db_val` estão destacados como errados, assim como a macro `val!`. Vamos falar do `val!` logo, mas primeiro vamos entender como funciona a criação do tipo `AttributeValue` para ser inserido dentro do banco. A função `into` espera como retorno um tipo `HashMap<String, AttributeValue>`, no qual `AttributeValue` é uma struct com a seguinte estrutura:

```rust
pub struct AttributeValue {
    pub b: Option<Bytes>,
    pub bool: Option<bool>,
    pub bs: Option<Vec<Bytes>>,
    pub l: Option<Vec<AttributeValue>>,
    pub m: Option<HashMap<String, AttributeValue>>,
    pub n: Option<String>,
    pub ns: Option<Vec<String>>,
    pub null: Option<bool>,
    pub s: Option<String>,
    pub ss: Option<Vec<String>>,
}
```

> `AttributeValue`
>
> Os tipos `T` dentro do `Option<T>` são os tipos possíveis dentro do DynamoDB. Veja que alguns tipos são bem fáceis de perceber como `bool`, `Vec<AttributeValue>` e `HashMap<String, AttributeValue>`, isto é, um valor booleano, um vetor de atributos do dynamo e um mapa com keys strings e valores como atributos, respectivamente. Outros valores podem ser confusos, como as chaves `s`, `ss`, `n` e `ns`. As chaves dos tipos `b` e `bs` são para valores binários como `"B": "dGhpcyB0ZXh0IGlzIGJhc2U2NC1lbmNvZGVk"`, além disso o tipo `n` serve para representar um tipo numérico, enquanto o tipo `s` serve para tipos String. Os tipos `ss` e `ns` são as versões vetores de `s` e de `n`, respectivamente.

Para resovermos a falha de compilação em `state.to_string()` precisamos implementar a trait `std::fmt::Display` que nos permite transformar o valor de `state` em uma String:

```rust
impl std::fmt::Display for StateDb {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{:?}", self)
    }
}
```

Agora vamos verificar a função `task_to_db_val`, cujo objetivo é transformar um vetor do tipo `TaskDb` em um vetor de `AttributeValue`. Essa transformação nos permite inserir as `tasks` como um único campo contendo um vetor de objetos, como se diria na linguagem JSON, `TaskDB`. A função `task_to_db_val` é bastante simples, pois recebe uma `tasks` do tipo `Vec<TaskDb>` e aplica um mapa sobre cada `TaskDb` para substituí-las por um `AttributeValue` da chave `m`, `Option<HashMap<String, AttributeValue>>`, e depois coleciona todos esses `Option<HashMap<String, AttributeValue>>` em um vetor `Vec<AttributeValue>`:

```rust
fn task_to_db_val(tasks: Vec<TaskDb>) -> Vec<AttributeValue> {
    tasks
        .iter()
        .map(|t| {
            let mut tasks_hash = HashMap::new();
            tasks_hash.insert("title".to_string(), val!(S => t.title.clone()));
            tasks_hash.insert("is_done".to_string(), val!(B => t.is_done));
            val!(M => tasks_hash)
        })
        .collect::<Vec<AttributeValue>>()
}
```

Ainda falta falarmos da `val!`. `val!` é uma macro criada para transformar os valores de nossa struct em valores do DynamoDB. Inseri essa macro em um novo módulo chamado `adapter`:

```rust
// src/todo_api/adapter/mod.rs
#[macro_export]
macro_rules! val {
    (B => $bval:expr) => {{
        AttributeValue::Bool($bval)
    }};
    (L => $val:expr) => {{
        AttributeValue::L($val)
    }};
    (S => $val:expr) => {{
        AttributeValue::S($val)
    }};
    (M => $val:expr) => {{
        AttributeValue::M($val)
    }};
}
```

Para que essa macro esteja disponível dentro do módulo `todo_api`, precisamos utilizar `#[macro_use]` na declaração dos módulos:

```rust
#[macro_use]
pub mod adapter;
pub mod db;
pub mod model;
```

Agora tudo deve estar funcionando. Podemos executar `make db` e `cargo run` para fazer um `curl` em `http://localhost:4000/api/create` com o seguinte JSON:

```json
{
	"title": "title",
	"description": "descrition",
	"state": "Done",
	"owner": "90e700b0-2b9b-4c74-9285-f5fc94764995",
	"tasks": [
        {
			"is_done": true,
			"title": "blob"
			
		}
    ]
}
```

E vamos receber um `Uuid` como resposta e o status `201`:

```json
{
    "id": "ae1cb12c-6c67-4337-bd7b-b557d7568c60"
}
```

### Organizando nosso código

Nosso controller possui um conjunto de códigos que não fazem sentido dentro do contexto de controller, no caso a função `put_todo`. A primeira coisa que vamos fazer é criar um módulo `todo` dentro de `todo_api/db` que conterá toda a lógica de banco de dados para o `todo`:

```rust
use super::helpers::get_client;
use crate::todo_api::db::helpers::TODO_CARD_TABLE;
use aws_sdk_dynamodb::Client;

pub async fn put_todo(client: &Client, todo_card: TodoCardDb) -> Option<uuid::Uuid> {
    match client
        .put_item()
        .table_name(TODO_CARD_TABLE.to_string())
        .set_item(Some(todo_card.clone().into()))
        .send()
        .await
    {
        Ok(_) => Some(todo_card.id),
        Err(_) => None,
    }
}

```

E agora podemos simplificar muito nosso controller com:

```rust
use crate::todo_api::db:: {helpers::get_client, todo::put_todo};
use crate::todo_api::model::TodoCardDb;
use crate::todo_api_web::model::todo::{TodoCard, TodoIdResponse};
use actix_web::{http::header::ContentType, post, web, HttpResponse, Responder};

#[post("/api/create")]
pub async fn create_todo(info: web::Json<TodoCard>) -> impl Responder {
    let todo_card = TodoCardDb::new(info);
    let client = get_client().await;
    match put_todo(&client, todo_card).await {
        None => HttpResponse::BadRequest().body("Failed to create todo card"),
        Some(id) => HttpResponse::Created()
        .content_type(ContentType::json())
            .body(
                serde_json::to_string(&TodoIdResponse::new(id))
                    .expect("Failed to serialize todo card"),
            ),
    }
}
```

Note que para declarar todos os módulos internos utilizei o `use crate::{// ...}`, pois ajuda na organização. Além disso, na minha opinião, a função `new` de `TodoCardDb` é um adapter e pode estar mal localizada. Uma possível solução para isso seria mover e renomear a função `new` para o módulo adapter com nome de `todo_json_to_db`, mas isso implicaria em tornar todos os campos de `TodoCardDb` públicos, assim como de `TaskDb`. Por isso, essa parte da refatoração fica a seu critério de estilo, mas vou fazer para exemplificar:

```rust
// src/todo_api/adapter/mod.rs
// ...
use actix_web::web;
use uuid::Uuid;
use crate::{
    todo_api_web::model::{State, TodoCard},
    todo_api::model::{StateDb, TodoCardDb, TaskDb}
};

pub fn todo_json_to_db(card: web::Json<TodoCard>) -> TodoCardDb {
    TodoCardDb {
        id: Uuid::new_v4(),
        title: card.title.clone(),
        description: card.description.clone(),
        owner: card.owner,
        tasks: card
            .tasks
            .iter()
            .map(|t| TaskDb {
                is_done: t.is_done,
                title: t.title.clone(),
            })
            .collect::<Vec<TaskDb>>(),
        state: match card.state {
            State::Doing => StateDb::Doing,
            State::Done => StateDb::Done,
            State::Todo => StateDb::Todo,
        },
    }
}
```

A compilação falha pois `StateDB` e `TaskDB` são privados, assim como quase todos campos de `TodoCardDb`, para isso modificamos o módulo `todo_api/model` para:

```rust
// ...
#[derive(Debug, Clone)]
pub struct TaskDb {
    pub is_done: bool,
    pub title: String,
}

#[derive(Debug, Clone)]
pub enum StateDb {
    Todo,
    Doing,
    Done,
}

#[derive(Debug, Clone)]
pub struct TodoCardDb {
    pub id: Uuid,
    pub title: String,
    pub description: String,
    pub owner: Uuid,
    pub tasks: Vec<TaskDb>,
    pub state: StateDb,
}

impl TodoCardDb {
    #[allow(dead_code)]
    pub fn get_id(self) -> Uuid {
        self.id
    }
}
// ...
```

Também precisamos mudar o controller para utilizar nossa nova função:

```rust
use actix_web::{HttpResponse, web, Responder};
use crate::{
    todo_api::{
        db::todo::put_todo,
        adapter
    },
    todo_api_web::model::{TodoCard, TodoIdResponse}
};


pub async fn create_todo(info: web::Json<TodoCard>) -> impl Responder {
    let todo_card = adapter::todo_json_to_db(info);

    match put_todo(todo_card) {
        None => HttpResponse::BadRequest().body("Failed to create todo card"),
        Some(id) => HttpResponse::Created()
            .content_type(ContentType::json())
            .body(serde_json::to_string(&TodoIdResponse::new(id)).expect("Failed to serialize todo card"))
    }
}
```

Uma última refatoração que podemos fazer é a função `task_to_db_val`, já que sua função é essencialmente transformar `TaskDb` em um tipo `AttributeValue`. Assim, podemos implementar uma função que faça isso com `TaskDb`:

```rust

impl Into<HashMap<String, AttributeValue>> for TodoCardDb {
    fn into(self) -> HashMap<String, AttributeValue> {
        let mut todo_card = HashMap::new();
        todo_card.insert("id".to_string(), val!(S => self.id.to_string()));
        todo_card.insert("title".to_string(), val!(S => self.title));
        todo_card.insert("description".to_string(), val!(S => self.description));
        todo_card.insert("owner".to_string(), val!(S => self.owner.to_string()));
        todo_card.insert("state".to_string(), val!(S => self.state.to_string()));
        todo_card.insert("tasks".to_string(), 
            val!(L => self.tasks.into_iter().map(|t| t.to_db_val()).collect::<Vec<AttributeValue>>()));
        todo_card
    }
}

impl TaskDb {
    fn to_db_val(self) -> AttributeValue {
        let mut tasks_hash = HashMap::new();
            tasks_hash.insert("title".to_string(), val!(S => self.title.clone()));
            tasks_hash.insert("is_done".to_string(), val!(B => self.is_done));
            val!(M => tasks_hash)
    }
}
```

Agora faltam alguns testes.

## Aplicando testes a nosso endpoint

Creio que uma boa abordagem agora seja começar pelos testes mais unitários, por isso vamos começar pelo adapter. Nosso primeiro teste será com a função `converts_json_to_db`:

```rust
#[cfg(test)]
mod test {
    use std::collections::HashMap;

    use super::*;
    use crate::{
        todo_api::model::{StateDb, TaskDb, TodoCardDb},
        todo_api_web::model::todo::{State, Task, TodoCard},
    };
    use actix_web::web::Json;

    #[test]
    fn converts_json_to_db() {
        let id = uuid::Uuid::new_v4();
        let owner = uuid::Uuid::new_v4();
        let json = Json(TodoCard {
            title: "title".to_string(),
            description: "description".to_string(),
            owner: owner,
            state: State::Done,
            tasks: vec![Task {
                is_done: true,
                title: "title".to_string(),
            }],
        });
        let expected = TodoCardDb {
            id: id,
            title: "title".to_string(),
            description: "description".to_string(),
            owner: owner,
            state: StateDb::Done,
            tasks: vec![TaskDb {
                is_done: true,
                title: "title".to_string(),
            }],
        };
        assert_eq!(todo_json_to_db(json, id), expected);
    }
}
```

Note que, para facilitar a testabilidade, mudamos a assinatura da função para receber um `id`, `todo_json_to_db(json, id)`. Isso se deve ao fato de que gerar id randomicamente não ajuda os testes e testar campo a campo não parece uma boa solução. Além disso, adicionamos a macro `PartialEq` nas structs `StateDb`, `TaskDb` e `TodoCardDb` para fins de comparabilidade. Agora precisamos testar a função `to_db_val` de `TaskDb`:

```rust
#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn task_db_to_db_val() {
        let actual = TaskDb {
            title: "blob".to_string(),
            is_done: true,
        }
        .to_db_val();
        let mut tasks_hash = HashMap::new();
        tasks_hash.insert("title".to_string(), val!(S => "blob".to_string()));
        tasks_hash.insert("is_done".to_string(), val!(B => true));
        let expected = val!(M => tasks_hash);
        assert_eq!(actual, expected);
    }
}
```

A lógica do teste `task_db_to_db_val` é basicamente a mesma que a implementação da função, mas já vale como um simples teste unitário. Agora podemos testar a função `into`, que também teria a mesma implementação da própria função, note que estamos utilizando apenas um id:

```rust
#[test]
    fn todo_card_db_to_db_val() {
        let id = uuid::Uuid::new_v4();
        let actual: HashMap<String, AttributeValue> = TodoCardDb {
            id: id,
            title: "title".to_string(),
            description: "description".to_string(),
            owner: id,
            state: StateDb::Done,
            tasks: vec![TaskDb {
                is_done: true,
                title: "title".to_string(),
            }],
        }
        .into();
        let mut expected = HashMap::new();
        expected.insert("id".to_string(), val!(S => id.to_string()));
        expected.insert("title".to_string(), val!(S => "title".to_string()));
        expected.insert(
            "description".to_string(),
            val!(S => "description".to_string()),
        );
        expected.insert("owner".to_string(), val!(S => id.to_string()));
        expected.insert("state".to_string(), val!(S => StateDb::Done.to_string()));
        expected.insert(
            "tasks".to_string(),
            val!(L => vec![TaskDb {is_done: true, title: "title".to_string()}.to_db_val()]),
        );
        assert_eq!(actual, expected);
    }
```

Se executarmos `cargo test` enquanto o `make db` roda, teremos duas situações: uma em que a base de dados já está configurada e tudo ocorre normalmente e outra em que ela não está configurada e o teste falha. Para resolvermos esse problema, bastaria adicionar o `create_table` ao cenário de teste assim:

```rust
 #[actix_web::test]
    async fn valid_todo_post() {
        let mut app = test::init_service(App::new().configure(app_routes)).await;
        let req = test::TestRequest::post()
            .uri("/api/create")
            .insert_header((CONTENT_TYPE, ContentType::json()))
            .set_payload(read_json("post_todo.json").as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        let body = resp.into_body();
        let bytes = body::to_bytes(body).await.unwrap();
        let id = from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();
        assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());
    }
```

É bem claro para mim que um teste que precisa executar o contêiner do banco de dados para passar é bastante frágil. Assim vamos precisar fazer algumas modificações para tornar o teste passável. A mudança que vamos fazer é, na minha opinião, uma forma mais elegante de fazer mocks em rust, pois ela não necessita criar uma trait e uma struct para mockar uma função específica, basta definirmos que para modo de compilação em test, `#[cfg(test)]`, a função terá outro comportamento, geralmente evitando efeitos colaterais com base de dados. Agora, o que vai mudar é que nosso teste de controller deixará de estar presente na pasta `tests` e passará a ser um módulo `#[cfg(test)]` junto ao controller:

```rust
#[cfg(test)]
mod create_todo {
    use crate::todo_api_web::{
        model::TodoIdResponse,
        routes::app_routes
    };

    use actix_web::{
        test, App,
    };
    use serde_json::from_str;

    fn post_todo() -> String {
        // ...
    }

     #[actix_web::test]
    async fn valid_todo_post() {
        let mut app = test::init_service(App::new().configure(app_routes)).await;
        let req = test::TestRequest::post()
            .uri("/api/create")
            .insert_header((CONTENT_TYPE, ContentType::json()))
            .set_payload(read_json("post_todo.json").as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        let body = resp.into_body();
        let bytes = body::to_bytes(body).await.unwrap();
        let id = from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();
        assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());
    }
}
```

Assim, agora precisamos fazer com que nossa interação com o banco de dados seja "mockada", para isso reescrevi o módulo `src/todo_api/db/todo.rs` para conter duas formas de compilacão "com testes" e  "sem testes":

```rust
// ...
#[cfg(not(test))]
pub async fn put_todo(client: &Client, todo_card: TodoCardDb) -> Option<uuid::Uuid> {
    use crate::todo_api::db::helpers::TODO_CARD_TABLE;

    match client
        .put_item()
        .table_name(TODO_CARD_TABLE.to_string())
        .set_item(Some(todo_card.clone().into()))
        .send()
        .await
    {
        Ok(_) => Some(todo_card.id),
        Err(e) => {
            println!("{:?}", e);
            None
        }
    }
}

#[cfg(test)]
pub async fn put_todo(_client: &Client, todo_card: TodoCardDb) -> Option<uuid::Uuid> {
    Some(todo_card.id)
}
```

Veja que `put_todo` com `cfg(test)` ativado pula a etapa `match client.put_item().table_name(TODO_CARD_TABLE.to_string()).set_item(Some(todo_card.clone().into())).send().await` e simplesmente retorna  um `Option<Uuid>`. 

Outro modo de fazer esse teste, utilizando `cfg`, é utilizar `features`, mas por ser um pouco mais sensível deixei para apresentar depois. Neste repositório, vamos utilizar `features` para testar os controllers, o que deixará o código mais limpo, porém mais difícil de gerenciar, podendo fazer com que uma feature indesejada suba para a produção. Assim, recomendo fortemente que os builds de produção utilizem a flag `--release` e que os `cfg` mapeie corretamente isso. Para utilizar essa feature, uma boa prática é adicioná-la ao campo `[features]` do `Cargo.toml`:

```toml
[package]
name = "todo-server"
version = "0.1.0"
authors = ["Julia Naomi <jnboeira@outlook.com>"]
edition = "2018"

[features]
dynamo = []

// ...
```

Além disso, precisamos gerar a nova função, muito semelhante ao `cfg(test)` de antes:

```rust
use crate::todo_api::model::TodoCardDb;
use aws_sdk_dynamodb::Client;

#[cfg(feature = "dynamo")]
pub async fn put_todo(client: &Client, todo_card: TodoCardDb) -> Option<uuid::Uuid> {
    use crate::todo_api::db::helpers::TODO_CARD_TABLE;

    match client
        .put_item()
        .table_name(TODO_CARD_TABLE.to_string())
        .set_item(Some(todo_card.clone().into()))
        .send()
        .await
    {
        Ok(_) => Some(todo_card.id),
        Err(e) => {
            println!("{:?}", e);
            None
        }
    }
}

#[cfg(not(feature = "dynamo"))]
pub async fn put_todo(_client: &Client, todo_card: TodoCardDb) -> Option<uuid::Uuid> {
    Some(todo_card.id)
}
```

E movemos novamente nosso teste para a pasta `tests`. Para executar todos os testes corretamente usamos `cargo teste --features "dynamo"`, é sempre bom adicionar este comando a um Makefile.

```sh
db:
	docker run -p 8000:8000 amazon/dynamodb-local

test:
	cargo test --features "dynamo"
```

O último passo para nossos testes é gerar ums função de teste que nos permita retirar a grosseria que é a função de teste `post_todo`. Assim, faremos uma função que le um arquivo `json` e retorna uma string contendo seu conteúdo. Vamos chamá-la de `read_json` e vai receber como argumento uma string com o nome do arquivo. A primeira mudança que faremos é adicionar `mod helpers` no arquivo `tests/lib.rs`. Depois vamos criar o módulo `tests/helpers.rs` e adicionar a função `read_json`:

```rust
use std::fs::File;
use std::io::Read;

pub fn read_json(file: &str) -> String {
    let path = String::from("dev-resources/") + file;
    let mut file = File::open(&path).unwrap();
    let mut data = String::new();
    file.read_to_string(&mut data).unwrap();
    data
}
```

Com a função `read_json` pronta, podemos adicionar o Json `post_todo.json` na pasta (que vamos criar junto) `dev-resources` do projeto:

```json
{
    "title": "This is a card",
    "description": "This is the description of the card",
    "owner": "ae75c4d8-5241-4f1c-8e85-ff380c041442",
    "tasks": [
        {
            "title": "title 1",
            "is_done": true
        },
        {
            "title": "title 2",
            "is_done": true
        },
        {
            "title": "title 3",
            "is_done": false
        }
    ],
    "state": "Doing"
}
```

Agora, podemos remover a função `post_todo()` do módulo `create_todo` encontrado no módulo `tests/todo_api_web/controller.rs` e adicionar o `use` da função `read_json`:

```rust
mod create_todo {
    // ...
    use crate::helpers::read_json;

    #[actix_web::test]
    async fn valid_todo_post() {
        ...
        let req = test::TestRequest::post()
            .uri("/api/create")
            .insert_header((CONTENT_TYPE, ContentType::json()))
            .set_payload(read_json("post_todo.json").as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        let body = resp.into_body();
        let bytes = body::to_bytes(body).await.unwrap();
        let id = from_str::<TodoIdResponse>(&String::from_utf8(bytes.to_vec()).unwrap()).unwrap();
        assert!(uuid::Uuid::parse_str(&id.get_id()).is_ok());
    }
}
```

No próximo capítulo vamos aprender a obter todos os `TodoCard` que criamos na base de dados para depois podermos melhorar as configurações do serviço, por exemplo logs.

[Anterior](./01-ping-pong.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](./03-get.md)
