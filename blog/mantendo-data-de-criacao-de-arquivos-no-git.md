Mantendo data de criação de arquivos no git

<abbr title="muito grande; nem li">mg;nl</abbr>: Vamos desenvolver dois scripts, um pra adicionar a data da ultima alteração de um arquivo em forma de uma timestamp, evitando que ela seja perdida quando o arquivo é copiado ou movido, e outro para ordená-los através da data presente na timestamp. 

Se você está versionando seus arquivos com git, usando um remote como o [gitlab](https://about.gitlab.com/), [github](https://github.com/) ou [heroku](https://www.heroku.com/home), 
já deve ter reparado que, cada vez que você clona o repositório, a data de criação e modificação dos arquivos passa pro momento que você clonou.

Vou apresentar nesse post os métodos que eu utilizo para persistir essa informação.

<hr/>

Se ao dar push nos seus arquivos, eles estavam assim:

    -rw-r--r--  1 guilhermegarcia  wheel   919 Feb  6 00:06 order_by_date.txt
    -rw-r--r--  1 guilhermegarcia  wheel  3316 Feb  6 00:06 looper.sh
    -rw-r--r--  1 guilhermegarcia  wheel    85 Feb  5 22:13 fev.html
    -rw-r--r--  1 guilhermegarcia  wheel   398 Feb  5 20:55 array.sh
    -rw-r--r--  1 guilhermegarcia  wheel   259 Feb  5 20:16 README.md
    -rw-r--r--  1 guilhermegarcia  wheel    79 Jan 27 00:45 20220127.html
    -rw-r--r--  1 guilhermegarcia  wheel    79 Jan 26 00:45 20220126.html
    -rw-r--r--  1 guilhermegarcia  wheel    80 Jan 25 00:45 20220125.html

Após clonar o repositório, eles provavelmente ficarão assim:

    -rw-r--r--  1 guilhermegarcia  wheel   919 Feb  6 00:31 order_by_date.txt
    -rw-r--r--  1 guilhermegarcia  wheel  3316 Feb  6 00:31 looper.sh
    -rw-r--r--  1 guilhermegarcia  wheel    85 Feb  6 00:31 fev.html
    -rw-r--r--  1 guilhermegarcia  wheel   398 Feb  6 00:31 array.sh
    -rw-r--r--  1 guilhermegarcia  wheel   259 Feb  6 00:31 README.md
    -rw-r--r--  1 guilhermegarcia  wheel    79 Feb  6 00:31 20220127.html
    -rw-r--r--  1 guilhermegarcia  wheel    79 Feb  6 00:31 20220126.html
    -rw-r--r--  1 guilhermegarcia  wheel    80 Feb  6 00:31 20220125.html

Caso algum script seu dependa da data de criação ou alteração dos arquivos, eles não vão funcionar como o esperado.

## Adicionando uma timestamp nos arquivos

Antes de perder essa informação (antes de subir no repositório), você pode utilizar um script simples pra adicionar uma marcação no final de cada arquivo.

    # arquivo set_timestamp.sh

    while read -r i; do
      [[ $i == $0 ]] && continue
      created=$(date -r $i +"%Y%m%d%H%M.%S")
      echo "timestamp: $created" >> $i
    done < <(ls -t)

Rodamos ele com `bash set_timestamp.sh`.

A verificação `[[ $i == $0 ]] && continue` impede que o script escreva em si mesmo. No bash, todo script tem um argumento, `$0`, que é seu próprio nome.

A timestamp salva é no formato `YYYYMMDDHHMM.S`, por exemplo: 202202060046.13 para a data 06/02/2022 00:46:13. O motivo vai ficar mais claro quando formos ajustar a data dos arquivos.

## Voltando a data original de modificação dos arquivos

Repare que executar o script alterou a data de todos os arquivos no diretório. Por sorte, com as timestamps salvas, podemos recuperar essa informação.

Vamos criar um novo script pra realizar o ajuste.

    # arquivo fix_timestamp.sh

    while read -r i; do
      [[ $i == $0 ]] && continue
      timestamp=$(sed -rn 's/^timestamp: (.*)$/\1/p' "$i")
      touch -t $timestamp $i
    done < <(ls -t)

Rodando com `bash fix_timestamp.sh` a data de modificação vai ser recuperada para a original.

O comando `touch -t` altera a data de modificação de um arquivo sem alterar seu conteúdo, e espera uma data no formato `YYYYMMDDHHMM.S`, o mesmo que utilizamos para gerar a timestamp acima.

Porém ainda temos alguns problemas, por exemplo, se seus arquivos não forem de texto simples, a marcação no final pode gerar uma série de erros. 

## Timestamp baseada na extensão do arquivo

Podemos filtrar pela extensão e adicionar a marcação como um comentário, em função do tipo de arquivo.

    # arquivo set_timestamp.sh

    while read -r i; do
      [[ $i == $0 ]] && continue
      created=$(date -r $i +"%Y%m%d%H%M.%S")
        extension=${i##*.}                                    ## ADICIONAR
        case $extension in                                    ## ADICIONAR
          html) timestamp="<!-- timestamp: $created -->" ;;   ## ADICIONAR
          sh) timestamp="# timestamp: $created" ;;            ## ADICIONAR
          js) timestamp="// timestamp: $created" ;;           ## ADICIONAR
          *) timestamp="timestamp: $created" ;;               ## ADICIONAR
        esac                                                  ## ADICIONAR
      echo "timestamp: $created" >> $i                        ## ALTERAR
    done < <(ls -t)

