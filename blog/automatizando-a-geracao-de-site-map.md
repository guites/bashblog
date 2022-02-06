Automatizando a geração de site map

<abbr title="muito grande; nem li">mg;nl</abbr>: Vamos falar sobre o porquê e como adicionar um gerador de sitemap pro [bashblog](https://github.com/guites/bashblog), atualizado sempre que um post for adicionado, editado ou removido. 

Neste post vamos entender a importância de utilizar um sitemap e como ele pode ser gerado "manualmente" para sites estáticos como esse blog.

Faço um panorama dos formatos de sitemap existente, suas diferenças, e depois proponho uma função, escrita em bash, para gerar um sitemap baseado numa listagem de arquivos html presentes num diretório. 

<hr/>

## A importância de um sitemap

Os sitemaps são usados para auxílio na indexação. Com a enorme quantidade de sites existentes, sites podem ficar um tempo
muito longo sem suas páginas indexadas pelos mecanismos de busca.

Isso ocorre principalmente se o site em questão não possui muitos [backlinks](https://en.wikipedia.org/wiki/Backlink), ou seja, 
não existem muitos outros sites que apontam pra ele.

Como os links são pouco citados, acabam não entrando no radar dos bots de indexação, o que resultado no site ficar fora dos resultados de busca.

O sitemap é uma forma padronizada de listar todos os recursos acessáveis de um site. Tanto o nome quanto a localização do arquivo é padronizada, o facilita seu descobrimento por um bot, como o bingbot, googlebot, etc.

## Formatos válidos para um sitemap

A formatação do arquivo sitemap pode ser feita de diversas formas.

1. Formato XML

  A formatação em XML é a mais flexível, pois permite a adição de parametros que indicam para os bots a última modificação de cada página, a frequência com a qual ela é alterada e também um nível de prioridade, de 0 a 1, interpretado de forma relativa pelo bot, para melhor compreender partes do site com conteúdo considerado mais expressivo.

  Um exemplo de sitemap em XML pode ser encontrado no site <https://www.sitemaps.org/protocol.html>

        <?xml version="1.0" encoding="UTF-8"?>
        <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
           <url>
              <loc>http://www.example.com/</loc>
              <lastmod>2005-01-01</lastmod>
              <changefreq>monthly</changefreq>
              <priority>0.8</priority>
           </url>
           <url>
              <loc>http://www.example.com/catalog?item=12&amp;desc=vacation_hawaii</loc>
              <changefreq>weekly</changefreq>
           </url>
           <url>
              <loc>http://www.example.com/catalog?item=73&amp;desc=vacation_new_zealand</loc>
              <lastmod>2004-12-23</lastmod>
              <changefreq>weekly</changefreq>
           </url>
        </urlset>

  Repare o nível de especificidade das URLs. Para sites grandes, o arquivo sitemap pode ser fragmentado em mais de um, agrupando
  páginas de conteúdo parecido.

  Um bom exemplo é o [sitemap do site da sony](https://www.sonymusic.com/sitemap_index.xml), que
  possui um _sitemap\_index.xml_, dentro do qual aponta para diversos sitemaps, cada com com uma listagem de urls pertencentes
  a um tópico diferente.

        <sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
          <sitemap>
            <loc>https://www.sonymusic.com/post-sitemap.xml</loc>
            <lastmod>2022-01-25T17:10:59+00:00</lastmod>
          </sitemap>
          <sitemap>
            <loc>https://www.sonymusic.com/videos-sitemap.xml</loc>
            <lastmod>2022-01-18T22:30:22+00:00</lastmod>
          </sitemap>
          <sitemap>
            <loc>https://www.sonymusic.com/labels-sitemap.xml</loc>
            <lastmod>2020-08-20T16:29:06+00:00</lastmod>
          </sitemap>
          <sitemap>
            <loc>https://www.sonymusic.com/artists-sitemap.xml</loc>
            <lastmod>2022-01-19T21:38:31+00:00</lastmod>
          </sitemap>
          <sitemap>
            <loc>https://www.sonymusic.com/executives-sitemap.xml</loc>
            <lastmod>2021-11-29T22:00:44+00:00</lastmod>
          </sitemap>
          <sitemap>
            <loc>https://www.sonymusic.com/category-sitemap.xml</loc>
            <lastmod>2022-01-25T17:10:59+00:00</lastmod>
          </sitemap>
        </sitemapindex>

  Repare que, se você acessar a URL do sitemap pelo seu navegador, a formatação vai ser diferente dessa que eu mostro aqui. 

  O seu navegador entende que se trata de um sitemap e o apresenta de uma forma mais legível.

  Você pode acessar a versão em texto do sitemap de qualquer site realizando uma requisição pela linha de comandos.

  Em um macos/linux, você pode utilizar algum dos comandos abaixo:

        curl https://www.sonymusic.com/sitemap_index.xml

        ## ou também

        wget -O - https://www.sonymusic.com/sitemap_index.xml

2. Formato RSS
    
  O Formato RSS, comumente usado em feeds de site, também pode ser utilizado como sitemap.

  Um exemplo desse tipo de arquivo é o [RSS aqui do blog](https://guilhermegarcia.dev/blog/feed.rss).

        <?xml version="1.0" encoding="UTF-8"?>
        <rss xmlns:atom="http://www.w3.org/2005/Atom" xmlns:dc="http://purl.org/dc/elements/1.1/" version="2.0">
           <channel>
            <title>Blog do guilherme</title>
            <link>https://guilhermegarcia.dev/blog/index.html</link>
            <description>programação e outras coisas menos úteis</description>
            <language>en</language>
            <lastBuildDate>Thu, 03 Feb 2022 08:10:13 -0300</lastBuildDate>
            <pubDate>Thu, 03 Feb 2022 08:10:13 -0300</pubDate>
            <atom:link href="https://guilhermegarcia.dev/blog/feed.rss" rel="self" type="application/rss+xml" />
            <item>
               <title>Barra de buscas e consulta a API com Flutter</title>
               <description><![CDATA[<p><abbr title="muito grande; nem li">mg;nl</abbr>: Tutorial para criar um app em flutter que realiza requisições http numa API e filtra o resultado usando uma barra de busca.</p>
              <p>Então essa semana eu comecei a aprender desenvolvimento de apps em Flutter.</p>
              <!-- aqui teria todo o conteúdo do post, removi pra não tomar muito espaço -->
              <hr/>
              <p>Tags: <a href='tag_flutter.html'>flutter</a>, <a href='tag_dart.html'>dart</a></p>
              <p id='twitter'><a href='http://twitter.com/intent/tweet?url=https://guilhermegarcia.dev/blog/barra-de-buscas-e-consulta-a-api-com-flutter.html&text=&lt;Escreva seu comentário aqui mas mantenha a url para outras pessoas seguirem o comentário&gt;&via=gui_garcia67'>Comentários? Tweet</a>
              <a href='https://twitter.com/search?q=https://guilhermegarcia.dev/blog/barra-de-buscas-e-consulta-a-api-com-flutter.html'><span id='count-6742'></span></a>&nbsp;</p>]]></description>
                       <link>https://guilhermegarcia.dev/blog/barra-de-buscas-e-consulta-a-api-com-flutter.html</link>
                       <guid>https://guilhermegarcia.dev/blog/./barra-de-buscas-e-consulta-a-api-com-flutter.html</guid>
                       <dc:creator>Guilherme Garcia</dc:creator>
                       <pubDate>Wed, 02 Feb 2022 22:10:37 -0300</pubDate>
            </item>
          </channel>
        </rss>

  Os mecanismos de busca utilizam duas informações presentes nos feeds RSS para indexação: o campo `<link>` que indica a URL da página e o campo `<pubDate>`, que indica a ultima atualização da página.

  Enquanto o uso de um arquivo RSS como sitemap é perfeitamente válido, um ponto negativo é que os feeds costumam mostrar apenas os últimos 10 ou 15 posts, e muitas vezes omitem outras partes do site, listando
  apenas as postagens do blog.

3. Arquivo de texto simples

  Esse é o formato mais simples. Se trata de uma listagem das URLs do site, divididas em uma por linha, sem nenhuma ordem específica. 

  Exemplo:

        https://guilhermegarcia.dev/blog/todos_posts.html
        https://guilhermegarcia.dev/blog/todas_tags.html
        https://guilhermegarcia.dev/blog/tag_flutter.html
        https://guilhermegarcia.dev/blog/tag_dart.html
        https://guilhermegarcia.dev/blog/tag_bashblog.html
        https://guilhermegarcia.dev/blog/tag_bash.html
        https://guilhermegarcia.dev/blog/ola-mundo.html
        https://guilhermegarcia.dev/blog/index.html
        https://guilhermegarcia.dev/blog/coisas-que-eu-nao-sabia-sobre-bash.html
        https://guilhermegarcia.dev/blog/barra-de-buscas-e-consulta-a-api-com-flutter.html

  Esse arquivo tem informações suficientes para guiar a indexação do site por parte dos mecanismos de busca.

  Existem algumas limitações que devem ser levadas em conta:

  - O arquivo deve ter uma URL por linha
  - A URL deve ser especificada de forma completa, incluindo o protocolo \(http ou https\).
  - Cada arquivo (sitemap1.txt, sitemap2.txt, etc) pode ter no máximo 50000\(!\) URLs e não mais do que 50MB. Passando esse limite, a listagem de URLs deve ser separada em multiplos arquivos de texto.
  - O arquivo pode ser comprimido com gzip. Isso agiliza o acesso ao sitemap e reduz o consumo de banda. Para sites com muito acesso \(e muitas URLs!\), isso pode significar uma redução expressiva no custo da hospedagem.

  Este formato ser tão simples de ser gerado foi o motivo pelo qual eu escolhi utilizar ele aqui no blog. Na seção abaixo eu mostro como extendi o código do [bashblog](https://github.com/cfenollosa/bashblog) para adicionar uma geração automática do arquivo sitemap com cada publicação.

## Gerando um sitemap de forma automática

Para sites estáticos, gerar um sitemap é uma tarefa tranquila.

Nesse exemplo vou usar bash, mas a mesma lógica pode ser aplicada a qualquer linguagem \(php, node, etc\).

Como todas as páginas existem em formato .html na raíz do site, podemos primeiro localizá-las e fazer um laço.

Podemos listar todos os arquivos no formato html passando a seguinte expressão no comando `ls`.

    ls -t *.html

A flag `-t` ordena o resultado, colocando os arquivos modificados mais recentemente por primeiro.

A expressão `*.html` restringe os resultados a arquivos no diretório presente, com a extensão .html.

    todos_posts.html                                  tag_dart.html                                     ola-mundo.html                                    barra-de-buscas-e-consulta-a-api-com-flutter.html
    todas_tags.html                                   tag_bashblog.html                                 index.html
    tag_flutter.html                                  tag_bash.html                                     coisas-que-eu-nao-sabia-sobre-bash.html

Essa saída pode ser usada para alimentar um while loop. A sintaxe é a seguinte:

    while read -r i; do
      echo $i;
    done < <(ls -t *.html);

output

    todos_posts.html
    todas_tags.html
    tag_flutter.html
    tag_dart.html
    tag_bashblog.html
    tag_bash.html
    ola-mundo.html
    index.html
    coisas-que-eu-nao-sabia-sobre-bash.html
    barra-de-buscas-e-consulta-a-api-com-flutter.html

Com este loop, podemos concatenar o nome do arquivo com a url do site, lembrando que os meus arquivos estão no diretório /blog.

    while read -r i; do
      echo "https://guilhermegarcia.dev/blog/"$i
    done < <(ls -t *.html);

    https://guilhermegarcia.dev/blog/todos_posts.html
    https://guilhermegarcia.dev/blog/todas_tags.html
    ...

Agora só falta direcionar este output pra um arquivo, sitemap.txt .

Dentro do while loop, podemos encapsular o output utilizando as chaves `{ }`.

    {
    while read -r i; do
        echo "https://guilhermegarcia.dev/blog"/$i
    done < <(ls -t *.html)
    } 3>&1 >"sitemap.txt"

[Neste post](https://guilhermegarcia.dev/blog/coisas-que-eu-nao-sabia-sobre-bash.html) falei um pouco sobre redirecionamento de streams.

Basicamente, os stream 0, 1 e 2 são reservados para os valores de entrada \(emitidos pelo nosso teclado\), os valores emitidos no terminal, e os valores de erro. Mas ainda temos os streams 3 ao 9, 
que são atribuídos automaticamente para qualquer arquivo aberto. No bash, [uma função é considerada como um arquivo](https://en.wikipedia.org/wiki/Everything_is_a_file), e isso nos permite acessar os valores de
saída do nosso loop.

Pronto! Agora você tem em mãos um sitemap do seu site.

### Adicionando o gerador de sitemap ao bashblog.

No bashblog, existem duas situações onde um novo arquivo é adicionado:

1. Um novo post é publicado: isso gera um arquivo para o post, por exemplo, 'automatizando-a-geracao-de-site-map.html', além de um arquivo para cada tag do post.

2. Um post existente é editado, e durante a edição uma tag inédita é adicionada. Isso gera um arquivo do tipo 'tag\_minha\_nova\_tag.html'.

Nestes dois casos eu preciso atualizar o sitemap existente.

Inspecionando o código, existe uma função principal, `do_main()`, que organiza a lógica principal do programa.

Dentro dela temos toda a lógica para lidar com os argumentos passados, gerar os arquivos de css, os componentes como header e footer, etc.

No final da função, temos essa chamada de métodos em sequência:

    do_main(){

      # toda a lógica principal aqui ...

      rebuild_index
      all_posts
      all_tags
      make_rss
      delete_includes
    }

Na lógica do bashblog, alguns métodos criam arquivos temporários, \( geralmente prefixados por ., por ex .header.html, .footer.html \) que são reaproveitados na hora
de gerar os arquivos finais.

Por isso, é importante chamar nossa função após a remoção desses arquivos, que é feito na função `delete_includes`

Adicione a função, agora formatada, dentro do arquivo.


    # Generate site map file
    generate_sitemaps() {

      echo -n "Generating sitemaps "

        {
        while read -r i; do
            echo "$global_url"/$i
        done < <(ls -t *.html)
        } 3>&1 >"$sitemaps_file"

    }

Repare que trocamos a url do site por uma variável global, `global_url` e também o nome do arquivo para `sitemaps_file`. 

Com a função pronta, basta adicionar uma chamada em `do_main()`:

    do_main(){

      # toda a lógica principal aqui ...

      rebuild_index
      all_posts
      all_tags
      make_rss
      delete_includes
      generate_sitemaps # aqui !
    }

Pronto! Extendemos a funcionalidade do blog com sucesso.

## Informações adicionais

Com o sitemap gerado, podemos enviá-lo diretamente para os mecanismos de busca.

O google utiliza o [search console](https://search.google.com/search-console/about), enquanto o bing disponibiliza um [painel webmaster tools](https://www.bing.com/webmasters/).

O processo é parecido, você cria uma conta, verifica o domínio através de um apontamento DNS, ou subindo um arquivo no servidor, e depois adiciona a URL do seu sitemap. Isso agiliza a indexação do site.

O [Yahoo](https://help.yahoo.com/kb/learn-submit-website-yahoo-robotstxt-directive-sln2213.html), assim como outros mecanismos de busca que não oferecem um painel, podem localizar o sitemap mais facilmente
se você adicionar um diretiva no seu arquivo robots.txt .

    # Dentro do arquivo robots.txt

    Sitemap: https://guilhermegarcia.dev/blog/sitemap.txt

Sobre a formatação do arquivo:

A codificação do arquivo deve ser UTF-8.

Os arquivos podem ter qualquer nome, estando de acordo com o padrão RFC-3986:

Use codificação por centro (troque espaços por `%20`, aspas duplas `"` por `%22`, etc) nos seguintes caracteres: `! 	* 	' 	( 	) 	; 	: 	@ 	& 	= 	+ 	$ 	, 	/ 	? 	# 	[ 	]`

O arquivo deve estar sempre no diretório de maior nível o qual o mecanismo de busca deve pesquisar. Por exemplo, se o meu sitemap está dentro do diretório blog/, ele não pode referenciar nenhuma url que esteja na raíz do projeto.

Fontes:

- <https://github.com/cfenollosa/bashblog>
- <https://developers.google.com/search/docs/advanced/sitemaps/overview?hl=pt-br>
- <https://www.sitemaps.org/protocol.html>
- <https://www.bing.com/webmasters/help/sitemaps-3b5cf6ed>
- <https://developers.google.com/search/docs/advanced/sitemaps/build-sitemap?hl=pt-br>
- <https://en.wikipedia.org/wiki/Backlink>
- <https://www.sonymusic.com/sitemap_index.xml>
- <https://tldp.org/LDP/abs/html/io-redirection.html>
- <https://datatracker.ietf.org/doc/html/rfc3986#page-12>

Tags: seo, bash, bashblog
