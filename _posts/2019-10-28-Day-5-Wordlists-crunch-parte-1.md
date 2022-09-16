---

layout: post

title: Dia 5 de 100 - Wordlists com Crunch - Parte 1 de 2
---

Conforme apontado na última postagem, hoje abordaremos algo um pouco mais _hackingtheworld-like_. Entretanto, começaremos aos poucos, com um tema "acessório" e não necessariamente "aplicado" - pelo menos por enquanto. Trata-se da criação e da manipulação de **dicionários** (ou _**wordlists**_) com o software/script **crunch**. Esse tópico será dividido em duas partes - uma mais simples, outra mais complexa, mas ambas mutuamente complementares e com informações adicionais "para a vida". 🏴‍☠️ 

Antes de mais nada, vamos entender o que cada uma das expressões do título significa. 

### A tal da _wordlist_

Uma _**wordlist**_ nada mais é que um conjunto de _strings_ que será testada, uma a uma, por alguma aplicação (normalmente em uma abordagem de _brute force_  [força bruta]) para descobrir uma senha - seja esta de um sistema, um arquivo, um site, uma rede sem fio, entre outras. Por exemplo, usa-se uma _wordlist_ associada ao Hydra ou ao John the Ripper para descobrir senhas de emails, sites ou arquivos; usa-se uma _wordlist_ associada ao _aircrack-ng_ para descobrir a senha de uma rede sem fio; entre outras aplicações. Como os métodos constituem tentar _string_ por _string_ à exaustão, seu uso é conhecido justamente como força bruta - o método vai persistir em tentativa-e-erro até encontrar a resposta correta.

Abrindo um parênteses bastante pertinente, é exatamente daqui surge a ideia de utilização "senhas fortes" (em algum dos 100 dias, abordarei o tópico). Suponha que alguém deseje descobrir a senha de sua rede sem fio e que, por meios de engenharia social, descubra que ela é composta apenas por 8 caracteres numéricos. A quantidade de combinações possíveis de 8 caracteres numéricos, ou seja, o número máximo de tentativas que alguém terá que realizar para descobrir tal senha, é 10^8 = 100 milhões de combinações/tentativas máximas. Parece muito, não? Mas infelizmente não é. Se, ainda por engenharia social, fosse possível descobrir os 4 últimos caracteres da senha (suponhamos, "1983"), bastaria descobrir os 4 primeiros - para tanto, há apenas 10^4 = 10 mil combinações/tentativas máximas possíveis! Para piorar: se esses quatro últimos algarismos forem relacionados à data de nascimento de algum familiar, talvez fosse possível descobrir a senha com 5 minutos de Google (não necessariamente Google Hacking, só buscas normais no Google) com UMA ÚNICA TENTATIVA. Já uma senha de 8 caracteres contendo letras maiúsculas, minúsculas, números e caracteres especiais pode ser da ordem de 10^19 = 10 TRILHÕES de combinações/tentativas máximas. Então, lição de casa: assegure-se que suas senhas são suficientemente seguras sempre - desde sua rede sem fio até seu sistema corporativo estratégico! 😉

Voltando à _wordlist_ , sua criação pode ser tão abrangente quanto possível - pode-se ter um dicionário com algumas dezenas de GB com trilhões de combinações de caracteres.. 🙄 - ou tão restrito quanto se queira - vide caso da senha numérica de 8 dígitos terminada em "1983". Note-se que, apesar de existir ainda um terceiro caso possível, considerando um público brasileiro, um determinado nicho desse grupo ou um determinado alvo, quando se usa um dicionário com milhares de senhas mais "comuns", este não será abordado aqui, pois sua criação é feita por amostragem, não pela combinação de caracteres. Entretanto, para os dois primeiros casos e para todos que a eles se assemelham, a geração do dicionário dá-se com o uso do o software/script **crunch**.

### O tal do _crunch_

#### Descrição

_Long story short_, o _*crunch*_  é um _software_ disponivel no magnífico ambiente Linux (quase tudo que eu postar aqui valerá tao somente para esse ambiente, acostume-se e adapte-se! 😜) para gerar _wordlists_ através de elaboração de todas combinações possíveis de um conjunto de caracteres informados. O _crunch_ já vem pré-instalado no Kali Linux, porém também pode ser instalado na sua distro com um simples `sudo apt install crunch`. 

Obs.: recomenda-se leitura e consulta do manual do comando (`man crunch`) para ter uma noção completa de todas as suas opções. Aqui vou explorar umas duas ou três somente. 

#### Gerando uma  _wordlist_ simples

A sintaxe básica do _crunch_ é `$ crunch min max [char] -o arquivo.txt`, onde _min_ e _max_ representam respectivamente o tamanho mínimo e máximo dE cada _string_ gerada, [char] é uma sequência de caracteres que comporão cada uma dessas _strings_ e _-o arquivo.txt_ é a opção para gravar o _output_ em "arquivo.txt". Por exemplo:

```bash
$ crunch 5 10 1234567890 -o dic.txt
```

Grava no arquivo "dic.txt" um dicionário com todas as combinações possíveis de números (de 1 a 0) contendo de 5 a 10 caracteres. Parece simples, não? Realmente é, e não piora muito.

Para "piorar" um pouco, suponhamos que você esteja encarando a situação já supramencionada da senha de rede sem fio com exatamentr 8 dígitos e terminada com "1983". Para gerar a _wordlist_ que com certeza terá a senha desejada, basta usar o _crunch_ indicando com a opção _-t @@@@1983_ e indicando o padrão a ser seguido (4 caracteres numéricos variando, antecedendo a porção fixa "1983") e novamente a opção _-o arquivo.txt_ para gravar o _output_ em "arquivo.txt". O comando final ficaria assim:

```bash
$ crunch 8 8 1234567890 -t @@@@1983 -o arquivo.txt
```

Essa versão do _crunch_ é especialmente interessante para descobrir senhas de redes sem fio em um roteador da Claro/NET com configurações de fábrica. Neste tipo de roteador, a senha padrão é igual a seu endereço MAC sem os 4 primeiros caracteres; todavia, nem sempre tem-se acesso a meios para descobrir o endereço MAC em questão. Contudo, o nome padrão de uma rede em um roteador desse tipo é algo do tipo "NET_5GA245BD", de tal forma que o trecho "A245BD" corresponde aos 6 últimos caracteres do endereço MAC do roteador! Sendo assim, faltariam apenas os 2 CARACTERES HEXADECIMAIS iniciais da senha para ter pleno acesso à rede sem fio - sim, são 256 senhas possíveis, daria até para testar na mão uma a uma. O comando para gerar o dicionário que conterá essa senha será:

```bash
$ crunch 8 8 1234567890ABCDEF -t @@A245BD -o dic.txt
```

### Conclusão parcial

Nesta primeira parte, tivemos uma introdução ao tópico de _wordlist_ e ao uso do _crunch_, mostrando inclusive a geração de dicionários simples e para ocasiões "especiais". Na próxima postagem, adianto que apresentarei uma geração um pouco mais complexa, com base no arquivo _charset.lst_. 😉

Em caso de alguma eventual dúvida, sigo à disposição para discutir o assunto.

#100daysofcode #day5of100 #programming #coding #code #python #developer #coder #programmer #peoplewhocode #hacking #ethicalhacking #hacktheplanet #wordlist #crunch

