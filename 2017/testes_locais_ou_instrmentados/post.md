# Android: testes locais ou instrumentados?

A plataforma Android é tradicionalmente bem adversa a testes. Como o ambiente em que desenvolvemos é um SO diferente daquele que nossa aplicação executará, práticas tradicionais de teste não se aplicam muito bem. Para dar um "jeitinho" podemos seguir as seguintes metodologias:

- Instalar em um dispositivo ou emulador o aplicativo que queremos testar e enviar "comandos" para ele como se fôssemos um usuário
- Fazer um clone de todo o sistema que o aplicativo executa que rode no mesmo sistema em que desenvolvemos

## Testes instrumentados

A primeira forma de execução é mandar comandos como se fôssemos um usuário. Isso é feito por uma outra aplicação que "intrumenta" aquela que queremos fazer asserções. Dessa forma sempre fazemos a instalação das aplicações juntamente, isso é: elas executam em um aparelho ou emulador.

A plataforma Android sempre nos proveu esta abordagem com o nome "testes instrumentados". Estes "comandos" que enviamos são executados por um framework de instrumentação. Temos classes como `android.test.ActivityInstrumentationTestCase2` ou `android.test.InstrumentationTestCase`. Estas provêem a API base para criar testes com comandos de instrumentação.

Quando executamos estes testes é importante saber que na verdade estamos instalando duas aplicações: a primeira, que é a nossa aplicação real, e a aplicação de teste. Esta última é feita pelo código que colocamos na pasta "androidTest". Ambas as aplicações são APKs (Android Package) completos e não apenas as classes de teste. Isso quer dizer que podemos ter uma pasta de recursos ("res"), `AndroidManifest.xml`, assets e tudo mais que uma aplicação tem.

Hoje em dia o Google não nos recomenda usar o pacote `android.test` diretamente. No lugar dele há uma biblioteca de suporte para testes. Ela tem várias melhorias sobre o pacote de testes normal incluindo compatibilidade com o JUnit 4 (por padrão o Android usa o JUnit 3). Nesse pacote também temos a classe `android.support.test.InstrumentationRegistry` que facilita bastante o acesso ao contexto de instrumentação. Os métodos nesta classe, no entanto, podem parecer controversos no começo.

Na classe `android.support.test.InstrumentationRegistry` temos:

- `getContext()`: ao contrário do que possa parecer, este método não retorna o contexto da aplicação que queremos testar. Este é o contexto da aplicação de teste. Por meio dele podemos acessar os assets, recursos ("res") e etc da nossa aplicação de teste. Lembre-se que neste tipo de teste nós instalamos dois APKs completos, e este contexto é do APK de teste.

- `getTargetContext()`: este é o contexto da aplicação que queremos testar.

Cuidado com essas diferenças. Se não estiver conseguindo encontrar algum asset, então tenha certeza que está usando o contexto correto.

## Testes locais

Outro tipo de testes no Android quando executamos testes puros em Java. No Android, nós não executamos bytecode Java. Java é uma linguagem que compila para uma representação intermediária que é executada pela máquina virtual, neste caso a JVM (Java Virtual Machine). Android é parecido, mas diferente. Ele contém sua própria máquina virtual chamada Dalvik, que entende um bytecode diferente do Java.

O fato que ambas as máquinas virtuais entendem bytecodes diferentes faz com que as coisas sejam um pouco mais complicadas. As vezes nós queremos executar testes diretamente na JVM porque isso é mais fácil do que instalar dois APKs inteiros e deixar o sistema iniciar um contexto para cada. Executar os testes na mesma JVM que nosso ambiente de desenvolvimento usa faz com que a velocidade dos testes seja ordens de magnitude mais rápida.

Para fazer isso nós podemos tentar tirar qualquer dependência de classes do Android do nosso código usando padrões como MVP (Model View Presenter) ou outras arquiteturas. Normalmente isso nos força a incluir uma quantia significativa de burocracia no código. Criamos diversas interfaces e classes que apenas delegam as implementações (e quase sempre só temos uma implmentação diferente das interfaces nos nossos testes). Quando temos classes que não são dependentes do Android, então podemos testá-las diretamente na JVM sem problemas.

Mas o que acontece quando temos uma dependência e ela é invocada durante um teste? Recebemos uma exceção com a mensagem "Stub!". Então, caminho sem saída?

Não ainda! A alternativa é fazer com que o bytecode do Android fique compatível! Como? Bom, vamos reimplementar toda a plataforma do Android apenas para os nossos testes :) Na verdade, nós podemos pular algumas partes e apenas delegar outras para o código da JVM real. O fato é: já existe uma biblioteca que busca fazer justamente isso. Robolectric.

Então, quando usamos Robolectric nós podemos executar testes localmente na JVM como se fosse código Java normal sendo testado. No fim, dentro da sua aplicação a máquina virtual será diferente da JVM e isto é crucial. Não esqueça! Há coisas que podem dar errado no seu teste que nunca dariam errado em um aparelho Android real. Coisas como criptografia são realmente diferentes entre as plataformas.

## Qual tipo de teste devo usar então?

Bom, essa é a pergunta de ouro, certo? Como é no caso dessas perguntas: não há uma resposta correta. Normalmente testes instrumentados são focados em asserções de interações do usuário. Isto é, você envia comandos como se tivesse fazendo exatamente o que um usuário real da aplicação faria. Daí você faz asserções baseado nos resultados.

