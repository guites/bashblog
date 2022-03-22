Apps de linha de comando em dart

<abbr title="muito grande;nem li">mg;nl</abbr>: Vamos dar sequência ao app iniciado [neste post](https://guilhermegarcia.dev/blog/oauth2-pela-linha-de-comando-em-dart.html), explorando ferramentas da linguagem dart para tornar nosso aplicativo mais interativo, lendo e escrevendo input do usuário e arquivos, assim como permitindo ao usuário escolher entre possibilidades que alteram o fluxo do app.

No [post anterior](https://guilhermegarcia.dev/blog/oauth2-pela-linha-de-comando-em-dart.html) desenvolvemos um aplicativo que autentica o usuário através do protocolo OAuth2, e, com as permissões concedidas, gera uma listagem dos seus repositórios, públicos e privados, com o link para acessá-los no GitHub.

Neste post, vamos extender algumas dessas funcionalidades, focando em tornar sua utilização mais fácil e intuitíva, facilitando o processo de configuração e partindo do princípio que o usuário final não possui conhecimentos de programação.

Se você quiser ir programando junto, nosso ponto de partida é o [branch oauth2\_flux deste repositório](https://github.com/guites/dart-github-app/tree/oauth2_flux).

O código finalizado pode ser encontrado no branch [add\_models](https://github.com/guites/dart-github-app/tree/add_models)

<hr/>

## Overview do projeto

Vamos seguir os seguintes passos ao longo deste post:

1. Salvar informações sensíveis em um arquivo _.env_.
2. Verificar se as configurações necessárias foram definidas e, em caso negativo, pedir seu preenchimento através de input do usuário.
3. Na listagem dos repositórios do usuário, permitir que ele possa selecioná-los, mostrando seus _issues_ em aberto.

## Salvando informações no .env

O uso do arquivo _.env_ é uma prática popular na programação, pois permite que que informações sensíveis, que não devem ser compartilhadas, não sejam compartilhadas através de softwares de versionamento de código, como o _git_ ou _svn_.

A ideia consiste em guardar dados como o _client\_secret_ e _client\_id_ de uma aplicação OAuth em um arquivo de texto, ao invés de colocá-lo diretamente no código.

Assim, você pode versionar seus arquivos de código e adicionar o arquivo com as informações \(normalmente chamado _.env_\) ao _.gitignore_.

Vamos criar uma classe responsável por cuidar da lógica de leitura e escrita em disco, e também por segurar na memória os valores lidos do arquivo _.env_.

Crie, dentro de `helpers`, o arquivo `environment_handler.dart`.

Nele, vamos utilizar o mesmo formato _Singleton_ utilizado no nosso `TokenModel`, mas com uma diferença: ele vai ser inicializado com um método estático `loadEnv`, que puxa os valores do disco, ao invés de um `factory`.

        import 'dart:convert';
        import 'dart:io';

        class EnvironmentHandler {
        Map _vars; // propriedade privada que vai guardar um Map com o nome e valor das variáveis presentes no .env
        EnvironmentHandler._(this._vars);

        // método estático utilizado para inicializar a classe
        static Future<EnvironmentHandler> loadEnv(String filePath, List requiredKeys) async {

        }

Nosso método recebe dois parâmetros, `filePath`, que nos permite manter mais de um `.env` no projeto \(`.env.test`, `.env.dev`, etc)  e `requiredKeys`, que define quais chaves, caso não estiverem preenchidas no `.env`, nós vamos pedir ao usuário para fornecer.

        static Future<EnvironmentHandler> loadEnv(String filePath, List requiredKeys) async {
            // verifica se o arquivo .env existe
            File envFile = File(filePath);
            if (!await envFile.exists()) {
                // não existe, deve ser criado
                await envFile.create();
            }

            // variável para salvar os valores do .env baseado nas requiredKeys recebidas
            final Map varsFromFile = {for (var v in requiredKeys) v: false};

            // inicia stream de leitura no .env
            Stream<String> lines = envFile
                .openRead()
                .transform(utf8.decoder)
                .`transform(const LineSplitter());

Nós iniciamos uma variável `varsFromFile` para usar de referência, mais abaixo, e testar se os valores foram corretamente fornecidos.

Em dart, a leitura de um arquivo nos retorna um objeto `Stream`, que pode receber diversas transformações \(`transform`s). Neste caso, vamos passar a codificação desejada \(UTF-8\) e utilizar o `LineSplitter`. Isso nos permite iterar sobre as linhas.

        await for (var line in lines) {
            var keyVal = line.split('=');
            if (keyVal.length == 1) { // (1)
                throw Exception('O arquivo .env deve ser no formato CHAVE=VALOR !!');
            }

            if (keyVal[1].trim().isEmpty) {
                // valor não foi preenchido, alertar usuário para preenchimento
                stdout.writeln('Digite seu ${keyVal[0]}: ');
                final input = stdin.readLineSync();
                if (input == null) {
                    throw Exception(
                        'Você precisa preencher o valor de ${keyVal[0]} no seu arquivo .env !');
                }
                varsFromFile[keyVal[0]] = input.trim(); // (2)
            } else {
                // valor já foi preenchido
                varsFromFile[keyVal[0]] = keyVal[1].trim(); // (2)
            }
        }

Iterando sobre as linhas, em \(1\), verificamos se elas estão no formato `CHAVE=VALOR`.

Existindo a chave sem o valor preenchido, vamos utilizar a função `input` para aguardar um preenchimento por parte do usuário.

Por exemplo, um arquivo `.env` preenchido com:

        SEGREDO=

Vai resultar em:

        > dart run
        Building package executable...
        Built oauth_server:oauth_server.
        Digite seu SEGREDO:

Em \(2\), passamos o valor da chave para nossa variável `varsFromFile`. No próximo passo, vamos verificar se os valores obrigatórios foram preenchidos.

        // verifica se as chaves obrigatórias estão presentes
        if (varsFromFile.containsValue(false)) {
            // gera uma lista com o nome das variáveis faltantes
            var emptyVar =
                varsFromFile.keys.where((k) => varsFromFile[k] == false).toList();
            for (var i = 0; i < emptyVar.length; i++) {
                // alerta usuário para preencher as variáveis faltantes
                stdout.writeln('Digite seu ${emptyVar[i]}: ');
                final input = stdin.readLineSync();
                if (input == null || input.trim().isEmpty) {
                throw Exception(
                    'Você precisa preencher a variável ${emptyVar[i]} no seu arquivo $filePath !!');
                }
                varsFromFile[emptyVar[i]] = input.trim();
            }
        }

Caso nenhuma exceção seja jogada até aqui, o preenchimento ocorreu de forma correta.

Podemos agora sobrescrever o arquivo `.env` com os novos valores.

            // sobrescreve valores no arquivo .env
            IOSink sink = envFile.openWrite();
            varsFromFile.forEach((k, v) => {sink.write('$k=$v\n')});
            sink.close();

            return EnvironmentHandler._(varsFromFile);
        } // fim do método loadEnv

Repare que nós retornamos uma instância da nossa classe `EnvironmentHandler`, passando como argumento os valores lidos do `.env` e preenchidos pelo usuário.

Precisamos de mais um método na nossa classe, que nos permita retornar o valor de cada chave do `.env`.

    String getVar(String key) {
        if (!_vars.containsKey(key)) {
        throw Exception(
            '$key não foi definida! Verifique se ela consta no seu arquivo .env .');
        }
        return _vars[key];
    }

Agora, podemos iniciá-lo no nosso `oauth_server.dart`.

        import 'helpers/browser_handler.dart';
        import 'helpers/server_handler.dart';
        import 'models/token_model.dart';
        import 'requests/repositories.dart';
        import 'dart:io';

        List requiredKeys = ['GITHUB_CLIENT_ID', 'GITHUB_CLIENT_SECRET'];
        late String clientId; // ALTERAR
        late String clientSecret; // ALTERAR
        String sentScopes = 'repo';
        String authorizationUrl =
            'https://github.com/login/oauth/authorize?scope=$sentScopes&client_id='; // ALTERAR

        void main() async {
            final EnvironmentHandler env = await EnvironmentHandler.loadEnv('.env', requiredKeys); // ADICIONAR
            clientId = env.getVar('GITHUB_CLIENT_ID'); // ADICIONAR
            clientSecret = env.getVar('GITHUB_CLIENT_SECRET'); // ADICIONAR
            final ServerHandler serverHandler = ServerHandler(clientId, clientSecret);
            final HttpServer server = await serverHandler.createServer();
            print('Servidor iniciado: ${server.address} port ${server.port}');
            BrowserHandler.launchBrowser('$authorizationUrl$clientId'); // ALTERAR
            await serverHandler.handleRequests(server);
            final TokenModel token = UniqueToken.instance.tokenModel();
            getUserRepositories(token);
        }

Experimente rodar o programa sem um arquivo `.env` existente. O terminal irá guiá-lo no preenchimento dos valores necessários!

## Melhorando a listagem de repositórios

Para conseguirmos expandir nossa funcionalidades, esse é um bom momento para modelar os valores que recebemos da API.

São três objetos principais com os quais vamos lidar:

1. [Users](https://docs.github.com/en/rest/reference/users#get-the-authenticated-user)
2. [Repositories](https://docs.github.com/en/rest/reference/repos#list-repositories-for-the-authenticated-user)
3. [Issues](https://docs.github.com/en/rest/reference/issues#list-repository-issues)

Vamos utilizar o `built_value` para transformá-los em classes dart rapidamente. Você pode conseguir mais informações sobre esse plugin em <https://pub.dev/packages/built_value>.

Você pode consultar os campos de cada objeto nos seus endpoints equivalente na API do GitHub (clique nos nomes na listagem acima).

Não vamos criar o model para `Issues` agora. Vamos primeiro implementar `Users` e `Repositories`, e verificar seu funcionamento.

### Instalando o `built_value`.

Você pode instalar o plugin adicionando as seguintes linhas no seu `pubspec.yaml`:

    name: oauth_server
    environment:
      sdk: '>=2.12.0 <3.0.0'
    dependencies:
      built_value: ^8.1.4 # ADICIONAR
      http: ^0.13.4
      build_runner: ^2.1.7 # ADICIONAR
      built_value_generator: ^8.1.4 # ADICIONAR

Depois, na raíz do projeto, rode um `dart pub get`.

### Criando o Model User

Vamos adicionar em `models/user_modal.dart`:

        import 'dart:convert';

        import 'package:built_value/built_value.dart';
        import 'package:built_value/serializer.dart';
        import 'serializers.dart';

        part 'user_model.g.dart';

        abstract class User implements Built<User, UserBuilder> {
            String get login;
            int get id;
            String get node_id;
            String get avatar_url;
            String get gravatar_id;
            String get url;
            String get html_url;
            String get followers_url;
            String get following_url;
            String get gists_url;
            String get starred_url;
            String get subscriptions_url;
            String get organizations_url;
            String get repos_url;
            String get events_url;
            String get received_events_url;
            String get type;
            bool get site_admin;

            User._();
            factory User([void Function(UserBuilder)? updates]) = _$User;

            static User? fromJson(String jsonString) {
                return serializers.deserializeWith(
                    User.serializer, json.decode(jsonString));
            }

            static Serializer<User> get serializer => _$userSerializer;
        }

### Criando o model Repository

Vamos adicionar em `models/repository_model.dart`:

        import 'package:built_value/built_value.dart';
        import 'package:built_collection/built_collection.dart';
        import 'package:built_value/serializer.dart';
        import 'serializers.dart';

        import 'user_model.dart';

        part 'repository_model.g.dart';

        abstract class Repository implements Built<Repository, RepositoryBuilder> {
            int get id;
            String get node_id;
            String get name;
            String get full_name;
            User get owner; // instace of User model (1)
            bool get private;
            String get html_url;
            String? get description;
            bool get fork;
            String get url;
            String get archive_url;
            String get assignees_url;
            String get blobs_url;
            String get branches_url;
            String get collaborators_url;
            String get comments_url;
            String get commits_url;
            String get compare_url;
            String get contents_url;
            String get contributors_url;
            String get deployments_url;
            String get downloads_url;
            String get events_url;
            String get forks_url;
            String get git_commits_url;
            String get git_refs_url;
            String get git_tags_url;
            String get git_url;
            String get issue_comment_url;
            String get issue_events_url;
            String get issues_url;
            String get keys_url;
            String get labels_url;
            String get languages_url;
            String get merges_url;
            String get milestones_url;
            String get notifications_url;
            String get pulls_url;
            String get releases_url;
            String get ssh_url;
            String get stargazers_url;
            String get statuses_url;
            String get subscribers_url;
            String get subscription_url;
            String get tags_url;
            String get teams_url;
            String get trees_url;
            String get clone_url;
            String? get mirror_url;
            String get hooks_url;
            String get svn_url;
            String? get homepage;
            String? get language;
            int get forks_count;
            int get stargazers_count;
            int get watchers_count;
            int get size;
            String get default_branch;
            int get open_issues_count;
            bool get is_template;
            BuiltList<String> get topics;
            bool get has_issues;
            bool get has_projects;
            bool get has_wiki;
            bool get has_pages;
            bool get has_downloads;
            bool get archived;
            bool get disabled;
            String get visibility;
            String get pushed_at;
            String get created_at;
            String get updated_at;
            BuiltMap<String, bool> get permissions;
            bool? get allow_rebase_merge;
            String? get template_repository;
            String? get temp_clone_token;
            bool? get allow_squash_merge;
            bool? get allow_auto_merge;
            bool? get delete_branch_on_merge;
            bool? get allow_merge_commit;
            int? get subscribers_count;
            int? get network_count;
            BuiltMap<String, String> get license;
            int get forks;
            int get open_issues;
            int get watchers;

            Repository._();
            factory Repository([void Function(RepositoryBuilder)? updates]) =
                _$Repository;

            static Repository fromJson(Map<String, dynamic> jsonObj) {
                return serializers.deserializeWith(Repository.serializer, jsonObj)!;
            }

            static Serializer<Repository> get serializer => _$repositorySerializer;
        }

Optei por carregar todas as propriedades listadas no endpoint, mas você pode separa apenas as que deseja utilizar.

Repare que em \(1\), o tipo da propriedade é a classe `User` que criamos anteriormente.

### Criando o Serializer

Repare que todas as classes criadas possuem um método estático, `fromJson`. Esse método nos permite instanciar a classe passando como valor de entrada uma string, que é deserializada usando `serializers.deserializeWith`. Para que possamos utilizar este método, precisamos primeiro criar uma classe, `Serializer`.

Adicione, em `models/serializers.dart`:

    import 'package:built_collection/built_collection.dart';
    import 'package:built_value/standard_json_plugin.dart';
    import 'package:built_value/serializer.dart';

    import 'repository_model.dart';
    import 'user_model.dart';

    part 'serializers.g.dart';

    @SerializersFor([Repository, User])
    final Serializers serializers =
        (_$serializers.toBuilder()..addPlugin(StandardJsonPlugin())).build();

Repare que em `SerializersFor`, passamos, no array, `Repository` e `User`.

### Criando os arquivos gerados automaticamente

Antes de rodarmos o `build` para a geração dos arquivos automáticos, precisamos ajustar a organização dos nossos diretórios e arquivos.

Isso é necessário pois o `built_values` espera uma certa configuração específica para conseguir encontrar os arquivos necessários e salvar os dados gerados.

Vamos criar, na raíz do projeto, um diretório `bin`.

Depois, vamos passar todos nossos arquivos `.dart` pra dentro dele.

Nossa estrutura final vai ficar da seguinte forma:

      - pubspec.yaml
      - pubspec.lock
      - README.md
      - .env
      - bin/
        - oauth_server.dart
        - helpers/
          - browser_handler.dart
          - environment_handler.dart
          - server_handler.dart
        - models
          - repository_model.dart
          - serializers.dart
          - token_model.dart
          - user_model.dart
        - requests/
          - repositories.dart

Com os models criados, podemos agora rodar

    dart run build_runner build

Agora, podemos instaciar nossas classes.

Para isso, vamos alterar o método utilizado para realizar a requisição na API do GitHub.

## Alterando a requisição na API

Vamos alterar nosso arquivo `requests/repositories.dart`.

A primeira alteração é no retorno. Nossa função agora vai retornar uma listagem de repositórios.

Cada item da lista vai ser uma instância da classe `Repository`. Isso, junto ao fato de ser uma requisição assíncrona, vai nos dar a seguinte tipagem:

Precisamos também importar a classe `Repository`.

    import 'dart:convert';

    import '../models/repository_model.dart'; // ADICIONAR
    import '../models/token_model.dart';

    import 'package:http/http.dart' as http;

    Future<List<Repository>> getUserRepositories(TokenModel tokenModel) async { // ALTERAR
      final http.Client client = http.Client();
      final String requestUrl = 'https://api.github.com/user/repos';
      final http.Response response = await client.get(
        Uri.parse(requestUrl),
        headers: {
          "Accept": "application/vnd.github.v3+json",
          "Authorization": "token ${tokenModel.accessToken}",
        },
      );

      if (response.statusCode != 200) {
        throw Exception('Ocorreu um erro na conexão com a API do GitHub!');
      }
      final List<dynamic> payload = json.decode(response.body);
      List<Repository> repositories =                                    // ALTERAR
          payload.map((data) => Repository.fromJson(data)).toList();

      return repositories;
    }

Agora temos acesso à essa listagem no nosso `oauth_server.dart`. Com isso, podemos focar em deixar nosso app um pouco mais interativo.

## Permitindo que o usuário escolha um repositório

Com os novos dados que estamos salvando em memória, podemos adicionar três novas funcionalidades:

1. Mostrar dados do usuário autenticado
2. Ao mostrar a listagem dos repositórios, permitir que o usuário escolha um deles.
3. Ao selecionar um repositório, mostrar os detalhes recebidos da API de forma ordenada.

### Mostrando dados do usuário autenticado

Vamos criar uma nova requisição, criando o arquivo `requests/authenticated_user.dart`.

Vamos utilizar essa função para, usando o `access_token` recebido, puxar as informações do usuário autenticado.

    import '../models/user_model.dart';
    import '../models/token_model.dart';

    import 'package:http/http.dart' as http;

    Future<User> getAuthenticatedUser(TokenModel tokenModel) async {
      final http.Client client = http.Client();
      final String requestUrl = 'https://api.github.com/user';
      final http.Response response = await client.get(
        Uri.parse(requestUrl),
        headers: {
          "Accept": "application/vnd.github.v3+json",
          "Authorization": "token ${tokenModel.accessToken}",
        },
      );

      if (response.statusCode != 200) {
        throw Exception('Ocorreu um erro na conexão com a API do GitHub!');
      }

      User authenticatedUser = User.fromJson(response.body);

      return authenticatedUser;
    }

Para mostrarmos os dados no terminal, vamos criar um arquivo que vai ter todas as funções responsáveis por mostrar dados da classe `User`.

Leve em consideração que os dados que recebemos do usuário autenticado, após instanciar a classe `User`, ficam no seguinte formato:

    User {
      login=usuario,
      id=124971234,
      node_id=MASJF898ASF8,
      avatar_url=https://avatars.githubusercontent.com/u/124971234?v=4,
      gravatar_id=,
      url=https://api.github.com/users/usuario,
      html_url=https://github.com/usuario,
      followers_url=https://api.github.com/users/usuario/followers,
      following_url=https://api.github.com/users/usuario/following{/other_user},
      gists_url=https://api.github.com/users/usuario/gists{/gist_id},
      starred_url=https://api.github.com/users/usuario/starred{/owner}{/repo},
      subscriptions_url=https://api.github.com/users/usuario/subscriptions,
      organizations_url=https://api.github.com/users/usuario/orgs,
      repos_url=https://api.github.com/users/usuario/repos,
      events_url=https://api.github.com/users/usuario/events{/privacy},
      received_events_url=https://api.github.com/users/usuario/received_events,
      type=User,
      site_admin=false,
    }

Crie um diretório `bin/output`, e, dentro dele, `user.dart`.

Vamos criar um método que printa um cabeçalho simples no terminal.

    import 'dart:io';
    import '../models/user_model.dart';

    void showAuthUser(User user) {
      stdout.writeln('------------${ '-' * user.login.length }---${'-' * user.html_url.length}');
      stdout.writeln('Logado como ${user.login} - ${user.html_url}');
      stdout.writeln('------------${ '-' * user.login.length }---${'-' * user.html_url.length}');
    }

Apenas um cabeçalho simples, calculando o tamanho da string `login` para criar uma faixa com traços.

### Mostrando a listagem de repositórios

Vamos reformular nosso loop para listagem dos repositórios disponíveis.

Em `output`, crie o arquivo `repository.dart`.

    import 'dart:io';
    import '../models/repository_model.dart';

    void showUserRepositories(List<Repository> repos) {
      stdout.writeln('Os seguintes repositórios foram encontrados:\n');
      for (var i = 0; i < repos.length; i++) {
        stdout.writeln('[${i + 1}]: ${repos[i].full_name}');
      }
    }

A mesma listagem, mas agora recebemos uma `List` de `Repository`s como entrada.

Outra alteração é a adição do índice entre colchetes `[1]`. Vamos usar essa numeração para buscar mais informações de cada repositório.

Nosso método `main()` no arquivo `oauth_server.dart` vai ficar no seguinte formato:


    void main() async {
      final EnvironmentHandler env =
          await EnvironmentHandler.loadEnv('.env', requiredKeys);
      clientId = env.getVar('GITHUB_CLIENT_ID');
      clientSecret = env.getVar('GITHUB_CLIENT_SECRET');
      final ServerHandler serverHandler = ServerHandler(clientId, clientSecret);
      final HttpServer server = await serverHandler.createServer();
      BrowserHandler.launchBrowser('$authorizationUrl$clientId');
      await serverHandler.handleRequests(server);
      final TokenModel token = UniqueToken.instance.tokenModel();
      final User user = await getAuthenticatedUser(token);
      showAuthUser(user);
      List<Repository> repos = await getUserRepositories(token);
      showUserRepositories(repos);
    }

Porém, ainda não temos nenhuma interatividade. Vamos agora implementar a lógica para a seleção dos repositórios.

### Mostrando detalhes de um repositório selecionado

Uma forma fácil de implementar, que impede o programa de terminar a execução, é uma combinação de um `while(true)` com `stdin.readLineSync()`.

O `while` mantém o programa rodando, enquanto o `readLineSync` trava a execução e aguarde indefinidamente por uma ação do usuário.

Como temos um índice associado a cada repositório da listagem, podemos aceitar como input o índice, e então listar os dados do repositório.

No nosso arquivo `output/repository.dart`, vamos adicionar o seguinte método:

      void showRepositoryInfo(List<Repository> repos) {
        stdout.write('Digite o número do repositório para receber mais informações:');
        final input = stdin.readLineSync();
      }

Antes de acessarmos a lista `repos` passando o índice recebido, precisamos fazer uma série de verificações:

1. O usuário digitou algum valor, ou apenas apertou <kbd>ENTER</kbd> ?
2. O valor digitado é um número?
3. O número recebido é um índice válido na nossa listagem?

Se a resposta a todas essas questões for _sim_, podemos ter uma boa confiança de que o valor recebido est
á num formato aceitável.

Para isso, vamos utilizar um estratégia de `try catch`, jogando exceções diferentes para cada provavel erro.

      void showRepositoryInfo(List<Repository> repos) {
        stdout.write('Digite o número do repositório para receber mais informações:');
        final input = stdin.readLineSync();
        try {
          if (input == null || input.trim().isEmpty) { // (1)
            throw Exception('Digite o número à esquerda do repositório que você deseja analisar!');
          }
          String filteredInput = input.trim().replaceAll(RegExp(r'\[|\]'), ''); // (2)
          int index = int.parse(filteredInput); // (3)
          final Repository repo = repos[index - 1]; // (4)
          // caso chegarmos até aqui, podemos disponibilizar a informação para o usuário!
        } catch (e) {
          // precisamos lidar aqui com cada caso de erro possível!
        }
      }

Vamos analisar uma por uma as verificações realizadas.
Em \(1\) caso o usuário não passar nenhum valor, ou apenas espeços, jogamos uma Exceção manualmente.

Em \(2\) removemos todos os espaços em branco, e também os colchetes, caso a pessoa tente digitar `[1]` ao invés de `1`.

Em \(3\) utilizamos o método `int.parse` para verificarmos se o valor recebido se trata de um número inteiro. Caso não seja, por ex. `teste`, este método joga uma `FormatException`, com a qual lidaremos depois.

Em \(4\) utilizamos o índice recebido, e diminuimos 1, para acessar a nossa lista. Caso o valor do índice seja maior do que a quantiade de itens, o dart nos fornece um `RangeError` automaticamente. Caso o usuário digite 0, recebemos o mesmo `RangeError`. Com isso, poupamos uma verificação do tipo

    if (repos.length >= index)

Agora, vamos preencher o bloco `catch` e mostrar mensagens amigáveis para o usuário


      void showRepositoryInfo(List<Repository> repos) {
        stdout.write('Digite o número do repositório para receber mais informações:');
        final input = stdin.readLineSync();
        try {
          if (input == null || input.trim().isEmpty) { // (1)
            throw Exception('Digite o número à esquerda do repositório que você deseja analisar!');
          }
          String filteredInput = input.trim().replaceAll(RegExp(r'\[|\]'), ''); // (2)
          int index = int.parse(filteredInput); // (3)
          final Repository repo = repos[index]; // (4)
          // caso chegarmos até aqui, podemos disponibilizar a informação para o usuário!
        }  on FormatException {
          stderr.writeln('Por favor, digite um número de 1 a ${repos.length}!');
        } on RangeError {
          stderr.writeln(
              'Repositório não encontrado! Por favor, digite um número de 1 a ${repos.length}!');
        } catch (e) {
          stderr.writeln(e);
        }
      }

Este formato deixa claro o poder do `try catch` para controle de fluxo.

Com nossas verificações prontas, vamos agora disponibilizar as informações do repositório selecionado.

Nosso método completo vai ficar desta forma:

    void showRepositoryInfo(List<Repository> repos) {
      stdout.write('Digite o número do repositório para receber mais informações:');
      final input = stdin.readLineSync();
      try {
        if (input == null || input.trim().isEmpty) {
          throw Exception(
              'Digite o número à esquerda do repositório que você deseja analisar!');
        }
        String filteredInput = input.trim().replaceAll(RegExp(r'\[|\]'), '');
        int index = int.parse(filteredInput);
        final Repository repo = repos[index];
        // send formatted output
        stdout.writeln('-' * (repo.full_name.length + 12));
        stdout.writeln('\nRepositório ${repo.full_name}\n');
        stdout.writeln('Página no GitHub: ${repo.html_url}\n');
        stdout.writeln(
            'Criado em ${repo.created_at} - Última atualização em ${repo.updated_at}\n');
        stdout.writeln('Visibilidade: ${repo.private ? 'PRIVADO' : 'PÚBLICO'}\n');
        stdout.writeln('URL para clonar (ssh): ${repo.ssh_url}\n');
        stdout.writeln(
            'Issues em aberto (quantidade): ${repo.open_issues} - ${repo.html_url}/issues \n');
        if (repo.description != null && repo.description!.length > 0) {
          stdout.writeln('Descrição:\n${repo.description}\n');
        }
        if (repo.language != null) {
          stdout.writeln('Linguagem: ${repo.language}\n');
        }
      } on FormatException {
        stderr.writeln('Por favor, digite um número de 1 a ${repos.length}!');
      } on RangeError {
        stderr.writeln(
            'Repositório não encontrado! Por favor, digite um número de 1 a ${repos.length}!');
      } catch (e) {
        stderr.writeln(e);
      }
    }

### Utilizando um bloco while para manter o programa rodando

Para permitir que o usuário possa testar vários repositórios, vamos realizar uma simples adição em nosso `main()` dentro de `oauth_server.dart`.

    void main() async {
      final EnvironmentHandler env =
          await EnvironmentHandler.loadEnv('.env', requiredKeys);
      clientId = env.getVar('GITHUB_CLIENT_ID');
      clientSecret = env.getVar('GITHUB_CLIENT_SECRET');
      final ServerHandler serverHandler = ServerHandler(clientId, clientSecret);
      final HttpServer server = await serverHandler.createServer();
      BrowserHandler.launchBrowser('$authorizationUrl$clientId');
      await serverHandler.handleRequests(server);
      final TokenModel token = UniqueToken.instance.tokenModel();
      final User user = await getAuthenticatedUser(token);
      showAuthUser(user);
      List<Repository> repos = await getUserRepositories(token);
      showUserRepositories(repos);
      while (true) { // ADICIONAR
        showRepositoryInfo(repos);
      }
    }

Pronto! O fluxo inicial do nosso app está completo.

Você pode testar o funcionamento com `dart run` na raíz do projeto.

Repare que náo precisamos mais passar o arquivo `oauth_server.dart` como parâmetro, devido à estrutura de colocar os arquivos `.dart` dentro do diretório `bin`.

<a title="Clique para ver em formato de vídeo" href="./imgs/apps-de-linha-de-comando-em-dart/working_comp.mp4">![Funcionamento do app com escolha de repositório](./imgs/apps-de-linha-de-comando-em-dart/working_comp.jpg)</a>

## Listandos os issues abertos de cada repositório

Para listarmos os _issues_, vamos precisar de dois novos models: `Issue` e  `Label`.

Enquanto os _labels_ não são realmente necessários, vão servir como uma informação adicional, sem muita dificuldade de ser implementada.

Os dois models serão gerados utilizando o `built_value`.

Comece criando, em `models`, o arquivo `issue_model`.

Neste caso, optei por selecionar apenas as chaves que eu considero importantes para a nossa aplicação. Você pode ver todas as chaves disponíveis [neste link](https://docs.github.com/en/rest/reference/issues#list-repository-issues), na aba `Response`.


    import 'user_model.dart';
    import 'label_model.dart';
    import 'package:built_value/built_value.dart';
    import 'package:built_value/serializer.dart';
    import 'package:built_collection/built_collection.dart';
    import 'serializers.dart';

    part 'issue_model.g.dart';

    abstract class Issue implements Built<Issue, IssueBuilder> {
      int get id;
      String get node_id;
      String get url;
      String get repository_url;
      String get labels_url;
      String get comments_url;
      String get events_url;
      String get html_url;
      int get number;
      String get state;
      String get title;
      String get body;
      User get user;
      BuiltList<Label> get labels;
      User get assignee;
      BuiltList<User> get assignees;
      bool get locked;
      String? get active_lock_reason;
      String? get closed_at;
      String get created_at;
      String get updated_at;
      User? get closed_by;
      String? get author_association;

      Issue._();
      factory Issue([void Function(IssueBuilder)? updates]) = _$Issue;

      static Issue fromJson(Map<String, dynamic> jsonObj) {
        return serializers.deserializeWith(Issue.serializer, jsonObj)!;
      }

      static Serializer<Issue> get serializer => _$issueSerializer;
    }

Repare que estamos importando uma classe \(e um arquivo\) que ainda não existem.

Vamos criar também `models/labels`:

    import 'package:built_value/built_value.dart';
    import 'package:built_value/serializer.dart';

    part 'label_model.g.dart';

    abstract class Label implements Built<Label, LabelBuilder> {
      int get id;
      String get node_id;
      String get url;
      String get name;
      String get description;
      String get color;

      Label._();
      factory Label([void Function(LabelBuilder)? updates]) = _$Label;

      static Serializer<Label> get serializer => _$labelSerializer;
    }

Você pode conferir todas as chaves disponíveis para os _labels_ em <https://docs.github.com/en/rest/reference/issues#get-a-label>.

Por último, vamos adicionar a classe `Issue` na listagem do nosso `Serializers`, para que possamos utilizar o método `fromJson`.

Em `models/serializers.dart`:

    import 'package:built_collection/built_collection.dart';
    import 'package:built_value/standard_json_plugin.dart';
    import 'package:built_value/serializer.dart';

    import 'repository_model.dart';
    import 'user_model.dart';
    import 'issue_model.dart'; // ADICIONAR
    import 'label_model.dart'; // ADICIONAR


    part 'serializers.g.dart';

    @SerializersFor([Repository, User, Issue]) // ALTERAR
    final Serializers serializers =
        (_$serializers.toBuilder()..addPlugin(StandardJsonPlugin())).build();

Repare que, mesmo não utilizando a classe `Label` diretamente no `Serializers`, precisamos importar o arquivo.

Isso é pois vamos utilizá-la na serialização dos `Issue`s.

Agora, podemos rodar no terminal, na raíz do projeto:

    dart run build_runner build

Isso vai gerar os arquivos necessários.

### Realizando a requisição dos issues na API

Vamos criar a função responsável pela requisição em `requests/issues.dart`.

    import '../models/issue_model.dart';
    import '../models/repository_model.dart';
    import '../models/token_model.dart';

    import 'dart:convert';
    import 'package:http/http.dart' as http;

    Future<List<Issue>> getIssuesFromRepo(
        TokenModel tokenModel, Repository repo) async {
      final http.Client client = http.Client();
      final String requestUrl =
          'https://api.github.com/repos/${repo.full_name}/issues';
      final http.Response response = await client.get(
        Uri.parse(requestUrl),
        headers: {
          "Accept": "application/vnd.github.v3+json",
          "Authorization": "token ${tokenModel.accessToken}",
        },
      );

      if (response.statusCode != 200) {
        throw Exception('Ocorreu um erro na conexão com a API do GitHub!');
      }
      final List<dynamic> payload = json.decode(response.body);
        List<Issue> issues =
          payload.map((data) => Issue.fromJson(data)).toList();

      return issues;
    }

Para realizar a requisição, vamos precisar tanto do _token_ com as chaves de acesso, quanto do repositório.

Para isso, precisamos alterar o método `showRepositoryInfo`, que recebe o repositório escolhido para o usuário.

A ideia é que ele retorne o repositório, para ser usado na requisição dos _issues_.

Dentro do arquivo `output/repository.dart`:

    Repository? showRepositoryInfo(List<Repository> repos) { // ALTERAR

E depois

      if (repo.language != null) {
        stdout.writeln('Linguagem: ${repo.language}\n');
      }
        return repo; // ADICIONAR
      } on FormatException {
        stderr.writeln('Por favor, digite um número de 1 a ${repos.length}!');
        return null;    // ADICIONAR
      } on RangeError {
        stderr.writeln(
            'Repositório não encontrado! Por favor, digite um número de 1 a ${repos.length}!');
        return null;  // ADICIONAR
      } catch (e) {
        stderr.writeln(e);
        return null;  // ADICIONAR
      }

Vamos retornar o repositório selecionado pelo usuário, ou `null` caso ocorra algum erro.

Podemos agora chamar a função no nosso `main()`:

      while (true) {
        final Repository? repo = showRepositoryInfo(repos);
        if (repo != null) {
          List<Issue> issues = await getIssuesFromRepo(token, repo);
        }
      }

### Mostrando uma listagem dos issues em aberto

Tendo carregado os valores na variável `issues`, vamos criar uma função para renderizá-los no terminal:

Crie em `output` o arquivo `issues.dart`, com o conteúdo abaixo:

    import 'dart:io';
    import '../models/issue_model.dart';

    void showRepositoryIssues(List<Issue> issues) {
      stdout.writeln('Issues em aberto:');
      for (var i = 0; i < issues.length; i++) {
        if (issues[i].closed_at == null) {
          stdout.writeln('    ${i + 1}. ${issues[i].title.toUpperCase()}');
          stdout.writeln('        Aberto em: ${issues[i].created_at}');
          if (issues[i].labels.isNotEmpty) {
            final String issueNames =
                issues[i].labels.map((el) => el.name).join(',');
            stdout.writeln('        Labels: $issueNames');
          }
          if (issues[i].body != null) {
            final List<String> splitDescr = issues[i].body!.split('\n');
            splitDescr.forEach((element) {
              stdout.writeln('        $element');
            });
          }
        }
        stdout.writeln();
        stdout.writeln();
      }
    }

Agora você pode chamar esta função no `main()`:

    void main() async {
      final EnvironmentHandler env =
          await EnvironmentHandler.loadEnv('.env', requiredKeys);
      clientId = env.getVar('GITHUB_CLIENT_ID');
      clientSecret = env.getVar('GITHUB_CLIENT_SECRET');
      final ServerHandler serverHandler = ServerHandler(clientId, clientSecret);
      final HttpServer server = await serverHandler.createServer();
      BrowserHandler.launchBrowser('$authorizationUrl$clientId');
      await serverHandler.handleRequests(server);
      final TokenModel token = UniqueToken.instance.tokenModel();
      final User user = await getAuthenticatedUser(token);
      showAuthUser(user);
      List<Repository> repos = await getUserRepositories(token);
      showUserRepositories(repos);
      while (true) {
        final Repository? repo = showRepositoryInfo(repos);
        if (repo != null) {
          List<Issue> issues = await getIssuesFromRepo(token, repo);
          showRepositoryIssues(issues);
        }
      }
    }

Pronto! Agora temos uma listagem das informações de cada repositório, com seus issues em aberto.

## Considerações finais

Neste post, nós vimos como criar as ferramentas básicas necessárias para configuração de um aplicativo de terminal escrito em Dart.

Criamos uma classe dedicada para lidar com a leitura e escrita de informações num arquivo _.env_.

Criamos, utilizando a biblioteca `built_values`, modelos dos dados recebidos do GitHub \(usuários, repositórios e issues\).

Criamos um fluxo básico para uso do aplicativo, onde o usuário pode escolher o repositório desejado, e receber mais informações.

Existem alguns pontos que precisam ser melhorados, caso você queira ampliar este aplicativo: Atualmente, nossa função `showUserRepositories` recebe diretamente os dados digitados pelo usuário.

Para possibilitar mais ações do que a simples seleção de um repositório \( como por ex. aperta `q` para finalizar o programa, ou apertar alguma tecla para abrir um novo issue \), precisariamos criar uma interface responsável por receber todo input do usuário e direcioná-lo para a função ou método correto.

Também precisamos de uma forma de persistir os dados recebidos da API do GitHub, diminuindo a quantidade de consultas e evitando que o usuário tenha que abrir a página de autenticação toda vez que rodar o programa.

No mais, espero que tenha entendido como podemos iniciar a estruturação de um aplicativo interativo no terminal, usando a linguagem Dart.

Abraço!

Fontes:

- <https://dart.dev/tutorials/server/cmdline>
- <https://api.dart.dev/stable/2.16.1/dart-io/File-class.html>
- <https://www.youtube.com/watch?v=hNbOSSgpneI>
- <https://charafau.github.io/json2builtvalue/>
- <https://pub.dev/packages/built_value>
- <https://www.youtube.com/watch?v=XoZ01mY-cUg&list=PLNnAcB93JKV9YAl4QDygDEMfplrt5uUUx&index=6>

Tags: dart, cli, api, github
