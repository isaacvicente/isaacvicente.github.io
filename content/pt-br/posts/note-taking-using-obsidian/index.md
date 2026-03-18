---
title: Sistema de anotações usando Obsidian e o método PARA
date: 2024-07-13
summary: O método PARA e como você deve fazer anotações usando o Obsidian
toc: true
readTime: true
tags: [note-taking]
---

## Introdução

Sou o tipo de pessoa que esquece as coisas facilmente (não _tão_ facilmente assim, mas você
sabe como é) e às vezes penso que preciso de algo para anotar o que
preciso lembrar depois. O que realmente preciso lembrar, geralmente crio um
evento no Google Calendar para não perder. Mas e as novas ideias
que tenho? E as coisas que quero fazer? Todas essas coisas eventualmente esqueço e
não lembro mais. Isso é ruim.

Mas o que preciso para começar? Bem, eu realmente queria a forma mais simples
possível para criar minhas notas e armazená-las em algum lugar. Já tentei
[Notion](https://www.notion.so/) (como todo mundo tentou, eu acho) e assisti alguns
vídeos de produtividade. Complicado demais. Eu precisava de algo mais simples.
E foi aí que encontrei o [Obsidian](https://obsidian.md/).

O que eu gosto no Obsidian? Bem, há algumas vantagens em usar o Obsidian
em relação a outros apps de anotações:

* **Formato Markdown**: isso é uma grande vitória para mim, já que Markdown é o formato
  mais simples para escrever texto. Em qual formato os READMEs são escritos? Você adivinhou:
  Markdown. Além disso, posso escrever em qualquer editor compatível com Markdown. Na
  verdade, estou escrevendo este post no Vim :)
* **Facilidade de edição**: É muito fácil e simples escrever notas no Obsidian.
  Ele formata os arquivos Markdown da forma que eu quero.
* **Arquivos locais**: Tudo que escrevo é armazenado como arquivos locais, não na
  nuvem. Dessa forma, quem possui o conhecimento (ou seja, os arquivos) sou eu, não alguma
  empresa.
* **Portabilidade**: O Obsidian também oferece um aplicativo móvel. Assim, posso anotar
  o que preciso e ver depois no meu celular. Você pode sincronizar seu
  computador com o celular usando o serviço pago que o Obsidian oferece. Mas você pode
  sincronizar suas notas usando outra coisa. Eu uso o
  [Syncthing](https://docs.syncthing.net/intro/getting-started.html) para sincronizar minhas
  notas entre meu laptop e meu smartphone.

> Vou escrever um post sobre sincronização de notas usando Syncthing em breve, então fique ligado.

## Método PARA

O método PARA é uma forma de organizar sua vida e não apenas notas. É
um método criado por Tiago Forte em seu livro [Building a Second
Brain](https://a.co/d/cJWvCuL).

PARA é um acrônimo para:

* **P**rojetos (Projects)
* **Á**reas (Areas)
* **R**ecursos (Resources)
* **A**rquivo (Archive)

### Projetos

Você pode pensar em um _projeto_ como uma ordem de tarefas que precisam ser feitas para
alcançar um determinado objetivo. Um projeto também tem um estado final claro e um período de
alguns meses. Por exemplo, planejar férias pode ser um projeto, assim como
criar um blog.

No método PARA, haveria uma pasta "Projetos" e você coletaria
informações para completar um determinado objetivo (ou seja, um _projeto_). Para isso,
você escreveria notas para suas pequenas tarefas, podendo ser uma única nota para
uma determinada tarefa ou muitas notas para outra tarefa, dependendo da
complexidade da tarefa.

### Áreas

Uma área é um projeto que nunca termina. Pense em uma área como uma área da sua vida,
ou seja, envolve um engajamento contínuo e terá um papel importante na sua
vida. Pode ser: família, trabalho, amigos, sua saúde.

Então, você pode fazer anotações sobre como está indo ou como pode melhorar. Na verdade,
você pode fazer uma anotação de qualquer coisa relacionada a alguma das suas áreas.

Perceba que essas pastas podem ser movidas para outras pastas. Talvez você queira
comprar uma casa nova. É apenas um plano com certas tarefas. Então, se encaixa melhor na
pasta _Projetos_. Com o tempo, você realiza as tarefas e comprou a
casa. Agora, você tem que cuidar dela, e não sabe por quanto tempo
terá a casa. É um projeto e não tem tempo de fim. Então, você pode mover
essa pasta que nomeou "Casa" para sua pasta _Áreas_.

### Recursos

Recursos são coleções de informações que podem ser úteis no futuro, ou seja,
são referências. Bem, para dar um exemplo, eu costumava reunir esse tipo
de informação nos favoritos do meu navegador. Tenho várias coleções de pastas
nos meus favoritos, como _Linux_, _Programação_, _Animes/Mangá_, etc. Na
pasta de Linux há outras pastas relacionadas a Linux. Esses são todos os links
que achei úteis e/ou interessantes. Não sei, talvez eu use no
futuro ou seja útil para alguém me pedindo ajuda.

### Arquivo

É aqui que você coloca notas que não são úteis atualmente ou no futuro
próximo, mas que você não quer jogar fora.

Imagine que você fez uma nota de 3 anos atrás. Você não a acha tão útil mais,
mas, quem sabe, pode ser útil no futuro. Então, é uma nota que você coloca na
pasta _Arquivo_.

Às vezes você fará uma conexão inesperada entre uma nota que está escrevendo
agora com essa mesma nota que fez 3 anos atrás, e isso será
mágico. Também é bom ver como evoluímos ao longo do tempo, o que costumávamos
pensar antes, e como pensamos _agora_.

Além disso, você pode mover um _projeto_ ou até mesmo uma _área_ para a pasta Arquivo, já que é
algo que você realmente não precisa mais.

### Extra: Caixa de Entrada

Lembra quando eu disse que queria algo sem atrito para escrever notas? Imagine
que você escreveu uma nota. Onde colocar essa nota? Bem, não perca seu tempo pensando
nisso. Se a pasta não é óbvia, coloque na pasta _Caixa de Entrada_.

A pasta Caixa de Entrada é o lugar onde você coloca uma nota que acabou de fazer e não sabe onde ela
deveria estar.

Claro, depois, você tem que reservar um tempo para pensar onde as notas na
pasta Caixa de Entrada deveriam estar, para não acumular muitas notas lá. Dessa
forma, você pode fazer uma anotação quando quiser e colocá-la no lugar certo quando
tiver tempo para isso. Lembre-se: é o seu sistema de anotações, a
importância dele é fazer anotações e não organizá-lo da melhor forma possível
imediatamente.

### Estrutura de pastas

No meu Obsidian Vault (a _raiz_ das suas pastas e arquivos), organizo as
pastas assim:

```text
+ Obsidian Vault
  |
  +-- 0 Caixa de Entrada
  +-- 1 Projetos
  +-- 2 Áreas
  +-- 3 Recursos
  +-- 4 Arquivo
```

Como você pode ver, tudo está sob meu `Obsidian Vault` e há cinco
pastas abaixo dele. Numerei minhas pastas para que quando eu `ls` nas pastas, elas
fiquem na ordem que quero.

### Armadilhas e princípios

* **Tome decisões rapidamente**: não pense demais, escreva uma nota e coloque na
  pasta Caixa de Entrada. Se você sabe onde a nota deveria estar, não deveria levar mais
  de 3 segundos para decidir onde deveria estar.
* **Evite mover arquivos o tempo todo**: às vezes você vai pensar: bem,
  essa nota ficaria melhor aqui e ali, e essa outra ali, e assim por diante.
  Parece produtividade, mas geralmente é perda de tempo. Tudo bem fazer
  isso raramente.
* **Suas anotações devem ser eficientes para você**: Se você sempre pensa demais
  sobre onde uma determinada nota deveria estar, talvez você precise parar e refletir sobre o que
  você quer, e talvez o método PARA não seja para você. E tudo bem. Mas
  dê alguns meses de chance.

## Conclusão

O método PARA realmente nos ajuda a organizar nossas notas conforme elas crescem. O Obsidian, por
outro lado, é de longe o melhor editor Markdown disponível. Markdown é
um formato muito simples de aprender, então fazer anotações não deveria ser um problema. O
importante é: faça uma anotação. É só isso.

Eu mesmo sou iniciante nisso: tanto em fazer anotações quanto em fazer no Obsidian.
Escrevi este blog para poder consultar e saber o que devo estar fazendo. Também é
uma forma de fazer anotações, porque os próprios posts são escritos em Markdown.

## Links

Recomendo muito que você confira esses links:

* [(vídeo) Note-taking with the PARA method - Best for Beginners](https://youtu.be/oxUVn37-Igk?si=S0kjK8jslyoLRLJk)
* [(docs) Obsidian Help](https://help.obsidian.md/Home)
