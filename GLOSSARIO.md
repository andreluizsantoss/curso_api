# Glossário

Todo termo técnico que aparece nas aulas, explicado em linguagem simples.
Este arquivo cresce a cada aula nova.

> Não decore. Volte aqui sempre que esbarrar em uma palavra desconhecida.
> Perguntar "o que é isso mesmo?" pela décima vez é normal e não é vergonha nenhuma.

---

## A

**API** — Sigla de _Application Programming Interface_. É a parte do sistema que atende
pedidos de outros programas, em vez de atender pessoas diretamente. Analogia: o garçom do
restaurante. Você não entra na cozinha; você pede ao garçom, ele leva o pedido e traz o prato.

**Ambiente** — O "lugar" onde o sistema está rodando. Os três comuns: `development`
(sua máquina), `staging`/homologação (cópia de teste) e `production` (o servidor real, que
os cidadãos usam).

**Assíncrono** — Código que não trava o programa enquanto espera algo demorado (ler um
arquivo, consultar o banco, chamar outra API). Em TypeScript se escreve com `async` e `await`.

---

## B

**Backend** — A parte do sistema que o usuário não vê: regras de negócio, banco de dados,
segurança. É o que estamos construindo aqui.

**Build** — O processo de transformar o código que escrevemos (TypeScript, em `src/`) no
código que o servidor executa (JavaScript, em `dist/`). Comando: `npm run build`.

---

## C

**Camada** (_layer_) — Cada instrução do `Dockerfile` produz uma camada, e o Docker guarda
cada uma em cache. Quando uma camada muda, ela e **todas as que vêm depois** são refeitas — é
por isso que a ordem das linhas do arquivo é decisão de desempenho, e não de estética.

**Contexto de build** — A pasta que o Docker recebe para trabalhar quando você roda
`docker build ... .` (o ponto final). O `.dockerignore` é a lista do que fica de fora dele.

**Container** — Uma execução de uma imagem. Se a imagem é a receita, o container é o bolo:
da mesma imagem sobem muitos containers ao mesmo tempo, e apagar um não afeta os outros nem
a imagem.

**Clickjacking** — Ataque em que um site coloca a página de outro dentro de uma moldura
invisível e engana a pessoa para clicar onde ela não vê. O cabeçalho `x-frame-options`, que o
Helmet liga, é o que impede a moldura de existir.

**Código de saída** (_exit code_) — Número que um comando devolve ao terminar. **Zero
significa sucesso**; qualquer outro número significa falha. É por isso que uma esteira de
CI/CD consegue saber sozinha se o build passou.

**CORS** (_Cross-Origin Resource Sharing_) — Regra que decide quais sites podem chamar uma API
pelo navegador de alguém. **Não é um porteiro na entrada da API:** a API responde normalmente,
com corpo e tudo, e apenas inclui — ou não — o cabeçalho `access-control-allow-origin`. Quem
descarta a resposta é o **navegador**, na máquina da pessoa. Contra um programa que fale HTTP
direto, o CORS não faz nada, e nunca teve essa pretensão: ele protege o cidadão, não o
servidor.

**CSP** (_Content Security Policy_) — Cabeçalho que restringe de onde uma página pode carregar
script, estilo e imagem. É a defesa central contra script injetado. Vale para páginas; uma API
que só responde JSON não tem página própria para proteger.

**CommonJS** — Sistema antigo de módulos do Node, que usa `require()`. Este projeto **não**
usa; usamos ESM.

**Compilar** — Traduzir código de uma linguagem para outra. Aqui, de TypeScript para
JavaScript, porque o Node só entende JavaScript.

**Controller** — A camada que recebe a requisição HTTP e devolve a resposta. Analogia: o
recepcionista. Ele anota o pedido e entrega o resultado, mas **não** resolve nada sozinho.

**CRLF e LF** — Os caracteres invisíveis que marcam o fim de cada linha de um arquivo de
texto. Windows grava dois (`CRLF`); Linux e Mac gravam um (`LF`). O projeto padroniza em
`LF` pelo arquivo `.gitattributes`, senão o mesmo arquivo teria conteúdo diferente conforme
quem clonou — e as ferramentas acusariam mudança onde ninguém mudou nada.

---

## D

**Dígito verificador** — Algarismo calculado a partir dos outros, que existe só para
detectar erro de digitação. O CPF tem dois. Refazer a conta prova que o número é **bem
formado**; não prova que ele existe, nem que pertence a quem apresentou.

