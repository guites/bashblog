Remova os gifs do medium

A plataforma medium tem muito conteúdo bom, mas infelizmente muitas postagens ótimas são permeadas de gifs super coloridos, que eu considero distrativos.

Quando eu tô procurando direcionamento sobre um tópico, e finalmente acho um post sobre o assunto, eu não gosto muito da mudança de tom que os gifs trazem, inseridos a cada parágrafo, tipo um comic relief desnecessário \([ahem](https://www.google.com/search?channel=fs&client=ubuntu&q=jar+jar+binks)\).

Daí esse snippet esconde todos os gifs e põe um botãozinho pra mostrar/esconder, aí você pode dar uma olhada e desligar a coisa.

![Aba dos favoritos no firefox](./imgs/remova-os-gifs-do-medium/adding-bookmarklet.png "Aba dos favoritos no firefox")

Você pode ir "Add Bookmarks" e adicionar o seguinte, com um nome tipo "hide-medium-gifs":

        javascript:(function() {const doc_imgs=document.querySelectorAll('img');function toggleImgVisibility(e){let prevSibling=e.target.previousElementSibling;while(prevSibling){if(prevSibling.classList.contains('hidden-g0f')){if(prevSibling.style.visibility=='hidden'){prevSibling.style.visibility='visible';}else{prevSibling.style.visibility='hidden';}break;}prevSibling=prevSibling.previousElementSibling;}}function createToggleBtn(){const button=document.createElement("BUTTON");button.innerText="Toggle gif visibility";button.addEventListener("click",toggleImgVisibility);return button;}doc_imgs.forEach((img) => {if (!img.src.includes('.gif')) return;img.classList.add("hidden-g0f");img.style.visibility = "hidden";img.parentElement.appendChild(createToggleBtn());});}())

Daí, em qualquer página do medium (por [exemplo...](https://medium.com/perry-street-software-engineering/clean-api-architecture-2b57074084d5)), basta clicar no favorito criado, e pronto: gifs pausáveis.

<video controls style="width:100%;height:100%;"src="./imgs/remova-os-gifs-do-medium/hide-g0fs.mp4">

Abaixo eu explico melhor a função.

<hr/>

O código é o seguinte:

        const doc_imgs = document.querySelectorAll('img');

        function toggleImgVisibility(e) {
          let prevSibling = e.target.previousElementSibling;
          while (prevSibling) {
            if (prevSibling.classList.contains('hidden-g0f')) {
              if (prevSibling.style.visibility == 'hidden') {
                prevSibling.style.visibility = 'visible';
              } else {
                prevSibling.style.visibility = 'hidden';
              }
              break;
            }
            prevSibling = prevSibling.previousElementSibling;
          }
        }

        function createToggleBtn() {
          const button = document.createElement("BUTTON");
          button.innerText = "Toggle gif visibility";
          button.addEventListener("click", toggleImgVisibility);
          return button;
        }

        doc_imgs.forEach((img) => {
          if (!img.src.includes('.gif')) return;
          img.classList.add("hidden-g0f");
          img.style.visibility = "hidden";
          img.parentElement.appendChild(createToggleBtn());
        });

Eu pego todas as imagens da página na variável `doc_imgs`, e itero sobre elas, verificando se o `src` da imagem tem `.gif` nele.

Se tiver, eu adiciono uma classe (hidden-g0f :P), e altero a propriedade [visibility](https://developer.mozilla.org/en-US/docs/Web/CSS/visibility) pra 'hidden'. Isso esconde o gif, mas ele continua ocupando espaço na página.

Depois, pego o elemento pai e adiciono um botão, pela função `createToggleBtn`.

Esse botão tem um `eventListener`, que é chamado no clique, a função `toggleImgVisibility`.

Quando ela é chamada, ela itera pelos elementos irmãos do botão, e busca aquele que tiver a classe adicionada no passo anterior.

Se tiver a classe, eu alterno a propriedade `visibility` entre `hidden` e `visible`.

Daí fica fácil esconder/mostrar os gifs um por um.

Fica a dica, medium ヽ(￣～￣　)ノ.

Tags: bookmarklet
