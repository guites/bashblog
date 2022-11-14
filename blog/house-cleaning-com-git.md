House cleaning com git

Nesse post eu quero compartilhar com vocês algumas operações que eu tive que realizar em uns repositórios antigos, que estavam mal gerenciados e abandonados, pra que eu pudesse voltar a desenvolver neles.

As coisas que eu precisei aprender a fazer foram as seguintes:

- Listar autores de um repositório
- Filtrar os commits do repositório por autor, pra ver o que cada um contribuiu
- Relacionar combinações de autores/email diferentes de um mesmo usuário (usando o arquivo `.mailmap`)
- Em um commit, ver quais arquivos foram alterados
- Ver o que foi alterado em um arquivo específico dentro de um commit
- Verificar o histórico de alteração de um arquivo no repositório (git log arquivo.js)
- Alterar o nome e autor de alguns commits, utilizando o [git-filter-repo](https://github.com/newren/git-filter-repo)

Essas tarefas são ótimas pra se familiarizar \(ou relembrar\) com o andamento de um projeto, ver o ritmo das alterações, quantas mudanças costumam ser embaladas num mesmo commit, e criar um feeling sobre o perfil dos desenvolvedores de um projeto.

Também servem pra organizar e arrumar o histórico e os meta dados dos commits passados, e por isso eu chamo de "House cleaning".

<hr/>

## Listar autores de um repositório

Vamos filtrar o `git log` em combinação com o comando `sort`, listando apenas as resultados não duplicados.

        git log | grep Author: | sort -u

Você vai ter um resultado nesse formato:

        Author: Guilherme Garcia <guilherme67@gmail.com>
        Author: Guilherme <guilherme@empresa.com>
        Author: guites <6653499+guites@users.noreply.github.com>
        Author: guites <guilherme67@gmail.com>
        Author: gustavo <gustavo@empresa.com>
        Author: gustavo2112 <gustavo2112@gmail.com>

No github, nem todo autor representa uma conta separada. Isso porque a pessoa pode utilizar um `git config user.name` e `git config user.email` diferente em diferentes máquinas, conectadas a mesma conta do github.

Como o identificador, no github, é o email, sabemos que o `guites <guilherme67@gmail.com>` e `Guilherme Garcia <guilherme67@gmail.com>` são a mesma conta.

## Filtrar os commits do repositório por autor

Agora que temos os autores, podemos ver as contribuições de cada um usando

        git log --author=nome_ou_email

Aqui você não precisa digitar o nome ou email completo, pois ele funciona como um filtro. Usando o exemplo anterior, as seguintes combinações funcionariam:

        git log --author=gustavo #mostra resultados tanto de gustavo@empresa.com quanto gustavo2112@gmail.com
        git log --author=guites #mostra resultados tanto de guites quanto 6653499+guites@users.noreply.github.com

## Relacionar combinações de autores/email diferentes de um mesmo usuário (usando o arquivo `.mailmap`)

No exemplo acima, um mesmo usuário tinha configurações locais diferentes, o que pode acabar dificultando a filtragem dos seus commits.

Pra resolver isso, podemos criar um arquivo `.mailmap` na raíz do repositório \(no mesmo diretório onde se encontra o diretório .git\).

A estrutura básica do mail map é a seguinte:

        Foo Name <foo@email> commit name <commit@email>
        \--------+---------/ \----------+-------------/
                 |                      |
              substituir por          buscar por

No nosso exemplo, digamos que os autores

- Author: Guilherme Garcia <guilherme67@gmail.com>
- Author: Guilherme <guilherme@empresa.com>
- Author: guites <6653499+guites@users.noreply.github.com>
- Author: guites <guilherme67@gmail.com>

Sejam a mesma pessoa, configurada em diferentes computadores com nomes e emails diferentes (por ex. usando tanto o email pessoal quanto o empresarial, e em uma outra máquina não configurou o email, daí ficou com um email auto associado do github).

Podemos preencher um `.mailmap` com as seguintes entradas

       Guilherme Garcia <guilherme67@gmail.com> <guilherme67@gmail.com>
       Guilherme Garcia <guilherme67@gmail.com> <guilherme@empresa.com>
       Guilherme Garcia <guilherme67@gmail.com> <6653499+guites@users.noreply.github.com>

Agora, se rodarmos o filtro por autor uma segunda vez

        git log | grep Author: | sort -u

Vamos ter a seguinte saída:

        Author: Guilherme Garcia <guilherme67@gmail.com>
        Author: gustavo <gustavo@empresa.com>
        Author: gustavo2112 <gustavo2112@gmail.com>

## Em um commit, ver quais arquivos foram alterados

Se você quiser saber o que foi feito num commit, pode utilizar o `git show` no formato abaixo

        git show <commit hash>

Ele vai mostrar na tela um diff de tudo o que foi alterado naquele commit em relação ao anterior. Para commits grandes, pode ser uma boa ideia verificar quais arquivos foram alterados, ao invés de ver quais alterações foram feitas nos arquivos. Pra isso, podemos alterar o comando

        git show <commit hash> --name-only

E vamos receber uma listagem com o nome dos arquivos alterados.

## Ver o que foi alterado em um arquivo específico dentro do commit

Depois de decidir qual arquivo você quer analisar dentro do commit selecionado, você pode ver apenas as alterações feitas nele, passando-o como argumento no `git show`

        git show <commit hash> -- caminho/ate/o/arquivo.extensao

## Verificar o histórico de alteração de um arquivo no repositório

Se você precisar do caminho contrário, ou seja, quiser saber qual a sequência de alterações pelas quais um arquivo específico passou, você pode usá-lo como argumento diretamente no `git log`

        git log caminho/ate/o/arquivo.extensao

E então pode ir usando `git show <commit hash> -- caminho/ate/o/arquivo.extensao` em cada commit da lista, pra ir acompanhando as alterações realizadas.

## Alterar o nome e autor de alguns commits, utilizando o git-filter-repo

Essa é uma forma mais radical de corrigir autores e emails trocados, que envolve reescrever o histórico do repositório.

Se você estiver trabalhando em conjunto nesse repositório, ou se as alterações que você fizer forem envolver commits que já foram enviados pro remote \(com o `git push` por exemplo\), essa opção pode dificultar a vida dos outros colaboradores.

O [git-filter-repo](https://github.com/newren/git-filter-repo) é um conjunto de scripts escrito em python que permite alterar e reescrever meta dados de um repositório.

Você pode seguir as instruções de instalação no repositório acima, e utilizar o seguinte filtro pra, por exemplo, alterar todos os commits nos quais o autor é `guilherme@empresa.com` para `guilherme67@gmail.com`.

Repare que o git possui dois campos de autoria: o autor do commit \(commit.author\), e a pessoa que registrou o commit \(commit.committer\).

No caso abaixo, estou alterando ambos os campos.

A flag `--force` pode ser removida e, nesse caso, o `git filter-repo` só permite rodar o filtro em um repositório que não possua alterações locais \(ou seja, recém clonado\).

        git filter-repo --force --commit-callback '
            if commit.author_email == b"guilherme@empresa.com":
                commit.author_email = b"guilherme67@gmail.com"
                commit.author_name = b"guites"
                commit.committer_email = b"guilherme67@gmail.com"
                commit.committer_name = b"guites"
        '

## Considerações finais

Espero que esses comandos te ajudem a navegar em novos repositórios, tentando se familiarizar com o código existente, ou a ajustar repositórios antigos bagunçados.

Abraço!

## Fontes

- <https://lukasmestan.com/using-mailmap-in-git-repository/>
- <https://stackoverflow.com/a/750182/14427854>

Tags: git