**`.dockerignore`** — Lista do que não entra no contexto de build. Impede que o
`node_modules`, o `.git` e — principalmente — o `.env` com segredos entrem na imagem. Um
segredo que entra numa imagem fica nela para sempre, porque a camada anterior continua
legível mesmo se um comando posterior apagar o arquivo.

**`Dockerfile`** — Arquivo de texto com as instruções para construir uma imagem. As mais
usadas: `FROM` (de onde partir), `COPY` (trazer arquivos), `RUN` (executar durante a
construção) e `CMD` (o que rodar quando o container subir).

**Docker Compose** — Ferramenta que descreve um ambiente inteiro — vários containers, a rede
entre eles e o disco que eles usam — em um único arquivo versionado, o `docker-compose.yml`.
Se o `Dockerfile` é a receita de um prato, o Compose é o menu do jantar inteiro.

**`depends_on`** — Campo do Compose que define a ordem de partida dos serviços. Na forma
simples, ele espera apenas o container **existir**, e não o serviço lá dentro estar pronto
para atender. Para esperar de verdade, combina-se com `condition: service_healthy`, que só
libera o próximo serviço quando o `healthcheck` do anterior passar.

**Driver adapter** — Peça que ensina o ORM a conversar com um banco específico. No Prisma 7 ele
é **obrigatório**: `new PrismaClient()` sem adapter é erro. Para MySQL, o pacote é o
`@prisma/adapter-mariadb` — os dois bancos falam o mesmo protocolo.

**Dependência** — Um pacote de código pronto que o projeto usa. Ficam listadas no
`package.json` e são baixadas para `node_modules/`.

**devDependency** — Dependência usada só enquanto programamos (TypeScript, ESLint,
Prettier). Não vai junto para o servidor de produção. Analogia: martelo e furadeira ficam
na marcenaria; o cliente leva só o móvel pronto.

**dist** — Pasta com o código já compilado, pronto para rodar. De _distribution_. É gerada
pelo `npm run build` e **não** é versionada no Git.

**Dump** — Arquivo com o conteúdo inteiro de um banco, gerado pelo `mysqldump`: estrutura mais
todas as linhas. É o formato do backup — e, por trazer todo o dado pessoal, nunca vai para o
Git. Dump que nunca foi restaurado é esperança, não backup.

**Desligamento gracioso** (_graceful shutdown_) — Encerrar um programa terminando o que já
começou, em vez de parar no meio. Na prática: parar de aceitar requisição nova, esperar as
que estão em andamento responderem e só então sair. Sem isso, o deploy corta a requisição de
quem estava usando o sistema naquele segundo.

---

## E

**EditorConfig** — Arquivo que faz qualquer editor de código respeitar as mesmas regras
básicas do projeto (tamanho da indentação, tipo de quebra de linha).

**Endpoint** — Um endereço específico da API que responde a alguma coisa. Ex.: `/health`.

**Exclusão lógica** — Marcar um registro como excluído (preenchendo uma data) em vez de apagar
a linha. O registro some das consultas e continua no banco. Existe porque cadastro costuma ter
histórico ligado a ele — apagar transformaria esse histórico em referência quebrada, e `DELETE`
não tem desfazer.

**Enumeração** — Técnica de ataque em que alguém provoca erros de propósito, um atrás do
outro, e vai montando a planta do sistema pelas mensagens que voltam. É por isso que uma
mensagem de erro descuidada é um problema de segurança, e não só de acabamento.

**Erro esperado × erro inesperado** — O esperado foi previsto pela regra de negócio
("Protocolo não encontrado"): não é bug, é o sistema funcionando, e a mensagem foi escrita
para o cliente ler. O inesperado ninguém previu (banco fora do ar, bug, biblioteca que
quebrou): a mensagem veio de fora e pode conter qualquer coisa. Nesta API, só a do primeiro
tipo chega ao cliente — a régua é a **procedência** da mensagem, não o conteúdo dela.

**Expande/contrai** — Padrão para alterar tabela sem derrubar o sistema, em quatro passos:
acrescenta a coluna nova (opcional), passa a gravar nas duas, copia o dado que faltava e só
então remove a antiga. Ele existe porque, durante um deploy, o **código velho roda sobre o
schema novo** por alguns minutos.

**ESLint** — Ferramenta que analisa o código procurando problemas de **lógica e qualidade**
(variável nunca usada, comparação que nunca é verdadeira). Cuida do conteúdo, não da aparência.

**ESM** (_ECMAScript Modules_) — Sistema moderno de módulos do JavaScript, com `import` e
`export`. É o que este projeto usa.

---

## F

