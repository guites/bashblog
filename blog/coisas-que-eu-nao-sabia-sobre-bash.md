Coisas que eu não sabia sobre bash

<abbr title="muito grande; nem li">mg;nl</abbr>: Compilei o aqui o resultado de uma noite analisando o código fonte do [bashblog](https://github.com/cfenollosa/bashblog).

O bash tem muitas particularidades, principalmente pra quem tem um background em linguagens web. Neste post eu falo sobre:

1. Uso de exit status como condicional
2. Redirecionamento de streams
3. Parenteses duplos para comparação numérica
4. Expansão de parametros e criação de substrings

ps. com o quarto item eu consegui gerar um [pull request](https://github.com/cfenollosa/bashblog/pull/168)

<hr/>

Segue um resumo do que eu estudei baseado nas referências, que estão no fim da página.

## Uso de exit status como condicional

O bash possui certos parametros, chamados [parametros especiais](http://www.gnu.org/software/bash/manual/bash.html#Special-Parameters) que podem ser acessados dentro de um script.

O parametro `$?` retorna o exit status do último programa executado. Um programa executado com sucesso retorna 0, enquanto um programa que tenha gerado erro retorna 1.

por ex, digamos que eu esteja recebendo uma string passada em uma variável, e precise me certificar de que ela possui uma substring "bar".

    STR="barbada"

    grep bar <<< "$STR"

    if [ $? == 0 ]; then
      echo "bar" está dentro de "$STR"
    fi

    STR2="batida"

    grep bar <<< "$STR"

    if [ $? == 0 ]; then
      echo "bar" está dentro de "$STR2"
    else
      echo "bar" não está dentro de "$STR2"
    fi

output:

    barbada

    "bar" está dentro de "barbada"

    "bar" não está dentro de "batida"

repare que o valor da primeira variável foi printado no console. Isso é devido ao comportamento padrão do comando `grep`. O que nos leva ao próximo tópico.

## Redirecionamento de streams

Em sistemas UNIX, existem três tipos de _stream_, também chamados de _files_, pois podem ser tratados como arquivos [\(assim como os diretórios\)](https://en.wikipedia.org/wiki/Everything_is_a_file):

o _stdin_ se trata dos valores de entrada emitidos pelo teclado, o _stdout_ são os valores emitidos no terminal, e o _stderr_ são os erros. Também são chamados, respectivamente, de 0, 1 e 2.

No exemplo acima, podemos utilizar o operador `&>` para redirecionar ambos _stdout_ e _stderr_, e realizar nosso teste de forma silenciosa.

    STR="barbada"

    grep bar <<< "$STR" &> /dev/null

    if [ $? == 0 ]; then
      echo "bar" está dentro de "$STR"
    fi

output:

    "bar" está dentro de barbada

Este padrão é comum em scripts que geram um tratamento de erro customizado para o usuário.

## Parenteses duplos para comparação numérica

O uso de parenteses duplos faz com que o bash crie um contexto onde os valores são interpretados como números e as palavras, como variáveis.

Podemos nos utilizar disso para escrever expressões mais enxutas:

    STR="barbada"

    grep bar <<< "$STR" &> /dev/null

    (( $? == 0 )) && echo "bar" está contida em "$STR"

output:

    bar está contida em barbada

Para lógicas com `else`, podemos utilizar o operador condiconal OR `||`:

    (( $? == 0 )) && echo "foo" está contida em "$STR" || echo foo não está presente em "$STR"

Em situações mais complexas, o ideal é utilizar a sintaxe regular.

## Expansão de parametros e criação de substrings

O tópico de _expansão de parametros_ no bash é bem amplo.

A expansão pode ser utilizada para:

**`${var}`** Substituir variáveis dentro de uma string:

    var=" uma string"

    echo esta variável vai ser expandida em${var}

output:

    esta variável vai ser expandida em uma string


**`${var:offset}`** Gerar uma string, partindo de um offset:

    var="Uma frase comprida"
    echo ${var:4}

output:

    frase comprida

**`${var:offset:length}`** Gerar um string, partindo de um offset e indicando um comprimento:

    var="Uma frase comprida"
    echo ${var:4:5}

output:

    frase

**`${#var}`** Calcular a quantidade de caracteres em uma string:

    var="Paralelepipedo"
    echo ${#var}

output:

    14


**`${var:=valor_substituicao}`** Atribuir valor a uma variável de forma condicional.

O operador `:=` atribui o valor da direita para uma variável à esquerda, caso ela seja nula ou indefinida.

    # neste exemplo, var é indefinida
    valor_substituicao="caso essa seja nula ou indefinida."
    echo o valor da direita será passado para a variável da esquerda, ${var:=$valor_substituicao}
    echo ${var}

output:

    o valor da direita será passado para a variável da esquerda, caso essa seja nula ou indefinida.
    caso essa seja nula ou indefinida.

Caso a variável não for indefinida, o valor não será substituido.

    # neste exemplo, var é definido
    var="variável definida"
    valor_substituicao="meu valor não será utilizado"
    echo Não ocorrerá substituição pois var é uma ${var}
    echo ${var}

output:
    Não ocorrerá substituição pois var é uma variável definida
    variavel definida

**`{var:-$valor_substituicao}`** Expandir um valor no lugar de uma variável `var` nula ou indifinada:

O operador não atribui valores. Caso o valor de `var` seja nulo ou indefinido, o valor de `$valor_substituicao` será expandido, mas não será passado para `var`.

    var_1="var_1 é uma variável definida"
    var_2="Eu não vou ser chamada, pois var_1 é definida"
    echo O valor da direita será passado para a variável da esquerda apenas se essa for indefinida.
    echo Isso não vai acontecer pois ${var_1:-$var_2}
    echo ${var_1}

    # neste exemplo, var_3 é indefinida
    var_2="valor expandido"
    echo Neste caso, var_2 vai ter seu {var_3:-$var_2}, mas não substituido.
    echo ${var_3}

output:

    O valor da direita será passado para a variável da esquerda apenas se essa for indefinida.
    Isso não vai acontecer pois var_1 é uma variável definida
    var_1 é uma variável definida
    Neste caso, var_2 vai ter seu valor expandido, mas não substituido.

**`${var#padrao}`** Deletar valores a partir do início de uma string `var`, removendo o menor padrão encontrado:

O operador `${var#padrao}` busca pelo `padrao` dentro do valor de `var`, partindo do primeiro caractere. Caso encontre, ele realiza uma expansão, deletando o padrão encontrado.

    var="Uma pequena oração."
    padrao="Uma"
    padrao_2="pequena"

    resultado=${var#${padrao}}
    echo $resultado

    resultado_2="${var#padrao_2}" # substituição não ocorre, pois o padrão deve coincidir com o início
    echo $resultado_2

output:

    pequena oração.
    Uma pequena oração.

Neste caso, podemos utilizar como `padrao` algumas [expressões especiais](https://www.gnu.org/software/bash/manual/bash.html#Pattern-Matching) \(não chegam a ser expressões regulares completas\).

**`${var##padrao}`** Deletar valores a partir do início de uma string `var`, removendo o maior padrão encontrado:

Esta variação, com dois `##`, expande e remove a maior combinação possível com o padrão utilizado.

Para que elas funcionem corretamente, é preciso ativar o `extended globbing`.

    shopt -s extglob # ativa extensão para uso de globbing avançado

    zero_a_esquerda="001"
    resultado=${zero_a_esquerda##+(0)} # vai remover o máximo de zeros possível
    resultado_2=${zero_a_esquerda#+(0)} # vai remover o menor padrão possível (apenas um zero)

    frase_inicial="Eu virei uma frase, com pontuação."

    # vai substituir o máximo de letras e espaços, trava na primeira pontuação
    frase_final=${frase_inicial##+([[:alnum:]]|[[:blank:]])}

    # o uso de apenas um símbolo # indica para parar no menor padrão encontrado possível
    frase_final_2=${frase_inicial#+([[:alnum:]]|[[:blank:]])}
    shopt -u extglob # desativa extensão

    echo $resultado
    echo $resultado_2
    echo $frase_final
    echo $frase_final_2

output:

    1

    01

    , com pontuação.

    u virei uma frase, com pontuação.


Abraço!

referências:

1. <https://tldp.org/LDP/Bash-Beginners-Guide/html/sect_07_01.html>
2. <https://tldp.org/LDP/abs/html/io-redirection.html>
3. <https://www.quora.com/In-shell-scripting-why-do-we-need-double-parentheses-when-we-need-to-enclose-some-expression>
4. <http://www.gnu.org/software/bash/manual/bash.html#Special-Parameters>
5. <https://en.wikipedia.org/wiki/Everything_is_a_file>
6. <https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion>
7. <https://tldp.org/LDP/abs/html/parameter-substitution.html>

Tags: bash