Do outro lado, testes na JVM são focados em asserções dos resultados de chamadas de métodos. Isto é, você executa um dado método com certas entradas e faz asserções nos resultados. Você não precisa do ciclo de vida inteiro ou de coisas como o tratamento de eventos de UX. Você apenas cria diversas entradas das quais você espera um certo resultado de saída.

Como já dissemos, a primeira abordagem casa melhor com testes de interface. O que acontece com a interface quando um botão é clicado ou se alguma lista é rolada pela tela? Isto é normalmente o que você quer testar com instrumentação.

Testes em Java puro querem encontrar lógica ruim nas implementações de métodos. Normalmente são baseados em lógica. Você quer garantir se seu cálculo de juros está cuidando de todos os casos.

Normalmente as aplicações Android não deveriam carregar muita lógica. Elas são o "front" de algum serviço. Exemplos são: Gmail, Facebook, Spotify e etc. Eles possuem muito código para lidar com a interação e bem menos para lidar com lógica de negócio. Isso porque as regras estão no "back" e não no "front", ou seja, nos servidores e no no client. Assim, em geral, deveria haver mais testes de instrumentação do que testes na JVM.

Isso não quer dizer que testes de instrumentação são sempre preferíveis, eles possuem alguns problemas. Por exemplo: eles são mais difíceis de tornar estáveis. Instabilidade pode ocasionar problemas de memória, dependências de outros aplicativos (testar abrir um e-mail) e por aí vai. No passado, isso tornava esse tipo de teste quase impossível. Ferramentas atuais como Espresso, TestButler do LinkedIn e o próprio `InstrumentationTestRunner` fazem com que os testes fiquem muito mais estáveis. Talvez, no entanto, não tanto quanto testes locais.

## Mas estão falando por aí sobre BDD e SBE. Eles não fazem a mesma coisa que os instrumentados?

Absolutamente não! Eles são outra camada de qualidade crucial para um produto completo. Primeiro, vamos esclarecer do que se tratam:

**BDD** Behaviour Driven Development (Desenvolvimento orientado a comportamento)
**SBE** Specification by example (Especificação por exemplo)

Uma coisa são testes unitários, nos quais focamos em uma única unidade de código ou de interação. Este é o objetivo dos nossos testes instrumentados e locais no Android. Nós não testamos um cenário do usuário completo apenas para garantir que um clique traga a mensagem esperada. Nós podemos iniciar diretamente a tela que queremos (`Activity`, `Fragment` ou `View`) e executar a ação diretamente sem precisar do contexto do cenário.

Não é assim que tanto BDD quanto SBE funcionam. Eles possuem outro objetivo que é uma asserção de caso de uso, você tem um cenário completo da jornada de um usuário. O que é especialmente interessante neles é que o foco em uma jornada os permite ser cross-plataforma. Isto quer dizer: provavelmente a mesma jornada deve ser usada para testar iOS, Android e a web. Quando algo dá errado a mensagem deveria ser consistente entre as plataformas. Não conseguimos isso com testes locais e/ou instrumentados.

O foco de BDD e SBE são de garantir a qualidade em um cenário do começo ao fim. Eles não usam mocks, stubs ou qualquer coisa. Eles navegam pelo aplicativo como um usuário do sistema faria e realmente fazem chamadas para os servidores. Assim, normalmente, eles precisam de algum ambiente no qual eles possam comprar coisas ou fazer qualquer transação que seu aplicativo faz. Se você quiser saber mais sobre BDD e SBE, pode dar uma olhada neste post aqui [https://www.concretesolutions.com.br/2014/12/16/introducao-bdd-e-cucumber/] ou nesta série aqui [https://www.concretesolutions.com.br/2016/09/22/especificacao-por-exemplo-1/]. 

## Ok, agora a pergunta de prata (a de ouro já foi usada): devemos escrever testes se já temos BDD ou SBE?

Você provavelmente já sabe a resposta. Claro que precisamos escrever testes! Como mencionado no último ponto, testes que escrevemos são diferentes em propósito do que os de BDD ou SBE. Nós estamos com o controle total de uma unidade de código ou interação. Isso quer dizer que você pode pegar as instâncias dos objetos em execução e garantir que eles estão no estado que você quer que eles estejam. Isso significa que você não precisa de uma API externa para exercitar situações de erro. Você DEVE mockar dependências externas e garantir apenas a execução do seu código.

Outro ponto diferente de BDD e SBE é que eles levam muito mais tempo para executar do que testes instrumentados. Com uma suíte de testes e jornadas relativamente grande é normal os testes levarem horas para executar.

O que é importante aqui é construir diversas camadas de qualidade. Desenvolvedores de aplicativos escrevem testes para seu código. Desenvolvedores de qualidade escrevem testes de jornadas do usuário para garantir os casos de uso. Ambos os testes não excluem ainda outras camadas de testes como de performance, segurança e assim por diante.

Ficou alguma dúvida ou tem algo a dizer? Aproveite os campos abaixo!

--
É desenvolvedor Android e gostaria de fazer parte de um time ágil e multidisciplinar? Clique aqui [https://www.concretesolutions.com.br/vagas/].