**Fastify** — O framework HTTP deste projeto. Cuida de receber requisições, encaminhar para
a rota certa e devolver respostas. Escolhido pela performance, pela validação por schema e
pelo logger já integrado.

**flat config** — Formato atual de configuração do ESLint, escrito em `eslint.config.js`.
Substituiu o formato antigo, em `.eslintrc.json`.

**Formatter** — Ferramenta que cuida da **aparência** do código (espaços, aspas, quebras de
linha). O nosso é o Prettier.

---

## H

**`HEALTHCHECK`** — Instrução do `Dockerfile` que manda o Docker perguntar de tempos em
tempos se a API responde de verdade. O veredito vem do código de saída do comando. Importante:
`unhealthy` é um **rótulo**, não uma ação — o container continua rodando, e quem age é quem
estiver orquestrando.

**Idempotente** — Operação que, repetida, tem o mesmo efeito de ter sido feita uma vez só. O
seed do projeto é idempotente: rodá-lo dez vezes deixa os mesmos três registros, porque usa
`upsert` em vez de `create`.

**Handler** — Função encarregada de tratar uma situação específica. O `errorHandler` deste
projeto é **global**: registrado uma vez em `buildApp()`, ele atende toda falha de toda rota,
inclusive das que ainda nem foram escritas.

**Helmet** — Conjunto de cabeçalhos de segurança que o navegador respeita, ligados de uma vez.
Eles existem no navegador há anos e vêm **desligados** por omissão, para não quebrar sites
antigos. Como CORS, é um pedido ao navegador: quem não for navegador ignora.

**Health check** — Rota que responde "estou vivo" para ferramentas de monitoramento. Se ela
parar de responder, o sistema de infraestrutura sabe que precisa reiniciar a aplicação.

**HTTP** — O idioma que navegadores e servidores usam para conversar. Define os métodos
(`GET`, `POST`, `PUT`, `DELETE`) e os códigos de status (200, 404, 500).

---

## I

**Imagem** — O pacote pronto e imutável com o código, as dependências e a versão exata do
Node. É a receita: dela saem os containers.

**Injeção de dependência** — Entregar a uma classe as coisas de que ela precisa, em vez de
ela mesma criá-las. No nosso código, o `HealthController` **recebe** o `HealthService` pelo
construtor. Isso é o que permite, nos testes, entregar um service falso.

**Import** — Instrução que traz código de outro arquivo. Neste projeto:
todo import é relativo e leva a extensão `.ts`, como em `./arquivo.ts` ou
`../../shared/env/index.ts`. O compilador troca para `.js` sozinho na hora do build.

---

## J

**JSON** — Formato de texto para representar dados, entendido por praticamente toda
linguagem. É o formato das respostas da nossa API.

---

## L

**Linter** — Categoria de ferramenta que "passa o pente fino" no código procurando
problemas. O nosso é o ESLint.

**Log estruturado** — Registro gravado como JSON, com campos separados, em vez de texto
solto. A diferença prática: dá para perguntar "me mostre todas as requisições que
demoraram mais de 1 segundo", o que é impossível com texto corrido.

---

## M

**Migration** — Arquivo versionado no Git que descreve, em SQL, uma alteração na estrutura do
banco: criar tabela, acrescentar coluna, criar índice. Sem migrations, a estrutura do banco só
existe dentro daquele banco — não há como reproduzi-la em outra máquina nem revisar a alteração
antes que ela aconteça.

**Migrate deploy** — Comando que **apenas aplica** as migrations já versionadas, sem criar
nenhuma e sem usar [[shadow-database]]. É o comando que roda no servidor; o `migrate dev` é o
da máquina de quem desenvolve.

---

## N

**Node.js** — O programa que executa JavaScript fora do navegador. É o que faz nossa API
funcionar no servidor.

**node_modules** — Pasta onde o npm guarda as dependências baixadas. Tem milhares de
arquivos, nunca se mexe nela na mão e nunca vai para o Git.

**npm** — _Node Package Manager_. O programa que instala dependências e executa os scripts
do `package.json`.

---

## O

**Orquestração** — Cuidar de vários containers como um conjunto: subir na ordem certa, ligar
uns aos outros, reiniciar o que cair. O Docker Compose orquestra em uma máquina; em produção,
esse papel costuma caber a ferramentas maiores. É a resposta para a pergunta que o
`HEALTHCHECK` deixa em aberto: o rótulo `unhealthy` só vira ação quando existe um orquestrador
escutando.

**ORM** (_Object-Relational Mapper_) — Ferramenta que traduz código em consulta ao banco. Em vez
de escrever SQL em texto, você declara a estrutura uma vez e trabalha com objetos tipados — e
errar o nome de uma coluna vira erro de compilação, não erro em produção.

