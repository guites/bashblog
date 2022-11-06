Busca sem backend - parte 1

<abbr title="muito grande;nem li">mg;nl</abbr>: Vamos ver como adicionar um mecanismo de busca em um site estático. Neste post vamos criar um script que gera um arquivo json contendo os conteúdos do site, juntando o título, sumário, tags e palavras chave geradas utilizando a ferramenta [yake](https://github.com/LIAAD/yake).

A barra de busca: essencial, mas de dificil implementação se você não tiver acesso a um banco de dados. 

Em sites dinâmicos, no qual o conteúdo é resgatado de um banco de dados a cada acesso, você pode utilizar um formulário, que envia os valores digitados pelo usuário (termos de busca) a um endpoint que as utiliza para realizar uma consulta no banco de dados, e retorna os resultados para o usuário.

Em sites estáticos \(como esse, que é hospedado no [github pages](https://pages.github.com/)\), todo o conteúdo disponível para o usuário precisa ser previamente criado e hospedado. Ou seja, não temos como responder de forma dinâmica às combinações \(praticamente infinitas\) de palavras chave que um usuário pode buscar.

Para um site pequeno, porém, podemos estruturar o conteúdo em um formato de fácil consulta, como o [json](https://www.json.org/json-en.html), e deixá-lo salvo no servidor. 

Daí podemos receber os termos de busca digitados pelo usuário e usar algumas linhas de javascript para carregar o arquivo, aplicar o filtro, e retornar os resultados.

Neste primeiro post, vamos desenvolver um script em python que percorre as publicações do site e formata as informações essenciais em um formato facilmente consultável.

<hr/>

## Visão geral do projeto

Nosso objetivo é criar um script que acesse os arquivos `html` do blog e gere um arquivo no formato:

    [
      {
        "title":"Título do post",
        "permalink":"link-do-post.html",
        "summary":"mg;nl: Este post é sobre ...",
        "keywords":"palavras chave geradas automaticamente usando yake",
        "tags":"tags do post original"
      },
      {
        ...
      }
    ]

Para que depois possamos consultar via javascript.

Repare no campo "keywords": as tags aqui do site são definidas manualmente \(eu invento tags conforme acho relevante\), e disso acontece que elas nem sempre refletem com precisão o assunto coberto pelo post.

Resolvi utilizar um processo chamado _Automatic Keyword Extraction_, que analisa o conteúdo de um texto arbritário e extrai palavras chaves.

Eu não vou me aprofundar muito no funcionamento desse processo, mas você pode ler mais sobre no [github do projeto](https://github.com/LIAAD/yake).

## Iterando sobre uma listagem de arquivos

Vamos criar um script, `generate_json.py`.

Vamos supor uma estrutura onde cada post é um arquivo html dentro de um mesmo diretório:

      - generate_json.py
      - blog/
        - automatizando-a-geracao-de-site-map.html
        - barra-de-buscas-e-consulta-a-api-com-flutter.html
        - coisas-que-eu-nao-sabia-sobre-bash.html
        - criando-um-app-com-flutter-e-sqlite.html
        - depurando-php-com-xdebug-no-vs-code.html
        - mantendo-data-de-criacao-de-arquivos-no-git.html

Vamos começar iterando sobre os arquivos.

    # import required modules
    from pathlib import Path
    import os

    # assign directory
    directory = 'blog'

    invalid_prefixes = ['index', 'todos', 'todas', 'tag']

    # iterate over files in that directory
    files = Path(directory).glob('*.html')
    for file in files:

        skip = False
        basename = os.path.basename(file)
        for prefix in invalid_prefixes:
            if basename.startswith(prefix):
                skip = True
        if skip: continue

Adicionei uma pequena lógica usando uma lista de prefixos, `invalid_prefixes`. Você pode adicionar nela qualquer arquivo presente no seu diretório que você não queira incluir no arquivo json resultante.

Caso chegarmos na ultima linha e o valor da variável `skip` seja verdadeiro, pulamos o arquivo. Caso contrário, seguimos em frente.

## Carregando arquivos html em memória e acessando o valor das tags

Precisamos carregar os arquivos em memória e _parseá-los_, ou seja, acessar os valores dentro das tags \(h1, p, div...\).

Vamos utilizar a biblioteca Beautiful Soup, que facilita esse processo.

### Instalando o Beautiful Soup

Você pode instalar o pacote com

    pip3 install beautifulsoup4

Mas eu recomendo sempre criar um ambiente virtual pra rodar qualquer aplicação python3:

    python3 -m virtualenv venv # cria um ambiente virtual
    source venv/bin/activate # ativa o ambiente
    pip3 install beautifulsoup4

Você pode utilizar o conteúdo [deste arquivo](https://raw.githubusercontent.com/guites/bashblog/blog/blog/barra-de-buscas-e-consulta-a-api-com-flutter.html) para acompanhar os exemplos.

### Acessando o valor das tags html

Vamos adicionar o import necessário: 

        # import required modules
        from pathlib import Path
        import os
      + from bs4 import BeautifulSoup
      + import bs4

Carregar os arquivos, e acessá-los usando o `open` e `BeautifulSoup`:

      + print(f"{basename}:")
      + with open(file, 'r', encoding='UTF-8') as html_file:
      +     parsed = BeautifulSoup(html_file, 'html.parser')
      +     if parsed.h3:
      +         print(f"--- {parsed.h3.text.strip()}")

## Criando o arquivo json

Agora que temos acesso as tags, basta escolher quais iremos utilizar e formatá-las no arquivo json de saída.

O formato que queremos é o seguinte:

    [
      {
        "title":"Título do post",
        "permalink":"link-do-post.html",
        "summary":"mg;nl: Este post é sobre ...",
        "keywords":"palavras chave geradas automaticamente usando yake",
        "tags":"tags do post original"
      },
      {
        ...
      }
    ]

Vamos iniciar uma lista para guardar os valores.

        invalid_prefixes = ['index', 'todos', 'todas', 'tag']

      + output_list = []

        # iterate over files in that directory
        files = Path(directory).glob('*.html')
        for file in files:

E iniciar um dicionário vazio para cada arquivo encontrado.

      - print(f"{basename}:")
        with open(file, 'r', encoding='UTF-8') as html_file:
        +  post_dictionary = {
        +       "title": "",
        +       "permalink": "",
        +       "summary": "",
        +       "keywords": "",
        +       "tags": ""
        +   }

Novamente, cada site vai ter um padrão nas suas postagens, e aqui no blog, tanto o título quanto a URL da página ficam nesse formato,

    <h3><a class="ablack" href="apps-de-linha-de-comando-em-dart.html">
    Apps de linha de comando em dart
    </a></h3>

Que podemos acessar com 
      
       -  if parsed.h3:
       -    print(f"--- {parsed.h3.text.strip()}")
       +  if parsed.h3: post_dictionary['title'] = parsed.h3.text.strip()
       +  if parsed.select('h3 a'): post_dictionary['permalink'] = parsed.select('h3 a')[0]['href'].strip()

<aside>Repare que o `.select('h3 a')` é equivalente a um `document.querySelector` do javascript.</aside>

### Acessando o summary

O `summary` vai ser um pouco mais complicado, pois envolve lidar com o texto "cru" da postagem \(plaintext\), que é no seguinte formato: 

    <div id="divbody"><div class="content">
    <!-- entry begin -->
    <h3><a class="ablack" href="barra-de-buscas-e-consulta-a-api-com-flutter.html">
    Barra de buscas e consulta a API com Flutter
    </a></h3>
    <!-- bashblog_timestamp: #202202022210.37# -->
    <div class="subtitle">Fevereiro 02, 2022 &mdash; 
    Guilherme Garcia
    </div>
    <!-- text begin -->

    <p><abbr title="muito grande; nem li">mg;nl</abbr>: Tutorial para criar um app em flutter que realiza requisições http numa API e filtra o resultado usando uma barra de busca.</p>

    <p>Então essa semana eu comecei a aprender desenvolvimento de apps em Flutter.</p>

    <p>Duas questões importantes em qualquer aplicativo, web ou mobile, é a possibilidade de acessar dados remotos, comumente em 
    formato JSON, e disponibilizar ao usuário uma busca, por nome ou palavras-chave.</p>

    <p>Dando uma procurada na net encontrei <a href="https://medium.com/flutter-community/parsing-complex-json-in-flutter-747c46655f51">este artigo</a>, com uma proposta
    muito boa: ele ensina a criar uma barra de busca, de uma fonte remota, usando as ferramentas incluídas no pacote básico do Flutter.  </p>

    <p>O artigo é um pouco datado (escrito ali por 2018) e utiliza algumas ferramentas já deprecadas.</p>

    <p>Neste post, vou analisar o processo do autor enquanto migro sua aplicação para a versão atual do Flutter (em janeiro/2022).</p>

    <hr/>

    <p>Vou partir do princípio de que você já tem o android studio instalado, com uma VM de um celular ou tablet, como o pixel X3, por exemplo.</p>

A parte que nos interessa começa em `<!-- text begin -->` e termina na primeira ocorrência da tag `<hr/>`.

O ideal seria que essa parte estivesse dentro de uma tag, digamos, `<div id="summary">`, e então poderiamos resgatá-la com 

    parsed.find(id='summary')

Mas, do jeito que está, o elemento mais próximo que temos acesso é a div com a classe "content".

Vamos utilizar a seguinte lógica: selecionamos a `div.content`, e iteramos sobre seus filhos \(tags que estiverem dentro dela\). O BeautifulSoup nos permite identificar o tipo de cada tag, e conseguimos verificar se uma tag é um comentário.

Caso o comentário tiver o texto "text begin", vamos começar a salvar o conteúdo de todas as tags que forem aparecendo, até chegarmos em uma tag do tipo `<hr/>`.

          if parsed.select('h3 a'): post_dictionary['permalink'] = parsed.select('h3 a')[0]['href'].strip()
        
        + is_summary = False
        + summary = ""
        + content = parsed.select_one('div.content')
        + for child in content.children:
        +     if (type(child) == bs4.element.Comment):
        +         if (child.strip() == 'text begin'):
        +             is_summary = True
        +     if child.name == "hr":
        +         is_summary = False
        +         break
        +     if is_summary:
        +         summary = summary + child.text
        + post_dictionary['summary'] = summary

Para decidirmos quais tags devem ser capturadas, vamos utilizar a flag `is_summary`.

Agora faltam apenas as tags e as keywords.

### Pegando as tags de cada post

As tags estão, no html do post, no seguinte formato

    <p>Tags: <a href='tag_flutter.html'>flutter</a>, <a href='tag_dart.html'>dart</a></p>

Novamente, um `id` ou uma `class` ajudariam, mas podemos resgatá-las usando um pequeno filtro.

           post_dictionary['summary'] = summary

        +  content_p_tags = content.select('p')
        +  for p_tag in content_p_tags:
        +      p_tag_text = p_tag.get_text()
        +      if p_tag_text.startswith('Tags:'):
        +          post_dictionary['tags'] = p_tag_text.replace('Tags: ','')
        +          break

Iteramos por todas as tags `p` dentro da variável `content` \(que usamos para encontrar nosso summary\), e pegamos o conteúdo daquela que começar com "Tags: ".

## Gerando as keywords com a biblioteca YAKE

YAKE significa "Yet Another Keyword Extractor" \("Mais um extrator de palavras chave" :P\). A ideia é que ele realize esse processo, que costuma ser bem teórico e trabalhoso, de forma rápida, com um pacote leve e simples de utilizar.

O [github do projeto](https://github.com/LIAAD/yake) tem uma ótima explicação do funcionamento da ferramenta, e aqui vamos focar em como utilizá-la em conjunto com nosso script.

Primeiro, precisamos instalá-la.

    pip3 install git+https://github.com/LIAAD/yake

E colocar um `import` no topo do nosso script

        # import required modules
        from pathlib import Path
        import os
        import bs4
        from bs4 import BeautifulSoup
      + import yake

E vamos também inicializar o extrator, passando alguns parâmetros necessários:

- lan \(language\): vamos usar `pt`, pois o padrão é inglês. Eu acho que funciona bem tanto pra português europeu quanto brasileiro. Inclusive o [exemplo utilizado no site do yake](http://yake.inesctec.pt/demo/sample/sample15) é sobre a seleção brasileira.
- n \(max ngram size\): Tamanho máximo dos ngrams coletados no resultado. Um ngram é um conjunto de palavras considerado como um único resultado. Por exemplo, "programação" seria um 1-gram, enquanto "acesso livre" seria um 2-gram, etc.
- dedupLim \(deduplication limiar\): Valor de corte utilizado na remoção de keywords consideradas duplicadas. Age em conjunto com o parâmetro `dedupFunc` na hora de decidir o que é duplicado e o que não é.
- dedupFunc \(deduplication function\): Função utilizada na remoção de keywords consideradas duplicadas. Pode ter um dos três valores: `leve|jaro|seqm`.
- top: número de palavras chaves que a extração deve retornar.

        from bs4 import BeautifulSoup
        import yake

      + custom_kw_extractor = yake.KeywordExtractor(lan='pt', n=3, dedupLim=0.3, dedupFunc='seqm', top=35)

<aside>Deixei os parâmetros nesse formato pra facilitar a visualização. Achei que o resultado fica mais fiél ao conteúdo das postagens usando `n=2` e `top=20`</aside>

Agora, basta aplica nosso extrator em cada página.

Um detalhe importante é a decisão de qual parte da página vai ser utilizada. No caso do blog, optei por usar apenas o texto visível dentro daquela div `.content`, que possui o conteúdo da postagem.

Ficou da seguinte forma:

              if p_tag_text.startswith('Tags:'):
                  post_dictionary['tags'] = p_tag_text.replace('Tags: ','')
                  break
          
      +   keywords = custom_kw_extractor.extract_keywords(content.get_text())
      +   for kw in keywords:
      +       print(kw)

O resultado do print, pro post usado de exemplo, fica assim:

    ('Adicionar', 0.0028274157124887687)
    ('Guilherme Garcia', 0.005811470715375073)
    ('Flutter', 0.013492457875624883)
    ('return', 0.016910840426021857)
    ('Breed', 0.01912511108143883)
    ('Text', 0.019214247534260948)
    ('widget', 0.019297320635162443)
    ('API', 0.022457665678024995)
    ('dados', 0.027917870615386184)
    ('lista', 0.030639494714601857)
    ('http', 0.03291428909366911)
    ('Barra', 0.03321703879633509)
    ('método', 0.03818383547178414)
    ('SearchPageState', 0.038862643368540825)
    ('Key', 0.03974743900536119)
    ('formato', 0.04344967620887332)
    ('Icons.search', 0.043656612355202365)
    ('String', 0.05005954258384762)
    ('função', 0.06279564452195556)
    ('responseObject', 0.07139370923874819)
    ('requisições http', 0.0721761189386072)
    ('usuário', 0.073361973736161)
    ('listagem de raças', 0.07681885961917626)
    ('class', 0.08299965908564721)
    ('JSON', 0.08444435407204463)
    ('override', 0.08531633811080994)
    ('buildListOfBreeds', 0.10062577219435372)
    ('appBarTitle', 0.10231932155808073)
    ('dynamic', 0.10265887744947501)
    ('seja', 0.10738858527930627)
    ('variável', 0.10965849337311194)
    ('extends', 0.1229440576959018)
    ('import', 0.12343809691625732)
    ('arquivo', 0.12670392651326004)
    ('message', 0.12672339486946965)

Quanto mais próximo de zero for o valor, mais relevante é a palavra chave.

Uma coisa que eu não gostei, é que muitos resultados são palavras encontradas dentro de tags `<code>`, pois coloco muitos exemplos nos posts. Eu acredito que o contexto do uso de YAKE é para texto natural, então vou primeiro filtrar os blocos de código presentes no corpo da postagem.

### Filtrando os blocos de código antes de aplicar o extrator

Na escrita dos textos do blog, fiz uma escolha arbitrária: código escrito dentro da mesma linha do resto do texto fica dentro de uma tag `<code>`, e blocos de código grandes o bastante pra ter sua própria linha e identação, ficam dentro de um `<pre><code>`, pra manter a formatação quando eu usar <kbd>ctrl + c</kbd> e <kbd>ctrl + v</kbd>.

Então o que precisamos fazer é clonar o `content` original e ir removendo todas as tags `<pre>` que existirem dentro dele.

A documentação do BeautifulSoup recomenda a clonagem de objetos com o pacote `copy`. Vamos importá-lo

        from bs4 import BeautifulSoup
        import yake
     +  import copy

E para cada arquivo html, vamos remover todas as tags indesejadas, e depois utilizaremos nossa cópia filtrada como argumento no extrator.

         + content_copy = copy.copy(content)
         + for child in content_copy.children:
         +     if child.name == 'pre':
         +        child.extract()

         - keywords = custom_kw_extractor.extract_keywords(content.get_text())
         + keywords = custom_kw_extractor.extract_keywords(content_copy.get_text())
           for kw in keywords:
               print(kw)

Podemos ver uma mudança grande nas keywords geradas:

    ('Guilherme Garcia text', 0.004532153520049423)
    ('Garcia text begin', 0.012948359380401397)
    ('Flutter', 0.016925800324343994)
    ('API', 0.02055282206994107)
    ('dados', 0.023249378371687807)
    ('lista', 0.025973503165796163)
    ('método', 0.03175054696754484)
    ('Adicionar', 0.03280023890613875)
    ('propriedade', 0.034090023290055324)
    ('formato', 0.03606688281671043)
    ('raça', 0.04101614672715731)
    ('http', 0.04219519416718389)
    ('requisição', 0.045245453868184786)
    ('widget', 0.05301021972241176)
    ('Tutorial', 0.07197539997471863)
    ('begin Barra', 0.07887160161743723)
    ('JSON', 0.09724311315576666)
    ('android', 0.10759621380487877)
    ('qualquer', 0.1170957486746625)
    ('Breed', 0.12396985431611546)
    ('código', 0.13581759727238618)
    ('import', 0.14748220665792972)
    ('criando', 0.15749912021084772)
    ('variáveis', 0.16391981569246364)
    ('será', 0.16434553439229052)
    ('precisamos realizar', 0.17273564442135858)
    ('package', 0.21369426404160313)
    ('classes', 0.2220703835489235)
    ('dynamic', 0.22544049880440167)
    ('erro', 0.23362063816061)
    ('listagem de raças', 0.23492778889638644)
    ('função fetchBreeds', 0.2467958041484861)
    ('consulta a API', 0.2524130851954772)
    ('API e filtra', 0.2524130851954772)
    ('bashblog', 0.29547454398648465)

Algumas keywords ainda estão repetidas, e outras não refletem o conteúdo do post em si \("Guilherme Garcia" e "Garcia", por exemplo\), mas isso pode ser resolvido com filtros adicionais, ou mexendo nos parâmetros do extrator.

<aside>O meu nome aparece provavelmente devido a posição que ele tem no texto \(aparece duas vezes logo no início\)</aside>

### Adicionando os resultados do extrator ao dicionário

Quando você estiver satisfeito com as palavras chave encontradas, podemos passar pro último passo, que é adicioná-las ao resultado.

         - keywords = custom_kw_extractor.extract_keywords(content.get_text())
         + keywords = custom_kw_extractor.extract_keywords(content_copy.get_text())
           for kw in keywords:
               print(kw)

         keywords = custom_kw_extractor.extract_keywords(content_copy.get_text())
      +  keyword_string = ""
      -  for kw in keywords:
      +   for keyword, score in keywords:
      +       keyword_string = keyword_string + keyword + " "
      +   post_dictionary['keywords'] = keyword_string

Como os resultados do extrator são um `tuple`, a palavra chave em si vai estar sempre na primeira posição.

Precisamos também adicionar o resultado a nossa listagem, `output_list`

          post_dictionary['keywords'] = keyword_string
       +  output_list.append(post_dictionary)

## Salvando o resultado em um arquivo json

Para escrever o resultado num arquivo, vamos utilizar o pacote `json`.

        import yake
        import copy
      + import json

E, após iterar por todos os arquivos válidos, vamos salvar nosso dicionário com o `json.dump()`.

                post_dictionary['keywords'] = keyword_string
                output_list.append(post_dictionary)

     +  with open("search_content.json", "w", encoding='UTF-8') as json_file:
     +      json.dump(output_list, json_file)

## Considerações finais

Nesse post vimos como podemos condensar o conteúdo de diversas páginas em um arquivo de fácil consulta, utilizando um processador de linguagem natural pra extrair algumas palavras-chave do texto.

Você pode analisar os exemplos \([1](http://yake.inesctec.pt/demo/sample/sample2),[2](http://yake.inesctec.pt/demo/sample/sample12),[3](http://yake.inesctec.pt/demo/sample/sample5)...\) presentes no site do YAKE se quiser entender melhor como a formatação do texto e os parâmetros utilizados influenciam no resultado.

Etapas adicionais de preparação do conteúdo das páginas do blog teriam certamente melhorado a qualidade dos resultados, mas a ideia aqui é mostrar de forma rápida como uma ferramenta desse tipo pode agregar valor a um projeto, mesmo que simples.

Pretendo dar continuidade ao tema da busca em sites estáticos, onde o json gerado nesse post vai fazer o papel de backend para gerar os resultados da busca.

Abraço.

## Fontes:

- <https://unix.stackexchange.com/a/264972>
- <https://matix.io/extract-text-from-webpage-using-beautifulsoup-and-python/>
- <https://www.educative.io/answers/what-are-n-grams>
- <https://www.crummy.com/software/BeautifulSoup/bs4/doc/#copying-beautiful-soup-objects>
- <https://www.crummy.com/software/BeautifulSoup/bs4/doc/#extract>

Tags: python, json, busca, nlp



    [
      {
        "title":"Automatizando a geração de site map",
        "permalink":"automatizando-a-geracao-de-site-map.html",
        "published_date":"Fevereiro 06, 2022 ",
        "summary":"mg;nl: Vamos falar sobre o porquê ...",
        "keywords":"sitemap arquivo loc lastmod url tag RSS https Formato post ",
        "tags":" seo  bash  bashblog"
      },
      {
        "title":"Barra de buscas e consulta a API com Flutter",
        "permalink":"barra-de-buscas-e-consulta-a-api-com-flutter.html",
        "published_date":"Fevereiro 02, 2022 ",
        "summary":"mg;nl: Tutorial para criar um app em flutter que ...",
        "keywords":"Adicionar breeds listagem return Flutter Text API dados Barra http ",
        "tags":" flutter  dart"
      },
    ]
