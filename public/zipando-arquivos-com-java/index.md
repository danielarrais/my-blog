---
title: 'Zipando arquivos com java'
date: '2021-04-13'
spoiler: "Aprenda a zippar multiplos arquivos e pastas utilizando java."
---

# Zipar arquivos/pastas usando Java

Zipar arquivos usando o Java é uma tarefas simples e não precisamos de bibliotecas de terceiros para isso. Para vermos
esta simplicidade na prática começarei o artigo implementando a zipagem de um arquivo, que também será utilizado na
zipagem de múltiplos arquivos e de pastas.

## Zipar um arquivo

O código necessário para zipar um arquivo é este:

<script src="https://gist.github.com/danielarrais/a2f9d955519d8d18de30476f6482a221.js?file=block_one.java"></script>

No código você pode notar que utilizei as classes `java.io.FileOutputStream` , `java.io.FileInputStream`,
`java.util.zip.ZipOutputStream` e a `java.util.zip.ZipEntry`. Elas possuem alguns papéis, a `FileOutputStream` é
necessária para gravar bytes no disco e a `FileInputStream` para ler; a `ZipOutputStream` é uma `FileOutputStream`
especializada em criar e gravar arquivos zip's e dispõe de métodos que auxiliam tal tarefa; a `ZipEntry` é utilizada
para adicionar arquivos dentro do ZIP, nela podemos adicionar informações sobre o arquivo, como o nome dele.

Como você deve ter notado (espero rsrsrsrs), o código é simples e consegue zipar um único arquivo. Nele não há segredos:

1. Primeiro criamos uma instância de File passando a URL do arquivo a ser compactado;
2. Instanciamos nossos Streams, a `FileOutputStream` e a `ZipOutputStream`;
3. Criamos o `FileInputStream` que lerá nosso arquivo, que é passado no construtor;
4. Criamos uma instância `ZipEntry` passando o nome do arquivo no construtor e depois passamos ela para a
   `ZipOutputStream` por meio do método `putNextEntry`;
5. Lemos os bytes do arquivo e gravamos eles, compactados, usando a nossa variável `zipOutputStream`;
6. Por fim, fechamos os streams instanciados.

Pronto! Agora temos um código Java capaz de zipar um arquivo e você entende como ele funciona! Não pararemos por aqui e
nos próximos tópicos você verá soluções para compactar múltiplos arquivos e pastas! Porém, sugiro que antes de continuar
a leitura, tente melhorar o código proposto e implementarmos suas próprias soluções. Deixo apenas uma dica: na
compactação de pastas a **recursividade** pode ser uma aliada.
<br>

## Zipar múltiplos arquivos

Antes de implementarmos a solução vamos encapsular o código anterior em um método com a seguinte
assinatura: `void zipFile(File file, String zipName)`. A principal mudança fica a cargo do recebimento do arquivo e do
nome do zip como argumentos. Vamos encapsular também a parte do código que adiciona arquivos dentro do zip, ele terá a
seguinte assinatura: `void addFileInZip(ZipOutputStream zipOutputStream, File file)`. Após essa refatoração nosso
primeiro código ficará como a seguir.

<script src="https://gist.github.com/danielarrais/a2f9d955519d8d18de30476f6482a221.js?file=block_two.java"></script>
<script src="https://gist.github.com/danielarrais/a2f9d955519d8d18de30476f6482a221.js?file=block_three.java"></script>

Agora que temos um método que adiciona um arquivo dentro do zip podemos criar o novo método que zipa vários. Nele temos
que receber uma lista de arquivos e o nome do zip, essa lista deverá ser iterada e ter cada item adicionado ao zip,
ficando assim:

<script src="https://gist.github.com/danielarrais/a2f9d955519d8d18de30476f6482a221.js?file=block_four.java"></script><br/>

## Zipar pastas

A zipagem de uma pasta é um pouco mais complexa - bem pouco mesmo - pois precisamos vasculhá-la em busca dos arquivos
para adicioná-los ao ZIP. Essa busca pode ser facilitada com uso de recursividade. Para isso vamos duplicar nosso
método `addFileInZip` adicionando mais um parâmetro chamado `parentPath` em sua assinatura. A ideia aqui é chamarmos ele
passando o arquivo/pasta e o nome do caminho deles, repetindo isso forma recursiva para cada arquivo ou pasta
encontrada. O método deve trabalhar receber tanto pastas quanto arquivos. Para melhor entendimento a seguir temos a
implementação do que foi comentado.

<script src="https://gist.github.com/danielarrais/a2f9d955519d8d18de30476f6482a221.js?file=block_five.java"></script>

Para verificar o File passado é um arquivo ou uma pasta utilizei o método `isDirectory()`. Se for uma pasta eu itero os
arquivos dela - obtido por meio do método `listFiles()`, chamando o método de forma recursiva e passando para ele o path
do arquivo/pasta concatendado com `parentPath`. A concatenação é necessária para que ao chegar em um arquivo, o path
recebido seja completo. Caso seja um arquivo, ele adiciona o arquivo ao ZIP normalmente, concatenando a
váriavel `parentPath` ao seu nome para que seu path dentro do zip fique correto.

E como você deve ter notado, agora o código do primeiro método `addFileInZip` está duplicado. Para resolver isso você
pode substituir todo o código do `addFileInZip` original por uma chamada do novo, passando no terceiro paramêtro uma
String vazia. Assim deixamos nosso código mais clean e manutenível.

Com essa refatoração no `addFileInZip` podemos perceber que não precisamos criar um método exclusivo para compactação
de pasta, pois o `zipFile` já vai ser capaz de tal tarefa. Porém, por questão de semântica, você pode renomêá-lo
para `zipFileOrFolder` ou criar um método chamado `zipFolder` que chama o `zipFile`.

O resultado final pode ser visualizado abaixo:

<script src="https://gist.github.com/danielarrais/a2f9d955519d8d18de30476f6482a221.js?file=block_six.java"></script>

<br />

#### Dicas ou sugestões? Deixe seu comentário!

<div class="powr-comments" id="09b98bed_1619106882"></div><script src="https://www.powr.io/powr.js?platform=html"></script>