**OpenAPI** — Formato padrão para descrever uma API HTTP: quais rotas existem, o que cada
uma recebe e o que devolve, campo por campo. O documento é JSON ou YAML, e o que o torna
valioso é ser **lido por máquina** — a partir dele, ferramentas geram página de documentação,
código de cliente e testes de contrato. Analogia: a planta baixa da casa. O texto "tem três
quartos" serve para conversar; a planta serve para construir.

---

## P

**PATCH × PUT** — Os dois alteram um recurso. O `PUT` espera o recurso **inteiro**, e o que
não vier é apagado; o `PATCH` altera **só o que foi enviado**. Em cadastro com muitos campos, o
`PUT` é um convite a apagar dado por omissão.

**Prefixo de versão** — O `/api/v1` no começo do endereço. Ele permite lançar uma `v2`
incompatível sem obrigar todos os sistemas que consomem a API a mudarem no mesmo dia. Rotas de
monitoramento ficam **fora** dele: quem as consulta é o alarme, não um integrador.

**package.json** — O painel de controle do projeto: nome, versão, dependências e os
atalhos de comando (`scripts`).

**package-lock.json** — Arquivo gerado pelo npm que registra a versão **exata** de cada
dependência instalada. Garante que todo mundo do time tenha exatamente as mesmas versões.

**peerDependencies** — Quando um pacote declara com quais versões de outro pacote ele
funciona. Foi por causa disso que este projeto fixou o TypeScript 6: o `typescript-eslint`
declara que só funciona com versões abaixo da 6.1.

**Pino** — O logger que vem dentro do Fastify. Um dos mais rápidos do ecossistema Node.

**Plugin** — No Fastify, um pedaço isolado de funcionalidade que se registra no app. Nossos
arquivos de rotas são plugins.

**Prettier** — Ferramenta que formata o código automaticamente, sempre do mesmo jeito.
Acaba com a discussão de "qual estilo é mais bonito".

**Produção** — O ambiente real, onde os cidadãos de verdade usam o sistema. Erro em
produção afeta pessoas.

**Promise flutuante** (_floating promise_) — Chamada a uma função `async` que ninguém
`await`-a nem trata com `.catch()`. Se ela falhar, existe um caminho de erro que o seu código
não vê: o processo morre sem passar pelo seu log. É por isso que o `server.ts` termina com
`start().catch(...)`.

**PID 1** — O primeiro processo de um sistema Linux, e o único processo de um container
comum. O Linux o trata de forma diferente: para ele **não existe ação padrão para sinal**, e
um `SIGTERM` que ninguém trata é simplesmente ignorado. É por isso que uma aplicação em
container precisa tratar sinais de propósito.

**Proxy reverso** — Programa que fica na frente da API, recebe todas as conexões e as repassa
para dentro. Do ponto de vista da API, todas as requisições passam a vir dele — e é por isso
que existe o `X-Forwarded-For`.

---

## R

**Retrocompatível** — Mudança que **não obriga ninguém a mexer no código de quem consome**.
Acrescentar campo opcional é; remover campo, renomear ou exigir o que era opcional não é. A
mesma pergunta vale para banco de dados, onde ela decide se a migration pode rodar com a versão
antiga ainda no ar.

**Rate limit** (limite de requisições) — Teto de quantas requisições um mesmo cliente pode
fazer numa janela de tempo. Diferente de Helmet e CORS, é a **própria API** que recusa, então
vale contra qualquer um. Neste projeto: 100 por minuto por IP, e 240 no `/health`, para não
bloquear o monitoramento. Quem estoura recebe **429**.

**Retrocompatível** — Alteração que o código **já em produção** aguenta sem quebrar.
Acrescentar coluna opcional é retrocompatível; remover coluna que o código velho lê não é. É a
pergunta que decide se uma migration pode subir com o sistema no ar.

**Repository** — Camada que conversa com o banco de dados. Só entra e sai dado; **nenhuma**
regra de negócio mora aqui.

**Requisição** (_request_) — O pedido que o cliente manda para a API.

**Resposta** (_response_) — O que a API devolve, com um código de status e, normalmente, um
corpo em JSON.

**REST** — Conjunto de convenções para organizar uma API: cada coisa tem um endereço, e o
método HTTP diz o que fazer com ela.

**Rota** — A ligação entre um endereço (`/health`) e o código que responde por ele.

**redact** — Recurso do logger que substitui o valor de campos declarados por
`[Redacted]` antes de a linha ser escrita. Protege credencial que alguém registre sem
pensar. Age sobre **campos de objeto**: texto dentro de uma URL não é alcançado por ele.

