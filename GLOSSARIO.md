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

**Código de saída** (_exit code_) — Número que um comando devolve ao terminar. **Zero
significa sucesso**; qualquer outro número significa falha. É por isso que uma esteira de
CI/CD consegue saber sozinha se o build passou.

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

**Dependência** — Um pacote de código pronto que o projeto usa. Ficam listadas no
`package.json` e são baixadas para `node_modules/`.

**devDependency** — Dependência usada só enquanto programamos (TypeScript, ESLint,
Prettier). Não vai junto para o servidor de produção. Analogia: martelo e furadeira ficam
na marcenaria; o cliente leva só o móvel pronto.

**dist** — Pasta com o código já compilado, pronto para rodar. De _distribution_. É gerada
pelo `npm run build` e **não** é versionada no Git.

---

## E

**EditorConfig** — Arquivo que faz qualquer editor de código respeitar as mesmas regras
básicas do projeto (tamanho da indentação, tipo de quebra de linha).

**Endpoint** — Um endereço específico da API que responde a alguma coisa. Ex.: `/health`.

**Enumeração** — Técnica de ataque em que alguém provoca erros de propósito, um atrás do
outro, e vai montando a planta do sistema pelas mensagens que voltam. É por isso que uma
mensagem de erro descuidada é um problema de segurança, e não só de acabamento.

**Erro esperado × erro inesperado** — O esperado foi previsto pela regra de negócio
("Protocolo não encontrado"): não é bug, é o sistema funcionando, e a mensagem foi escrita
para o cliente ler. O inesperado ninguém previu (banco fora do ar, bug, biblioteca que
quebrou): a mensagem veio de fora e pode conter qualquer coisa. Nesta API, só a do primeiro
tipo chega ao cliente — a régua é a **procedência** da mensagem, não o conteúdo dela.

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

**Handler** — Função encarregada de tratar uma situação específica. O `errorHandler` deste
projeto é **global**: registrado uma vez em `buildApp()`, ele atende toda falha de toda rota,
inclusive das que ainda nem foram escritas.

**Health check** — Rota que responde "estou vivo" para ferramentas de monitoramento. Se ela
parar de responder, o sistema de infraestrutura sabe que precisa reiniciar a aplicação.

**HTTP** — O idioma que navegadores e servidores usam para conversar. Define os métodos
(`GET`, `POST`, `PUT`, `DELETE`) e os códigos de status (200, 404, 500).

---

## I

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

## N

**Node.js** — O programa que executa JavaScript fora do navegador. É o que faz nossa API
funcionar no servidor.

**node_modules** — Pasta onde o npm guarda as dependências baixadas. Tem milhares de
arquivos, nunca se mexe nela na mão e nunca vai para o Git.

**npm** — _Node Package Manager_. O programa que instala dependências e executa os scripts
do `package.json`.

---

## P

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

---

## R

**Repository** — Camada que conversa com o banco de dados. Só entra e sai dado; **nenhuma**
regra de negócio mora aqui.

**Requisição** (_request_) — O pedido que o cliente manda para a API.

**Resposta** (_response_) — O que a API devolve, com um código de status e, normalmente, um
corpo em JSON.

**REST** — Conjunto de convenções para organizar uma API: cada coisa tem um endereço, e o
método HTTP diz o que fazer com ela.

**Rota** — A ligação entre um endereço (`/health`) e o código que responde por ele.

---

## S

**Schema** — Descrição formal do formato de um dado. No Fastify, declarar o schema da
resposta acelera a serialização **e** impede que campo não declarado vaze sem querer.

**Serialização** — Transformar um objeto do código em texto (JSON) para mandar pela rede.

**Service** — Camada que concentra a lógica de negócio. Analogia: o cérebro. É onde as
regras de verdade acontecem.

**src** — De _source_, código-fonte. A pasta onde escrevemos o TypeScript.

**Stack trace** — A lista de funções que estavam em execução no momento do erro, na ordem em
que foram chamadas. É o que diz a linha exata onde o problema nasceu. Vai para o log e
**nunca** para a resposta HTTP.

**Status code** — Número que resume o resultado da requisição: `200` deu certo, `404` não
encontrado, `500` erro do servidor. A faixa importa: `4xx` significa "o problema está do seu
lado" e `5xx`, "o problema está do meu lado".

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
