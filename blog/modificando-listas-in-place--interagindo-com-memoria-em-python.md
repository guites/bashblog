Modificando listas in place / interagindo com memória em python

<p><abbr title="muito grande;nem li">mg;nl</abbr>: entrei num buraco de minhoca tentando enviar uma resposta envolvendo modificar listas in place no LeetCode. Aqui relato a diferença entre um assignment usando <code>=</code> e um <code>slice</code> assignment quando se trata de sobrescrever listas.</p>

<p>Post originalmente escrito pro fórum do <a href="https://matehackers.org/">matehackers</a> em 06/09/2023.</p>

<hr>

<h2>Modificar listas in place</h2>

<p>Quando o problema pede pra ti alterar o valor de uma variável ao invés de retornar o novo valor em uma função, e esse valor for uma lista, o LeetCode não aceita que tu simplesmente atribua esse novo valor.</p>

<p>Por exemplo:</p>

<blockquote>
  <p>Crie uma função que recebe duas listas, e sobrescreva o valor da primeira lista com o valor da segunda. A função não deve retornar o novo valor da primeira lista, e sim alterá-la <em>in_place</em>.</p>
</blockquote>

<p>Se tu escrever algo assim:</p>

<pre><code># modify_list_in_place.py

def modify_list_in_place(input_list: list, new_list: list) -&gt; None:
  input_list = new_list
  print(input_list)

modify_list_in_place([1, 2, 3], [4, 5, 6])
</code></pre>

<p>e rodar código na tua máquina, vai ver que o valor é o esperado:</p>

<pre><code>$ python3 modify_list_in_place.py
[4, 5, 6]
</code></pre>

<p>Mas essa não é considerada uma resposta correta.</p>

<p>Acontece que quando tu usa apenas o sinal de igual como <code>assignment operator</code>, você está dizendo que a variável <code>input_list</code> agora aponta para (o local na memória onde fica) o valor de <code>new_list</code> naquele momento.</p>

<p>Essa operação não muda o valor anterior para o qual <code>new_list</code> estava apontando! Ainda existe o valor <code>[1, 2, 3]</code> salvo na memória.</p>

<p>Pra que o valor antigo seja de fato alterado, é preciso usar um operador de <strong>slice assignment</strong>.</p>

<p>Ele tem a seguinte sintaxe:</p>

<pre><code>my_list = [1, 2, 3]
my_list[:] = [4, 5, 6]
print(my_list)

# 4, 5, 6
</code></pre>

<p>O valor de cada item da lista <code>my_list</code> foi realmente alterado. Não existe mais a lista <code>[1, 2, 3]</code> salva em nenhum lugar.</p>

<p>Uma forma de visualizar esse comportamento (alterar pra onde a variável aponta vs. alterar o valor salvo em memória) é com o seguinte bloco de código:</p>

<pre><code>original = [1, 2, 3]
other = original
original = [4, 5, 6]
print(other)

# 1, 2, 3
</code></pre>

<p>O valor de <code>other</code> não tem motivo pra mudar. Afinal, você definiu que ele apontava pro valor que <code>original</code> tinha no momento em que você fez a atribuição (na linha <code>other = original</code>), e o valor para o qual <code>original</code> apontava era [1, 2, 3].</p>

<p>Atribuir um novo valor para <code>original</code> (na linha <code>original = [4, 5, 6]</code>) não afeta a existência do endereço em memória com <code>[1, 2, 3]</code>. Ele ainda existe.</p>

<p>Veja a diferença pra esse caso:</p>

<pre><code>original = [1, 2, 3]
other = original
original[:] = [4, 5, 6]
print(other)

# 4, 5, 6
</code></pre>

<p>Fez sentido? Ao usar o <strong>slice assignment</strong>, os valores de <code>original</code> são de fato alterados. E como o <code>other</code> aponta pra esse mesmo endereço na memória, o seu valor muda também.</p>

<p>A nossa função original ficaria com esse formato, pra que realmente altere os valores da lista original:</p>

<pre><code># modify_list_in_place_with_slice.py

def modify_list_in_place_with_slice(input_list: list, new_list: list) -&gt; None:
  input_list[:] = new_list
  print(input_list)

modify_list_in_place([1, 2, 3], [4, 5, 6])
</code></pre>

<p>E, nesse caso, a saída da função não muda (mas o valor da lista, sim).</p>

Fontes:

- <a href="https://leetcode.com/problems/merge-sorted-array/?envType=study-plan-v2&amp;envId=top-interview-150">https://leetcode.com/problems/merge-sorted-array/?envType=study-plan-v2&amp;envId=top-interview-150</a>
- <a href="https://stackoverflow.com/a/32448477">https://stackoverflow.com/a/32448477</a>
- <a href="https://stackoverflow.com/a/6167247">https://stackoverflow.com/a/6167247</a>

Tags: python, algorithms