Definimos a extensão de cada arquivo usando uma [substituição de parametros](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html), deletando todos os caracteres até chegar no ponto \(.\).

Depois, com um switch case, podemos adicionar a marcação de comentário necessária pra cada tipo de arquivo.

Agora, podemos alterar o script que ajusta a data de alteração de cada arquivo, mas uma outra possibilidade é listar os arquivos pela própria timestamp.

## Unindo os dois processos em um único arquivo

Usando uma expressão regular mais completa, podemos em um unico script corrigir a data de alteração do arquivo, caso a timestamp exista, e, caso não exista, adicionar uma com a data da sua última modificação.

    # arquivo timestamp.sh

    while read -r i; do
      [[ $i == $0 ]] && continue
      # pula p/ próxima iteração caso for um diretório
      [[ -d $i ]] && continue
      timestamp=$(sed -rn 's/.*timestamp: ([0-9]{12}\.[0-9]{2}).*/\1/p' "$i")
      # se a timestamp já existe, ajustamos o arquivo e pulamos p/ a próxima iteração
      [[ -n $timestamp ]] && touch -t $timestamp "$i" && continue

      # caso não exista, pegamos a data de modificação
      created=$(date -r $i +"%Y%m%d%H%M.%S")

      # formatamos a timestamp de acordo com a extensão
      extension=${i##*.}
      case $extension in
        html) timestamp="<!-- timestamp: $created -->" ;;
        sh) timestamp="# timestamp: $created" ;;
        js) timestamp="^// timestamp: $created" ;;
        *) timestamp="timestamp: $created" ;;
      esac

      # adicionamos ao arquivo
      echo $timestamp >> $i

    done < <(ls -t)

Começamos buscando uma timestamp existente. A opção `-r` no sed nos permite utilizar expressões regulares.

O formato que buscamos é uma sequência de quaisquer caracteres \(`.*`\) seguidos pela palavra `timestamp: `. Depois, usando os parenteses para definir um grupo de captura, queremos 12 digitos seguidos de um ponto, e mais dois digitos.

Podemos referenciar o grupo de captura com `\1`. A opção `-n` indica que não queremos retornar o conteúdo \(normalmente o sed retorna todo o texto do arquivo, com a substituição realizada\), e por fim a flag `p` nos permite retornar apenas a linha na qual ocorreu a substituição.

## Listando arquivos pela timestamp ao invés da data de modificação do arquivo

As alterações dos scripts acima partem do princípio de que você vai utilizar o comando `ls -t` para ordenar seus arquivos, alimentar scripts, etc.

Caso você queria listá-los diretamente pela timestamp que adicionamos, independente do tempo de criação ou modificação, podemos usar uma combinação do comando `grep` com o comando `sort`.


    list_files_by_timestamp() {
      grep -s "timestamp: " * \ # (1)
      | sed -nr 's/(:).*timestamp: ([0-9]{12}\.[0-9]{2}).*/\1\2/p' \ # (2)
      | sort --field-separator=: --key=2 --stable --reverse \ # (3)
      | sed -nr 's/^(.*):.*$/\1/p' # (4)
    }

Em \(1\) buscamos por todos os arquivos que possuem o texto "timestamp: " no diretório atual. A opção `-s` suprime erros, por exemplo se você tiver algum diretório, o grep retorna "grep: scripts: Is a directory".

O output do comando grep é no formato:

    20220125.html:<!-- timestamp: 202202060046.13 -->
    20220126.html:<!-- timestamp: 202202020046.13 -->
    20220127.html:<!-- timestamp: 202202060046.13 -->
    README.md:timestamp: 202202060117.25
    fev.html:<!-- timestamp: 202202060117.37 -->
    order_by_date.txt:timestamp: 202202060117.21

Em \(2\) adaptamos a expressão regular que usamos na seção anterior, dessa vez partindo dos dois pontos, pois precisamos manter o nome dos arquivos.

Em \(3\) usamos o comando sort, passando a opção `--field-separator` para identificar diferentes campos através dos dois pontos `:`. Com a opção `--key=2` dizemos que o campo à direita dos dois pontos deve ser usado para ordenar o resultado.

A opção `--reverse` coloca os resultados com o mais recente por primeiro.

Em \(4\), usamos novamente uma expressão regular para remover as informações de data, visto que nosso interesse é no nome dos arquivos.

### Considerações finais

Vimos como manter informações de criação e modificação de arquivos ao enviá-los para e cloná-los de um repositório remoto. 

Essas tecnicas também são válidas quando precisamos copiar arquivos, ou enviá-los via ssh, etc.

Abraço!

Fontes:

- <http://www.linfo.org/touch.html>
- <https://www.grymoire.com/Unix/Sed.html#uh-15b>
- <https://askubuntu.com/a/777456>
- <https://man7.org/linux/man-pages/man1/sort.1.html>
- <https://stackoverflow.com/a/29508361>
- <https://tldp.org/LDP/abs/html/assortedtips.html>

Tags: git, bash
