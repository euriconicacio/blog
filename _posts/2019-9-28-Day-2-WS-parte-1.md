---
layout: post
title: Dia 2 de 100 - Web Scraping com Python - Parte 1 de 3
---

Vamos lá, separei essa postagem em três partes por comodidade e por gostar muito do tema e por achar que ele merece atenção especial. Esta primeira postagem é mais teórica, mas trago alguns exemplos para trazer a noção da profundidade do tema. 

*Long story short*: **web scraping** é uma técnica de programação que permite extrair informações de *sites* - *data collection technique*. Deixando de lado alguns conceitos teóricos, ela pode ser usada para uma série de objetivos, desde o controle de gastos públicos pelo cidadão até a modelagem gráfica do histórico de cotações de moedas minuto a minuto nos últimos *x* anos. Além de viabilizar a aquisição, uma de suas soluções é a estruturação dos dados conforme a necessidade do cliente - passando de um formato HTML para bancos de dados, planilhas, JSON, etc.

Todavia, há ainda um impasse no limiar de legalidade desta técnica que eu costumo chamar "da técnica à ética". Imaginemos, por exemplo, um cliente que trabalhe com precificação de imóveis por localidade e por características definidoras. É bastante intuitivo como sua produtividade, sua assertividade e, provavelmente, seu desempenho financeiro melhorariam se ele contasse com os dados dos principais *marketplaces* do nicho (VivaReal, Zap, ImóvelWeb, entre outras), atualizados mensalmente. Em contrapartida, pode-se imaginar a naturalidade com que um desses *marketplaces* abomine tal ação, podendo tentar enquadrá-la como ilegal (normalmente válida quando há *disclaimer* visível de proibição) ou mesmo impondo critérios de segurança para realização de buscas em seu site - recursos como captcha ou mesmo controle de acessos recorrentes de um mesmo endereço de IP em curtos intervalos de tempo.

Com isso em mente, costumo balizar quão *gray* é o *job* no juízo de dois principais aspectos: o nível de "proteção" dos dados pela empresa que os possui (há necessidade de quebrar um captcha? há controle de acessos? há proibição de acesso por *web scraping*?) e o nível de "legalidade" das técnicas usadas para suplantar estes obstáculos. Quanto mais travado for o banco e quanto menos *user-friendly* for a abordagem, mais escuro é o tom de cinza entre *black* e *white*.

Por fim, ainda neste postagem introdutória, adianto que não vou falar sobre ferramentas "prontas" para a coleta - API x ou biblioteca y. Prezo pelo desenvolvimento personalizado de soluções baseado em templates básicos de acesso e um código eficiente (eficaz e efetivo) que solucione o problema. Alerto ainda que não vou tratar de técnicas ilegais ou condenáveis para coleta, tampouco incentivar seu uso - pelo menos por enquanto.

Amanhã começaremos a parte prática, com exemplo básico, e depois de amanhã o bicho pega!

#100daysofcode #day2of100 #programming #coding #code #python #developer #coder #programmer #peoplewhocode #hacking #ethicalhacking #hacktheworld