---

## S

**Shadow database** (banco de sombra) — Banco descartável que o `prisma migrate dev` usa para
conferir se as migrations, aplicadas em ordem, produzem exatamente o schema declarado. É o que
denuncia migration alterada à mão depois de aplicada. Neste projeto ele **vem pronto do
Compose**, porque o usuário da aplicação não tem — nem deve ter — permissão de criar bancos.

**Seed** (semeadura) — Arquivo que popula o banco com registros de partida, para que ele não
nasça vazio a cada recriação. Ver [[idempotente]].

**Schema** — Descrição formal do formato de um dado. No Fastify, declarar o schema da
resposta acelera a serialização **e** impede que campo não declarado vaze sem querer.

**Serialização** — Transformar um objeto do código em texto (JSON) para mandar pela rede.

**Service** — Camada que concentra a lógica de negócio. Analogia: o cérebro. É onde as
regras de verdade acontecem.

**Serviço** (no Docker Compose) — Uma peça do ambiente descrita no `docker-compose.yml`: a
API, o banco de dados. Cada serviço vira um container ao subir. O nome do serviço também é o
**endereço de rede** dele: dentro da rede do projeto, `mysql` resolve para o IP do container do
banco sem ninguém configurar nada.

**src** — De _source_, código-fonte. A pasta onde escrevemos o TypeScript.

**Swagger** — Nome do conjunto de ferramentas que popularizou o formato OpenAPI. Na prática,
os dois nomes aparecem juntos: o formato é OpenAPI, e as ferramentas em volta dele costumam
se chamar Swagger. Neste projeto, o `@fastify/swagger` **gera** a especificação e não publica
endereço nenhum.

**Swagger UI** — A página que lê uma especificação OpenAPI e a transforma em documentação
navegável, com botão para disparar cada rota de verdade. É quem publica `/documentation`,
`/documentation/json` e `/documentation/yaml`.

**Stack trace** — A lista de funções que estavam em execução no momento do erro, na ordem em
que foram chamadas. É o que diz a linha exata onde o problema nasceu. Vai para o log e
**nunca** para a resposta HTTP.

**Status code** — Número que resume o resultado da requisição: `200` deu certo, `404` não
encontrado, `500` erro do servidor. A faixa importa: `4xx` significa "o problema está do seu
lado" e `5xx`, "o problema está do meu lado".

**SIGTERM** — Sinal que pede a um processo que termine. Pode ser tratado: é o processo que
decide o que fazer ao recebê-lo. É o que o `docker stop` envia primeiro.

**SIGINT** — O mesmo pedido, vindo do teclado: é o `Ctrl+C`. Também pode ser tratado.

**SIGKILL** — Encerramento forçado. **Não** pode ser tratado nem ignorado: nem chega ao
processo. Um container que sai com código **137** (128 + 9) morreu assim.

---

## T

**TDD** (_Test Driven Development_) — Desenvolvimento guiado por testes. O ciclo é: escrever
o teste, **ver ele falhar**, escrever o código até passar, e então melhorar o código com o
teste de guarda. O passo do meio é o que parece bobo e é o mais importante: um teste que
nunca falhou não provou nada.

**TypeScript** — JavaScript com verificação de tipos. Avisa dos erros **enquanto você
escreve**, em vez de deixar o programa quebrar depois, com o usuário na frente.

**tsx** — Ferramenta que executa TypeScript diretamente, sem precisar compilar antes. É o
que faz o `npm run dev` recarregar sozinho a cada arquivo salvo.

**Tipagem estrita** (`strict`) — Modo do TypeScript que ativa todas as verificações de
segurança. Reclama mais, e é exatamente esse o objetivo.

---

## U

**uptime** — Há quanto tempo o processo está no ar, em segundos. Usado por ferramentas de
monitoramento para detectar aplicação que reinicia sozinha em looping.

---

## V

**Volume nomeado** — Área de disco gerenciada pelo Docker que **sobrevive** ao container ser
destruído. É onde o banco de dados guarda os dados: sem ela, tudo morre junto com o container,
que é descartável por natureza. O `docker compose down` preserva o volume; o `down -v` o apaga,
sem confirmação e sem volta.

---

## X

**X-Forwarded-For** — Cabeçalho HTTP em que um proxy escreve o endereço de quem realmente
chamou, já que a conexão que chega à API é a dele. Qualquer cliente pode enviá-lo, então
acreditar nele é uma decisão consciente — a do `trustProxy`.
