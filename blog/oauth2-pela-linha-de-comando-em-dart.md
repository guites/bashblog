OAuth2 pela linha de comando em Dart

<abbr title="muito grande; nem li">mg;nl</abbr>: vamos criar um app que autentica o usuário no GitHub via OAuth2 pela linha de comando, utilizando Dart.

Se você já utilizou aplicações como o heroku-cli, já experienciou o processo de autenticação OAuth2 pela linha de comando.

Você inicia o processo com um comando, por exemplo

        heroku login

E é direcionado para uma página da web onde você confirma as permissões necessárias para o aplicativo, digitando um login e senha.

Depois, a página mostra uma mensagem de que tudo ocorreu bem, e que você pode voltar para a linha de comando.

Neste post vamos ver como esse processo pode ser realizado utilizando Dart, criando um aplicativo que lista os repositórios da sua conta no GitHub.

<hr/>

## Overview do projeto

Você pode acessar o código completo do projeto [aqui](https://github.com/guites/dart-github-app)

O fluxo do nosso aplicativo vai ser o seguinte:

1. Usuário roda o comando necessário, `dart oauth_server`.

2. Nosso app abre uma página na web para realização de login e confirmar autorizações \(esta url vai ser gerada no momento em que criarmos o app no github\).

3. Usuário é direcionado para uma URL de callback. Para recebermos esse redirecionamento, vamos criar um servidor em dart que consegue servir uma página na URL definida.

4. Utilizamos os dados recebidos com o redirecionamento para realizar uma requisição POST na API do GitHub, recebendo em troca um _access Token_.

5. Com o _access Token_ em mãos, vamos acessar os repositórios do usuário, desenvolvendo a lógica do nosso app.

Para isso, o primeiro passo é a criação do aplicativo OAuth 2 no Github.

\( se você não conhece a linguagem dart, uma boa introdução é o [site oficial](https://dart.dev/)\)

## Criando o APP no Github

Você vai precisar de uma conta no GitHub.

Acesse [este link](https://github.com/settings/applications/new) e coloque as informações necessárias para cadastro do app.

O nome e a descrição são livres, mas preste atenção nos campos _Homepage URL_ e _Authorization callback URL_, pois eles serão utilizados no recebimento dos dados de autenticação do usuário.

![Tela de criação do aplicativo no GitHub](./imgs/oauth2-pela-linha-de-comando-em-dart/oauth_app_creation.png "Tela de criação do aplicativo no GitHub")

Optei por _localhost:4041_, mas qualquer porta em localhost pode ser utilizada, como veremos adiante.

Ao clicar em _register application_, você será direcionado para uma tela com seu _Client ID_ e uma opção para gerar seu _Client Secret_.

Gere o seu segredo e salve os dois valores.

![Tokens gerados para o app GitHub](./imgs/oauth2-pela-linha-de-comando-em-dart/oauth_tokens.jpg "Tokens gerados para o app GitHub")

**Importante**: Não compartilhe seus tokens, e cuide para não adicioná-los ao seu repositório. Com esses dados, outras aplicações podem se passar pela sua, e ter acesso aos dados dos seus usuário.

## Iniciando o projeto e abrindo o navegador para autenticação

O processo de autenticação na API do GitHub \(e em APIs utilizando OAuth2 em geral\) começa direcionando o usuário a uma URL, própria da sua aplicação.

No GitHub, a URL para início da autenticação é no formato

        https://github.com/login/oauth/authorize?scope={escopo}&client_id={seu_client_id}

O parâmetro `scope` define quais permissões de acesso nossa aplicação vai receber do usuário.

Você pode ver uma listagem completa dos escopos existentes [aqui](https://docs.github.com/en/developers/apps/building-oauth-apps/scopes-for-oauth-apps#available-scopes).

No nosso app, vamos utilizar apenas uma, `repo`. Ou seja, queremos acessar as informações de repositórios públicos e privados do nosso usuário.

O parâmetro `client_id` é o ID único da sua aplicação, gerado na etapa anterior.

Precisamos de uma forma de direcionar nosso usuário para esta URL.

Vamos começar criando um arquivo na raíz do nosso projeto, `oauth_server.dart`. Este será o ponto de partida da nossa aplicação.


        String clientId = 'seu-client-id'; // (1)
        String sentScopes = 'repo';
        String authorizationUrl =
            'https://github.com/login/oauth/authorize?scope=$sentScopes&client_id=$clientId';
        void main() {
          BrowserHandler.launchBrowser(authorizationUrl); // (2)
        }

Nossa função `main` vai definir o fluxo de eventos.

Em \(1\), vamos preencher o valor do nosso `client_id`.

A variável `sentScopes` vai guardar os escopos que estamos enviando na URL.

Em seguida, em \(2\), vamos utilizar um método `launchBrowser` para abrir o navegador no computador do usuário na URL que criamos acima.

Precisamos criar esta classe, `BrowserHandler`, e também o método.

Crie um novo diretório, `helpers`, e, dentro dele, um arquivo `browser_handler.dart`.

            // source https://stackoverflow.com/a/33033633/18081350
            import "dart:io";

            class BrowserHandler {

              static final String _userPlatform = Platform.operatingSystem; // (1)

              static void launchBrowser(String url) { // (2)
                bool fail = false;
                switch (_userPlatform) { // (3)
                  case "linux":
                    Process.run("xdg-open", [url]);
                    break;
                  case "macos":
                    Process.run("open", [url]);
                    break;
                  case "windows":
                    Process.run("explorer", [url]);
                    break;
                  default:
                    fail = true;
                    break;
                }

                if (!fail) {
                  print("Acessando $url através do navegador...");
                } else {
                  print("Por favor, acesse a URL abaixo pelo seu navegador: ");
                  print(url);
                }
              }

            }

Esta classe utiliza o pacote `dart:io` para verificar o sistema operacional do usuário, em \(1\).

Depois, em \(2\), criamos um método estático que utiliza um `switch-case` \(3\), e, dependendo do sistema, roda o processo equivalente para abertura do navegador usando `Process.run`.

Com nossa classe criada, vamos importá-la em `main.dart`.

        import 'helpers/browser_handler.dart';

Agora, podemos testar nosso app rodando pela linha de comando, na raíz do projeto:

        dart oauth_server.dart

Seu navegador deve redirecioná-lo para uma tela como esta:

![Tela de autenticação](./imgs/oauth2-pela-linha-de-comando-em-dart/authorize_screen.jpg "Tela de autorização")

Repare um output no seu terminal, na forma:

        Acessando https://github.com/login/oauth/authorize?scope=repo&client_id=47222bc3922bc180da0e através do navegador...

Se você autorizar a aplicação, clicando no botão verde, será redirecionado para uma página com erro de conexão. Isto é porque ainda não criamos a página que receberá o código para acesso.

Repare, na URL, a presença de um parâmetro `code`:

        http://localhost:4041/authorization-code?code=7e1e0c7b89ae6e2495a8

Se você **não autorizar** a aplicação, será direcionado para uma página com a seguinte URL:

        http://localhost:4041/authorization-code?error=access_denied&error_description=The+user+has+denied+your+application+access.&error_uri=https%3A%2F%2Fdocs.github.com%2Fapps%2Fmanaging-oauth-apps%2Ftroubleshooting-authorization-request-errors%2F%23access-denied

Neste caso, vamos utilizar o parâmetro `error=access_denied` para identificar que o usuário não nos concedeu a autorização.

Nosso próximo passo será um servidor para receber esse redirecionamento e utilizar o `code`.

## Criando um servidor para a URL de callback

Vamos criar em `helpers`, `server_handler.dart`, uma classe responsável por iniciar um servidor Http e gerar as respostas necessárias às requisições recebidas.

        import 'dart:io';
        import 'package:http/http.dart';

        class ServerHandler {
          late HttpServer _server;
          int _port = 4041;
          Future<HttpServer> createServer() async {
            final address = InternetAddress.loopbackIPv4;
            _server = await HttpServer.bind(address, _port);
            return _server;
          }
        }

Dessa forma, nossa classe consegue iniciar um servidor Http na porta definida por `_port`, mas precisamos definir uma resposta às requisições recebidas.

Repare também na variável `_server`. Vamos usá-la para guardar uma referência ao servidor iniciado, para que possamos finalizá-lo quando necessário.

Quando nossos usuários forem direcionados, a URL `http://localhost:4041` vai receber um `GET`. Vamos começar por ele.

Adicione os seguintes métodos na classe `ServerHandler`.

          Future<void> handleRequests(HttpServer server) async {
            await for (HttpRequest request in server) { // (1)
              switch (request.method) {
                case 'GET':
                  _handleGet(request);
                  break;
              }
            }
          }

          void _handleGet(HttpRequest request) async {
            final Uri uri = request.uri;
            final String path = uri.path; // (2)

            switch (path) {
              case '/authorization-code': // (3)
                // adicionar lógica para extrair o código da URL
                break;
            }
          }

Em \(1\), o método `handleRequests` coloca o servidor em modo de "espera": criamos uma conexão que aguarda qualquer requisição.

Com um `switch-case`, verificamos o método da requisição e, caso for um `GET`, chamamos nosso método `_handleGet` passando os dados da requisição como parâmetro.

Em `_handleGet`, pegamos a URL da requisição, \( no caso do redirecionamento após autorizar no github, seria `http://localhost:4041/authorization-code?code=7e1e0c7b89ae6e2495a8` \) e pegamos apenas o `path`, ou seja, a parte entre o fim do nome do domínio e o início dos parâmetros \( no caso da URL de redirecionamento, `authorization-code` \).

Sendo esse path o esperado, podemos colocar nossa lógica para extrair o valor do código recebido.

        void _handleGet(HttpRequest request) async {
            final Uri uri = request.uri;
            final String path = uri.path;

            switch (path) {
              case '/authorization-code':
                final Map<String, String> authParams = uri.queryParameters;
                final authCode = authParams['code'];
                break;
            }
        }

Utilizamos a propriedade `queryParameters` da nossa `uri`, que retorna um `Map`, onde a chave é o nome do parâmetro.

Vimos, porém, que o usuário pode não nos conceder a autorização. Neste caso, precisamos mostrar uma tela explicando o motivo do erro.

Vamos adicionar a seguinte verificação no nosso método:

            switch (path) {
              case '/authorization-code':
                final Map<String, String> authParams = uri.queryParameters;

                // independente de erro ou sucesso,
                // vamos renderizar uma página html para o usuário
                request.response.headers.contentType = ContentType.html;

                // caso não tenhamos recebido o parâmetro [code]
                if (!authParams.containsKey('code')) {
                  if (authParams.containsKey('error')) {
                    if (authParams['error'] == 'access_denied') {
                      // usuário não concedeu permissões
                      request.response
                        ..write('''
                          <h1>Permissão não concedida!</h1>
                          <p>Você precisa autorizar nossa aplicação junto ao GitHub para utilizar nossas funcionalidades.</p>
                        ''')
                        ..close();
                      return _server.close();
                    }
                    // erro desconhecido
                    request.response
                      ..write('''
                          <h1>Ocorreu um erro!</h1>
                          <p>Ocorreu um erro ao autorizar o uso da aplicação. Por favor, inicie o processo novamente.</p>
                        ''')
                      ..close();
                    return _server.close();
                  }
                }

                final authCode = authParams['code'];
                break;
            }

Primeiro definimos que o tipo da resposta enviada pelo nosso servidor local vai ser html.

Depois, verificamos se o usuário recebeu na URL o parâmetro `code`.

Caso não tenha recebido, podem ser dois casos: o usuário negou as permissões, o que resulta no envio do parâmetro `error=access_denied`, ou algum outro erro desconhecido.

No segundo caso, vamos tratar todos com uma mesma mensagem pedindo para tentar novamente.

Independente do ocorrido, vamos finalizar a execução do servidor com o método `_server.close()`.

Agora, caso tenhamos recebido nosso `code`, de acordo com a [documentação da API do GitHub](https://docs.github.com/en/developers/apps/building-oauth-apps/authorizing-oauth-apps#2-users-are-redirected-back-to-your-site-by-github) precisamos realizar uma requisição POST

        POST https://github.com/login/oauth/access_token

Na qual enviamos o `code` junto com nossos `client_id` e `client_secret`, recebendo em troca um `access_token`, utilizado para autenticar as requisições em nome do nosso usuário.

## Conseguindo o `access_token` do usuário

Com o Dart, podemos criar uma requisição HTTP facilmente com o pacote `http.dart`.

Para adicionar este pacote, crie, na raíz do seu projeto, um arquivo `pubspec.yaml`.

        name: oauth_server
        environment:
            sdk: '>=2.12.0 <3.0.0' # versão 2.12 habilita null safety

Com o arquivo criado, podemos adicionar nossa dependência, usando

        dart pub add http

Repare que seu `pubspec.yaml` foi atualizado com a linha

        dependencies: 
          http: ^0.13.4

Agora, vamos importar o pacote em `server_handler.dart`, e adicionar a lógica da requisição no nosso método `_handleGet`.

Esta requisição vai precisar dos dados de `client_id` e `client_secret`. Como eles foram definidos no nosso `main.dart`, vamos
passá-los como argumentos no construtor da nossa classe `ServerHandler`.

        import 'dart:io';
        import 'package:http/http.dart'; // ADICIONAR
        import 'dart:convert'; // ADICIONAR

        class ServerHandler {
            final String _clientId;                             // ADICIONAR
            final String _clientSecret;                         // ADICIONAR
            ServerHandler(this._clientId, this._clientSecret);  // ADICIONAR
            late HttpServer _server;
            final int _port = 4041;
            String _accessTokenUrl = 'https://github.com/login/oauth/access_token'; // ADICIONAR
            ...
            ...
            ...

Com acesso aos nossos _tokens_, podemos criar nossa requisição HTTP.

Vamos utilizar o método `post`, que aceita como parâmetros a URL da requisição, os cabeçalhos e o corpo da requisição.

        final authCode = authParams['code'];

        final tokenRequest = await post(
          Uri.parse(_accessTokenUrl),
          headers: {
            "Accept": "application/json",
          },
          body: {
            "client_id": _clientId,
            "client_secret": _clientSecret,
            "code": authCode,
          },
        );

        break;

Agora que realizamos nossa requisição, enviando os valores para o servidor do github, vamos receber nosso `access_token`.

O ideal nesta etapa seria verificar o código de resposta da requisição, onde 200 indica requisição bem sucedida, 400 indica erro
nos dados enviados, etc.

Acontece que o GitHub retorna status 200 mesmo que os dados enviados não estejam corretos. Para nos certificarmos de que tudo ocorreu bem, podemos verificar se a requisição retornou o campo 'error', e, neste caso, avisar o usuário.

Caso o campo 'error' venha vazio, podemos checar se o valor de 'access\_token' foi recebido, e, neste caso, finalizar o processo de autenticação.

            final tokenObject = json.decode(tokenRequest.body);

            // github retorna http 200 quando ocorre erro!

            if (tokenObject.containsKey('error')) {
              request.response
                ..write('''
                <h1> Ocorreu um erro!</h1>
                <p>Por favor, verifique se as permissões necessárias foram concedidas.</p>
                ''')
                ..close();
              return _server.close();
            }

            if (tokenObject.containsKey('access_token')) {
              request.response
                ..write('''
                <h1> Autenticação realizada com sucesso! </h1>
                <p>Você já pode voltar para a linha de comando.</p>
                ''')
                ..close();
              return _server.close();
            }

            break;

### Implementando a criação de servidor na função `main`

Agora que desenvolvemos nossa classe `ServerHandler`, vamos chamá-la na função `main`, e testar seu funcionamento.

Em `oauth_server.dart`, adicione:

        import 'helpers/browser_handler.dart';
        import 'helpers/server_handler.dart'; // ADICIONAR
        import 'dart:io'; // ADICIONAR

        String clientId = '47222bc3922bc180da0e';
        String clientSecret = 'cb32e0058343ee20c8e4d16cb1dc740ecec5c812'; // ADICIONAR
        String sentScopes = 'repo';
        String authorizationUrl =
            'https://github.com/login/oauth/authorize?scope=$sentScopes&client_id=$clientId';
        void main() async {
          final ServerHandler serverHandler = ServerHandler(clientId, clientSecret);
          final HttpServer server = await serverHandler.createServer();
          print('Servidor iniciado: ${server.address} port ${server.port}');
          BrowserHandler.launchBrowser(authorizationUrl);
          await serverHandler.handleRequests(server);
        }

Rodando o programa com `dart oauth_server.dart`, teremos dois outputs:

Ao permitir acesso à aplicação:

![Tela de permissão concedida](./imgs/oauth2-pela-linha-de-comando-em-dart/permission_granted.png "Tela de permissão concedida")

Ao recusar permissão:

![Tela de permissão recusada](./imgs/oauth2-pela-linha-de-comando-em-dart/permission_denied.png "Tela de permissão recusada")

Perfeito! Agora podemos utilizar nosso token nas requisições subsequentes.

## Guardando o access token e utilizando um singleton

Quando o usuário nos permite acesso, recebemos o seguinte _payload_ na nossa requisição POST:

        {"access_token":"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","token_type":"bearer","scope":"repo"}

Para termos acesso ao token fora da classe `ServerHandler`, vamos criar um _model_ `TokenModel`, que vai receber as informações do token.

Ele vai ter uma chave para cada campo passível de recebimento da API.

Crie um diretório `models` e, dentro dele, `token_model.dart`:

Este arquivo vai possuir duas classes, `UniqueToken` e  `TokenModel`.

`TokenModel`, como dito acima, vai guardar os valores \(ou o erro\) recebido da API.

A classe `UniqueToken` vai ser utilizada como um _singleton_, ou seja, vai ser utilizada para que possamos acessar, de qualquer lugar do projeto, a mesma instância de `TokenModel`.

Utilizamos uma lógica parecida no post [Criando um app com Flutter e sqlite](https://guilhermegarcia.dev/blog/criando-um-app-com-flutter-e-sqlite.html), na seção _Configurando o acesso ao banco de dados_.

        class TokenModel {
          String? accessToken;
          String? tokenType;
          String? scope;
          String? error;

          TokenModel({this.accessToken, this.tokenType, this.scope, this.error});

          factory TokenModel.fromJson(parsedJson) {
            return TokenModel(
                accessToken: parsedJson['access_token'],
                tokenType: parsedJson['token_type'],
                scope: parsedJson['scope'],
                error: parsedJson['error']);
          }
        }

O método `factory TokenModel.fromJson`, recebe um valor json decodificado \(parsedJson\), e retorna uma instância `TokenModel` a partir dele.

Já em nosso singleton, vamos verificar se ele já foi inicializado e, em caso negativo, utilizar o método `fromJson`, passando o retorno da API como argumento.

        class UniqueToken {
          static final UniqueToken _token = UniqueToken._internal();
          UniqueToken._internal();

          bool initialized = false;
          static UniqueToken get instance => _token;
          static late TokenModel _tokenModel;

          TokenModel tokenModel([parsedJson = null]) {
            if (initialized) {
              return _tokenModel;
            }
            _tokenModel = TokenModel.fromJson(parsedJson);
            initialized = true;
            return _tokenModel;
          }
        }

Caso o singleton já tenha sido inicializado, retornamos a instância existente, ao invés de criar uma nova.

Com os nossos models criados, vamos importá-los, primeiramente, em `helpers/server_handler.dart`.

        import 'dart:convert';
        import 'dart:io';
        import 'package:http/http.dart';
        import '../models/token_model.dart'; // ADICIONAR

A ideia é inicializar nosso `UniqueToken`, passando o json que recebemos da API, através do método `tokenModel`.

        if (tokenObject.containsKey('access_token')) {
          final token = UniqueToken.instance.tokenModel(tokenObject); // ADICIONAR
          request.response
            ..write('''
            <h1> Autenticação realizada com sucesso! </h1>
            <p>Você já pode voltar para a linha de comando.</p>
            ''')
            ..close();
          return _server.close();
        }

## Realizando uma requisição autenticada

Agora que podemos acessar nosso token, vamos criar um método para listar os repositórios do nosso usuário autenticado.

Crie um diretório `requests` e, dentro dele, `repositories.dart`:

        import 'dart:convert';
        import '../models/token_model.dart';

        import 'package:http/http.dart' as http;

        getUserRepositories(TokenModel tokenModel) async {
          final http.Client client = http.Client();
          final String requestUrl = 'https://api.github.com/user/repos';
          final http.Response response = await client.get(
            Uri.parse(requestUrl),
            headers: {
              "Accept": "application/vnd.github.v3+json",
              "Authorization": "token ${tokenModel.accessToken}",
            },
          );

          if (response.statusCode == 200) {
            final List<dynamic> responseObject = jsonDecode(response.body);
            responseObject.forEach((repo) => print(repo['html_url']));
          }
        }

Vamos definir apenas uma função, que recupera os repositórios, públicos e privados, do usuário autenticado.

Caso a requisição seja realizada com sucesso, vamos printar a URL destes repositórios no terminal.

Para utilizarmos a função, vamos adicionar em `oauth_server.dart`:

        import 'helpers/browser_handler.dart';
        import 'helpers/server_handler.dart';
        import 'models/token_model.dart';
        import 'requests/repositories.dart'; // ADICIONAR
        import 'dart:io';

E, no método `main`:

        void main() async {
          final ServerHandler serverHandler = ServerHandler(clientId, clientSecret);
          final HttpServer server = await serverHandler.createServer();
          print('Servidor iniciado: ${server.address} port ${server.port}');
          BrowserHandler.launchBrowser(authorizationUrl);
          await serverHandler.handleRequests(server);
          final TokenModel token = UniqueToken.instance.tokenModel();
          getUserRepositories(token); // ADICIONAR
        }

Pronto! Agora temos acesso aos dados do usuário autenticado.

## Considerações finais

Neste post vimos como autenticar um usuário através do processo OAuth2 pela linha de comandos, utilizando a linguagem Dart.

Utilizamos como exemplo a API do GitHub, mas esse processo pode ser generalizado para qualquer API que utilize OAuth2.

Alguns pontos que não foram explorados são: 

- persistir os dados do usuário utilizando um banco de dados: isso evita que o usuário precise autenticar no github toda vez que utilizar o aplicativo.
- gerar um binário executável: permite uso do app por usuários que não possuem a VM do Dart instalada na máquina.

Abraço!

Fontes:

- <https://devcenter.heroku.com/articles/heroku-cli#get-started-with-the-heroku-cli>
- <https://dart.dev/null-safety/understanding-null-safety>
- <https://api.dart.dev/stable/2.16.1/dart-io/HttpServer-class.html> 
- <https://docs.github.com/en/rest>
- <https://docs.github.com/en/rest/reference/repos#list-repositories-for-the-authenticated-user>

Tags: dart, github, oauth2, api
