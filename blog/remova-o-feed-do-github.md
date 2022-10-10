Remova o feed do github

<abbr title="muito grande;nem li">mg;nl</abbr>: vamos criar um bookmarklet pra remover o feed da página inicial do github.

Se você é como eu, provavelmente já entrou no seguinte loop:

1. Você fica com dúvida sobre um projeto hospedado no github
2. Você acessa <https://github.com>.
3. Você fica interessado no que as pessoas andaram fazendo, e acaba clicando em alguma coisa.
4. Você mata a curiosidade e sai do github.
5. ????
6. Você lembra da dúvida sobre o projeto
7. repeat

Vamos remover a distração da homepage do github usando um pouco de javascript.

<hr/>

## Bookmarklets

Bookmarklets são como favoritos, mas ao invés de direcionar para uma nova página, rodam um _snippet_ de javascript na página atual.

Eles precisam seguir as seguintes regras:

1. Ter `javascript:` no início
2. Estar encapsulado numa [IIFE](https://en.wikipedia.org/wiki/Immediately_invoked_function_expression)

Ou seja, deve seguir o seguinte formato:

        javascript:(function(){
          alert("Bookmarklet!");
        })();

## Feed: página inicial do github

Se você está logado, a página inicial do github tem esse formato:

![Captura de tela da home do Github](./imgs/remova-o-feed-do-github/github-home.jpg "Captura de tela da home do Github")

Inspecionando o HTML da página, vemos que a seção principal é dividida em dois containers.

O primeiro contém o menu da esquerda \(menu com os repositórios\):

![Menu com os repositórios na home do Github](./imgs/remova-o-feed-do-github/menu-repositorios-github-home.jpg "Menu com os repositórios na home do Github")

O segundo é dividido em dois. Primeiro, o feed central:

![Feed central da home do Github](./imgs/remova-o-feed-do-github/menu-central-feed-github-home.jpg "Feed central da home do Github")

Depois, o feed com as últimas atualizações e recomendações de repositório:

![Feed com atualizações da home do Github](./imgs/remova-o-feed-do-github/feed-atualizacoes-github-home.jpg "Feed com atualização da home do Github")

## Removendo os feeds

Sabendo o markup, podemos utilizar javascript para removê-los.

O feed principal pode ser selecionado pelo seu id:

        document.querySelector("#dashboard-feed-frame")

O feed de atualizações pode ser selecionado pelo seu `aria-role`:

        document.querySelector("aside[aria-label=Explore]")

Podemos utilizar uma iteração para selecionar e removê-los:

        [
            "#dashboard-feed-frame",
            "aside[aria-label=Explore]"
        ].forEach((selector) => document.querySelector(selector).remove())

Você pode ver o resultado na imagem abaixo:

![Home do github sem os feeds](./imgs/remova-o-feed-do-github/empty-github-home.png "Home do github sem os feeds")

## Ocupando a tela com informações (úteis)

A tela fica meio vazia, e o espaço mau aproveitado, sem os feeds.

Vamos expandir o conteúdo útil. O que sobrou na tela principal pode ser acessado pela classe `.application-main`.

        const appMain = document.querySelector(".application-main");

A estrutura dentro desse elemento é no seguinte formato:

    <div class="application-main " ...>
        <div class="d-md-flex color-bg-inset" style="min-height: 100vh;">
            <aside data-turbo="false" class="team-left-column ..." aria-label="Account">
                <!-- LISTA DE REPOSITÓRIOS -->
            </aside>
            <div class="flex-auto col-md-8 col-lg-8 px-3 px-lg-5">
                <!-- RODAPÉ -->
            </div>
        </div>
    </div>

A `<div>` logo abaixo do nosso `appMain` serve como um _wrapper_ flexbox. Vamos adicionar um `flex-direction: column` e centralizar o conteúdo:

    appMain.querySelector("div").style.flexDirection = "column";
    appMain.querySelector("div").style.alignItems = "center";

Vamos também expandir a lista de repositórios para ocupar toda a largura disponível:

        ["aside", "aside div"].forEach((sel) => appMain.querySelector(sel).style.minWidth = "100vw");

Sem distrações!

![Home do github sem distrações](./imgs/remova-o-feed-do-github/sem-distracoes.png "Home do github sem distrações")


## O bookmarklet final

O seguinte código vai ajustar sua home em um clique:


        [
            "#dashboard-feed-frame",
            "aside[aria-label=Explore]"
        ].forEach((selector) => document.querySelector(selector).remove());

        const appMain = document.querySelector(".application-main");

        appMain.querySelector("div").style.flexDirection = "column";
        appMain.querySelector("div").style.alignItems = "center";

        ["aside", "aside div"].forEach((sel) => appMain.querySelector(sel).style.minWidth = "100vw");

E, formatado para bookmarklet, fica da seguinte forma:

        javascript:(function(){
            [
                "#dashboard-feed-frame",
                "aside[aria-label=Explore]"
            ].forEach((selector) => document.querySelector(selector).remove());

            const appMain = document.querySelector(".application-main");

            appMain.querySelector("div").style.flexDirection = "column";
            appMain.querySelector("div").style.alignItems = "center";

            ["aside", "aside div"].forEach((sel) => appMain.querySelector(sel).style.minWidth = "100vw");
        })();

## Referências

1. <https://gist.github.com/caseywatts/c0cec1f89ccdb8b469b1>

Tags: bookmarklet, javascript
