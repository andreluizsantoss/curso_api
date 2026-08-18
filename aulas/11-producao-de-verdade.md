# 🛑 Aula 11: Produção de verdade — desligamento, proxy e logs

Na Aula 10 a API virou uma imagem que sobe em qualquer máquina. Nesta aula ela aprende três
coisas que **só existem depois** de estar dentro de um container:

1. **Morrer direito.** Hoje, quando alguém derruba a API, ela morre no meio de uma
   requisição — e quem estava do outro lado recebe a linha cortada, sem erro e sem resposta.
2. **Saber com quem está falando.** Em produção sempre existe um proxy na frente. Sem
   configurar isso, a API acha que **todos** os clientes são a mesma pessoa.
3. **Registrar sem entregar segredo.** Um token enviado por engano na URL fica hoje gravado
   no log, por extenso, para sempre.

Os três problemas já existiam antes. O que a Aula 10 deu foi a possibilidade de **provocá-los
e assistir** — que é a única forma de aprender isto sem decorar.

> [!IMPORTANT]
> Esta aula mede tudo com cronômetro. Cada número que aparece aqui saiu de uma execução real,
> e você vai reproduzir as mesmas medições na sua máquina. Se o seu número der diferente do
> nosso, isso é informação, não erro — anote e siga.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar a diferença entre `SIGTERM`, `SIGINT` e `SIGKILL`, e por que só dois deles podem
  ser tratados.
- Fazer a API terminar as requisições em andamento antes de sair, com código de saída 0.
- Entender por que o prazo de desligamento é **um acordo entre dois lados**, e o que acontece
  quando os dois discordam.
- Colocar um proxy na frente da API, ver o IP errado sendo registrado, e corrigir.
- Demonstrar como um cabeçalho forjado passa por cima do limite de requisições — e por que a
  configuração segura é a desconfiada.
- Fazer o log mudar de volume conforme o ambiente e nunca gravar dado sigiloso.

---

## 📋 Pré-requisitos

Você precisa ter concluído a **Aula 10** e ter:

- o Docker Desktop instalado e **em execução**;
- o projeto passando no `npm run check`;
- o terminal aberto na pasta do projeto.

Confirme antes de começar:

```bash
docker --version
npm run check
```

Os dois precisam terminar sem erro. Se o `docker --version` responder mas qualquer comando
`docker` seguinte reclamar de conexão, o Docker Desktop está instalado mas não está rodando —
abra-o e espere a baleia ficar estável.

---

## 💣 Capítulo 1: A dor, medida com cronômetro

Vamos provocar o problema antes de falar dele. Para isso precisamos de uma requisição que
**demore** — todas as rotas de hoje respondem em poucos milissegundos, e o problema acontece
numa janela pequena demais para se ver.

### O andaime

Crie o arquivo `src/exemplo-demorado.ts`:

```ts
/**
 * Rota de exemplo — ARQUIVO TEMPORÁRIO, apagado ao final desta aula
 *
 * Ela existe por um motivo só: para haver uma requisição **lenta o bastante**
 * para ser cortada no meio enquanto o servidor é desligado. Sem isso, toda
 * requisição desta API termina em poucos milissegundos, e o problema do
 * desligamento abrupto nunca aparece — não porque ele não exista, mas porque a
 * janela em que ele acontece é pequena demais para se ver a olho nu.
 *
 * Não faz parte da API. Não vai para produção.
 */

import type { FastifyInstance } from 'fastify'

/**
 * Quanto tempo a rota finge estar trabalhando.
 *
 * Cinco segundos é maior que o tempo entre apertar o `docker stop` e o container
 * sumir, e menor que o prazo de desligamento que vamos configurar. É essa
 * relação — e não o número em si — que faz a demonstração funcionar.
 */
const DEMORA_MS = 5_000

/**
 * Registra uma rota que demora de propósito.
 *
 * @param app Instância do Fastify que vai receber a rota.
 */
export function exemploDemoradoRoutes(app: FastifyInstance): void {
  app.get('/exemplo-demorado', async (request) => {
    request.log.info('Comecei um trabalho demorado.')

    await new Promise((resolve) => setTimeout(resolve, DEMORA_MS))

    request.log.info('Terminei o trabalho demorado.')

    return { terminou: true }
  })
}
```

> [!NOTE]
> Este é o único arquivo do curso que você escreve sabendo que vai apagar. Ele é um
> **andaime**: existe para construir o que está em volta, e sai quando a obra termina. O
> Capítulo 8 apaga.

E registre a rota no `src/app.ts`, logo depois do `app.register(healthRoutes)`:

```ts
import { exemploDemoradoRoutes } from './exemplo-demorado.ts'
```

```ts
  app.register(healthRoutes)

  // TEMPORÁRIO — demonstração da Aula 11. Sai daqui antes do fim da aula.
  app.register(exemploDemoradoRoutes)
```

### A medição

Construa a imagem e suba o container:

```bash
docker build -t curso_api .
docker run -d -p 3333:3333 --name curso_api curso_api
```

Agora, em um terminal, dispare a requisição lenta:

```bash
curl http://127.0.0.1:3333/exemplo-demorado
```

E, em **outro terminal**, com a requisição ainda no ar, derrube o container:

```bash
docker stop curso_api
```

Volte ao primeiro terminal. Foi isto que aconteceu na medição real:

```
CLIENTE: |HTTP=000|CURL=52 | duracao=4220ms
DOCKER STOP levou 3501ms
codigo de saida do container: 137
```

Três informações, e nenhuma delas é boa:

| O que apareceu      | O que significa                                                          |
| :------------------ | :----------------------------------------------------------------------- |
| `HTTP=000`          | Não houve resposta. Nem erro, nem status: a conexão simplesmente sumiu.  |
| `CURL=52`           | "Empty reply from server" — o servidor fechou no meio da frase.          |
| código de saída 137 | 128 + 9. O 9 é o `SIGKILL`: o container **não** saiu, ele foi executado. |

E o log do container termina assim:

```
{"msg":"Comecei um trabalho demorado."}
```

Não existe "Terminei". O trabalho parou no meio.

> [!WARNING]
> Imagine que a rota lenta fosse "registrar um protocolo": gravar no banco, gerar o número e
> mandar o e-mail. Cortada no meio, ela pode ter gravado o protocolo **sem** ter avisado
> ninguém — e o cidadão, que não recebeu resposta, vai tentar de novo e gerar o segundo.

---

## 📖 Capítulo 2: Sinais — como se pede a um programa que morra

Quando algo quer encerrar um processo, não existe "botão de desligar". Existe **sinal**: uma
mensagem curta que o sistema operacional entrega ao processo.

### A analogia

Pense em uma agência prestes a fechar:

| Sinal     | O equivalente na agência                                                                              |
| :-------- | :---------------------------------------------------------------------------------------------------- |
| `SIGTERM` | O gerente avisa: "vamos fechar, termine o atendimento que está na sua mesa e não chame mais ninguém." |
| `SIGINT`  | A mesma coisa, dita por quem está ali dentro — é o `Ctrl+C` de quem desenvolve.                       |
| `SIGKILL` | O prédio é lacrado com todo mundo dentro. Ninguém termina nada, ninguém é avisado.                    |

Os dois primeiros **podem ser tratados**: o programa decide o que fazer ao recebê-los. O
`SIGKILL` **não pode** — ele nem chega ao programa, é o sistema operacional derrubando na
marra. É por isso que "encerrar direito" é sempre uma corrida contra ele.

### O que o `docker stop` faz

O `docker stop` não mata: ele **pede**. A sequência é sempre a mesma:

1. envia `SIGTERM` ao processo principal do container;
2. espera um prazo;
3. se o processo ainda estiver vivo, envia `SIGKILL`.

Na medição do capítulo anterior, o container saiu com 137 — ou seja, chegou até o passo 3. O
`SIGTERM` foi enviado, e **nada aconteceu**.

### Por que nada aconteceu: o PID 1

Dentro de um container, o seu processo é o **PID 1** — o primeiro e, normalmente, o único. E
o Linux trata o PID 1 de forma diferente de todos os outros: para ele, **não existe ação
padrão para sinal**. Se ninguém registrar um tratamento, o sinal é simplesmente ignorado.

Fora do container, um `SIGTERM` sem tratamento encerra o processo. Dentro dele, o mesmo
`SIGTERM` sem tratamento **não faz absolutamente nada**.

Dá para conferir isso sem envolver a nossa API — qualquer programa serve:

```bash
docker run -d --name teste-pid1 node:24-slim node -e "setInterval(()=>{},1000)"
docker stop teste-pid1
docker inspect teste-pid1 --format "{{.State.ExitCode}}"
docker rm teste-pid1
```

O resultado medido foi `137` outra vez, depois de o `docker stop` levar 3,4 segundos
esperando à toa. Um programa que não trata sinal nenhum, dentro de um container, **só** morre
de `SIGKILL`.

> [!TIP]
> Este é o mesmo tipo de armadilha da Aula 09 (Helmet e CORS são **pedidos** ao navegador) e
> da Aula 10 (`unhealthy` é um **rótulo**, não uma ação). Repare no padrão: em todos os três,
> algo tem nome de ação mas é só um recado. Quem age é sempre alguém — e vale sempre a pena
> perguntar quem.

---

## 🛠️ Capítulo 3: Ensinando a API a se despedir

O Fastify já sabe encerrar direito. O método `app.close()` faz exatamente o que queremos:
para de aceitar conexão nova e **espera** as requisições em andamento terminarem de responder.

O que falta é alguém chamá-lo quando o sinal chega. Esse alguém é o `src/server.ts`.

Substitua o arquivo inteiro por este:

```ts
/**
 * Server — Ponto de entrada da aplicação
 *
 * Este arquivo é responsável APENAS por iniciar o servidor HTTP na porta
 * configurada e por encerrá-lo direito quando pedirem. Toda a montagem do
 * Fastify (plugins e rotas) está em `app.ts`.
 *
 * Essa separação permite que, nos testes automatizados, importemos apenas o
 * `app.ts` sem precisar abrir uma porta de rede real.
 */

import type { FastifyInstance } from 'fastify'
import { buildApp } from './app.ts'
import { env } from './shared/env/index.ts'

/**
 * Tempo máximo que o encerramento pode levar antes de o processo desistir.
 *
 * **Este prazo não manda sozinho.** Quem orquestra o container tem o prazo dele,
 * e é o menor dos dois que decide: passou daquele, chega um `SIGKILL`, que não
 * pode ser tratado e não deixa log nenhum.
 *
 * Isso torna o número aqui metade de um acordo, e não uma configuração isolada.
 * A outra metade é o `-t` do `docker stop` — cujo padrão medimos em cerca de
 * 3,4 s no Docker 29.7.2, **menos** do que este prazo. Ou seja: com o
 * `docker stop` puro, quem decide continua sendo o `SIGKILL`.
 *
 * Ao mudar este número, ajuste também o prazo de quem sobe o container.
 */
const PRAZO_DE_DESLIGAMENTO_MS = 10_000

/**
 * Sinais do sistema operacional que significam "termine o que está fazendo".
 *
 *   • SIGTERM — o que o `docker stop` e todo orquestrador enviam no deploy.
 *   • SIGINT  — o `Ctrl+C` de quem está desenvolvendo.
 *
 * Os dois pedem a mesma coisa e recebem o mesmo tratamento. O que **não** dá
 * para tratar é o `SIGKILL`: ele não chega ao processo, é o sistema operacional
 * derrubando na marra. Por isso o encerramento limpo precisa caber no prazo.
 */
const SINAIS_DE_DESLIGAMENTO = ['SIGTERM', 'SIGINT'] as const

/**
 * Marca que um desligamento já começou.
 *
 * Sem isso, apertar `Ctrl+C` duas vezes com pressa dispararia dois encerramentos
 * ao mesmo tempo, e o segundo encontraria o servidor no meio do fechamento do
 * primeiro.
 */
let desligando = false

/**
 * Encerra a aplicação esperando as requisições em andamento terminarem.
 *
 * @param app   Instância do Fastify que está no ar.
 * @param sinal Qual sinal pediu o encerramento — registrado no log, porque saber
 *              se foi o deploy ou alguém no teclado muda a investigação.
 */
async function encerrar(app: FastifyInstance, sinal: string): Promise<void> {
  app.log.info({ sinal }, 'Sinal de desligamento recebido. Encerrando com calma.')

  // A rede de segurança do encerramento. Uma requisição travada — um banco que
  // não responde, uma chamada externa sem tempo limite — seguraria o
  // `app.close()` para sempre, e a API ficaria eternamente "quase morrendo".
  //
  // O `unref` diz ao Node que este relógio, sozinho, não é motivo para manter o
  // processo vivo: se tudo terminar antes, ele não atrasa a saída em nada.
  const prazoFinal = setTimeout(() => {
    app.log.error(
      { prazoMs: PRAZO_DE_DESLIGAMENTO_MS },
      'O encerramento passou do prazo. Saindo à força, com requisições em andamento.',
    )

    process.exit(1)
  }, PRAZO_DE_DESLIGAMENTO_MS)

  prazoFinal.unref()

  try {
    // É esta linha que faz a diferença toda. O `close` do Fastify para de aceitar
    // conexão nova e **espera** as requisições que já estão sendo processadas
    // terminarem de responder. Sem ela, o processo morre no meio da resposta e o
    // cidadão do outro lado recebe conexão cortada, sem erro e sem registro.
    await app.close()

    clearTimeout(prazoFinal)
    app.log.info('Servidor encerrado. Nenhuma requisição foi cortada.')

    process.exit(0)
  } catch (error) {
    app.log.error(error, 'Falha ao encerrar o servidor.')

    process.exit(1)
  }
}

/**
 * Liga os ouvintes de sinal do sistema operacional.
 *
 * Dentro de um container o Node roda como PID 1, e processo PID 1 no Linux não
 * tem ação padrão para sinal: sem estes ouvintes, o `SIGTERM` é simplesmente
 * ignorado, o Docker espera o prazo dele inteiro e mata com `SIGKILL`.
 *
 * @param app Instância do Fastify que deve ser encerrada quando o sinal chegar.
 */
function registrarDesligamentoGracioso(app: FastifyInstance): void {
  for (const sinal of SINAIS_DE_DESLIGAMENTO) {
    process.on(sinal, () => {
      if (desligando) {
        app.log.warn({ sinal }, 'Desligamento já em andamento. Sinal ignorado.')
        return
      }

      desligando = true

      // O ouvinte de sinal não pode ser `async`: o Node não espera pela promessa
      // dele. O `void` deixa explícito que a promessa segue por conta própria, e
      // que o `encerrar` trata os próprios erros.
      void encerrar(app, sinal)
    })
  }
}

/**
 * Sobe o servidor HTTP e o deixa ouvindo requisições.
 */
async function start(): Promise<void> {
  const app = buildApp()

  // Antes do `listen`, de propósito: se o sinal chegar durante a subida, já
  // existe quem o trate.
  registrarDesligamentoGracioso(app)

  try {
    // `env.PORT` já chega aqui como número e já foi validado na partida.
    // Não há nada a converter nem a conferir neste ponto.
    await app.listen({ port: env.PORT, host: env.HOST })

    // Usamos o logger do Fastify em vez de `console.log`. A diferença é que ele
    // grava em JSON estruturado: o primeiro parâmetro são os dados (que ficam
    // pesquisáveis nas ferramentas de monitoramento) e o segundo é a mensagem
    // para humanos lerem.
    app.log.info(
      { port: env.PORT, host: env.HOST, ambiente: env.NODE_ENV },
      'Servidor iniciado com sucesso',
    )
  } catch (error) {
    app.log.error(error)

    // Código de saída diferente de zero avisa o sistema operacional (e o Docker)
    // que o processo morreu por causa de um erro, e não porque terminou bem.
    process.exit(1)
  }
}

// O `try/catch` lá dentro cobre o `app.listen`, mas `buildApp()` acontece antes
// dele. Se a montagem da aplicação falhar, a promessa devolvida por `start()`
// seria rejeitada sem ninguém escutando: o processo morreria sem log e sem
// código de saída controlado. Este `.catch()` é a rede de segurança final.
start().catch((error: unknown) => {
  // Aqui não existe `app.log`: se chegamos neste ponto, o app pode nem ter sido
  // montado. Escrevemos direto na saída de erro do processo, que é o canal que
  // o sistema operacional e o Docker leem.
  process.stderr.write(`\n❌ A API não conseguiu iniciar.\n${String(error)}\n\n`)

  process.exit(1)
})
```

### O que cada pedaço resolve

**Os dois ouvintes.** `SIGTERM` vem do `docker stop` e de todo orquestrador; `SIGINT` é o
`Ctrl+C`. Os dois pedem a mesma coisa, então recebem o mesmo tratamento.

**A trava `desligando`.** Apertar `Ctrl+C` duas vezes com pressa dispararia dois
encerramentos simultâneos, e o segundo encontraria o servidor no meio do fechamento do
primeiro. Com a trava, o segundo sinal vira uma linha de aviso no log.

**O `void encerrar(...)`.** Um ouvinte de sinal não pode ser `async`: o Node não espera pela
promessa dele. O `void` deixa explícito que a promessa segue por conta própria — e é por isso
que a função `encerrar` trata os próprios erros, em vez de deixá-los escapar.

**O prazo final.** Se uma requisição travar — um banco que não responde, uma chamada externa
sem tempo limite —, o `app.close()` esperaria para sempre e a API ficaria eternamente "quase
morrendo". O relógio garante que, no pior caso, quem desiste é a própria API, com log do que
ficou pendente. O `unref()` diz ao Node que esse relógio, sozinho, não é motivo para manter o
processo vivo: se tudo terminar antes, ele não atrasa a saída em nada.

### Meça de novo

Reconstrua a imagem e repita exatamente a medição do Capítulo 1 — mas com **uma diferença no
comando de parada**, que o próximo capítulo explica:

```bash
docker build -t curso_api .
docker rm -f curso_api
docker run -d -p 3333:3333 --name curso_api curso_api
```

Dispare a requisição lenta e, com ela no ar, execute:

```bash
docker stop -t 30 curso_api
```

O resultado real:

```
CLIENTE: {"terminou":true}|HTTP=200|CURL=0 | duracao=5080ms
DOCKER STOP levou 4329ms
codigo de saida do container: 0
```

E o log do container, agora com a história inteira:

```
{"sinal":"SIGTERM","msg":"Sinal de desligamento recebido. Encerrando com calma."}
{"reqId":"req-2","msg":"Terminei o trabalho demorado."}
{"reqId":"req-2","res":{"statusCode":200},"responseTime":5005.88,"msg":"request completed"}
{"msg":"Servidor encerrado. Nenhuma requisição foi cortada."}
```

Repare na ordem: o sinal chegou **antes** de o trabalho terminar, e mesmo assim o trabalho
terminou. Foi isso que compramos. E o código de saída virou **0** — o container agora sai,
em vez de ser executado.

---

## ⏱️ Capítulo 4: O prazo é um acordo entre dois lados

Você acabou de digitar `docker stop -t 30` sem saber por quê. Este capítulo é o porquê, e ele
é a parte mais fácil de errar do assunto inteiro.

**O seu prazo não manda sozinho.** Existem dois relógios correndo:

| Relógio                                   | Quem define      | Valor neste projeto |
| :---------------------------------------- | :--------------- | :------------------ |
| Quanto a API espera antes de desistir     | `server.ts`      | 10 segundos         |
| Quanto o Docker espera antes do `SIGKILL` | `docker stop -t` | você escolhe        |

**Vale o menor dos dois.** Se o prazo de fora for menor, o `SIGKILL` chega antes de a API
terminar, e todo o código do capítulo anterior não serve para nada.

E aqui vem a parte que só a medição revela. A documentação do Docker diz que o padrão são
**10 segundos**. Medindo na máquina em que esta aula foi escrita (Docker 29.7.2), o padrão se
comportou como **cerca de 3,4 segundos**:

```
docker stop -t 1   ->  1397ms   exit=137
docker stop -t 10  -> 10376ms   exit=137
docker stop        ->  3391ms   exit=137
```

Confira na sua máquina — é um comando só. E, seja qual for o número que aparecer, a conclusão
é a mesma: **não confie no padrão, passe o `-t`.**

Para provar que isso importa, repita a última medição com o `docker stop` puro, sem `-t`. O
código está correto, o sinal é recebido, e mesmo assim:

```
CLIENTE: |HTTP=000|CURL=52 | duracao=4250ms
codigo de saida do container: 137

{"sinal":"SIGTERM","msg":"Sinal de desligamento recebido. Encerrando com calma."}
```

O log mostra a API **começando** a se despedir — e nunca terminando. O prazo de fora acabou
primeiro.

> [!CAUTION]
> Este é o erro mais caro deste assunto: implementar o desligamento gracioso, ver o log dizer
> "encerrando com calma" e concluir que está resolvido. O log de quem começa a se despedir é
> idêntico nos dois casos. **O que separa é o código de saída: 0 ou 137.**

### E quando é a API que estoura o prazo?

Falta ver o outro lado. Aumente temporariamente o `DEMORA_MS` do andaime para `20_000`,
reconstrua e derrube com prazo de sobra (`docker stop -t 60`). A requisição de 20 segundos não
cabe no prazo interno de 10:

```
CLIENTE: |HTTP=000|CURL=52 | duracao=11239ms

{"sinal":"SIGTERM","msg":"Sinal de desligamento recebido. Encerrando com calma."}
{"level":50,"prazoMs":10000,"msg":"O encerramento passou do prazo. Saindo à força, com requisições em andamento."}
```

Exatamente 10 segundos depois do sinal, a própria API desistiu — e **registrou isso**. É a
diferença que importa: a requisição foi cortada nos dois casos, mas aqui existe uma linha de
log em nível de erro dizendo o que aconteceu. No caso do `SIGKILL`, não existe linha nenhuma.

Volte o `DEMORA_MS` para `5_000` antes de seguir.

---

## 🕵️ Capítulo 5: Atrás de um proxy, a API não sabe com quem fala

Nenhuma API séria fica exposta direto na internet. Na frente dela existe um **proxy reverso**
— um programa que recebe todas as conexões e as repassa para dentro.

Isso cria um problema silencioso: do ponto de vista da API, **todas** as conexões passam a vir
do proxy. O IP de quem realmente chamou some.

E isso quebra duas coisas que já existem no projeto:

- o **limite de requisições** da Aula 09, que conta por IP — todos os clientes viram um só, e
  o primeiro a estourar bloqueia os demais;
- o **log**, que passa a registrar sempre o mesmo endereço, tornando qualquer investigação
  inútil.

### Monte a cena

Vamos colocar um proxy de verdade na frente. Primeiro, uma rede para os dois containers
conversarem, e um arquivo de configuração para o proxy:

```bash
docker network create rede-aula11
```

Crie um arquivo `nginx.conf` **fora do projeto** (ele é descartável — pode ser na sua área de
trabalho):

```nginx
events {}
http {
  server {
    listen 8080;
    location / {
      proxy_pass http://api:3333;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

A linha que interessa é a do `X-Forwarded-For`. Como o proxy sabe que o IP de origem vai se
perder, ele **escreve o endereço original em um cabeçalho** antes de repassar. Essa é a
convenção que a web inteira usa.

Suba os dois containers:

```bash
docker rm -f api proxy
docker run -d --name api --network rede-aula11 curso_api
docker run -d --name proxy --network rede-aula11 -p 8080:8080 -v "/caminho/para/nginx.conf:/etc/nginx/nginx.conf:ro" nginx:alpine
```

E chame a API **através do proxy**, na porta 8080:

```bash
curl http://127.0.0.1:8080/health
```

Agora olhe o log da API e o IP do proxy, lado a lado:

```bash
docker logs api
docker inspect proxy --format "{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}"
```

O resultado medido:

```
IP do container proxy: 172.18.0.3

{"req":{"method":"GET","url":"/health","remoteAddress":"172.18.0.3"},"msg":"incoming request"}
```

São o mesmo endereço. A API registrou o **proxy** como se fosse o cliente.

### A correção

O Fastify sabe ler o `X-Forwarded-For` — ele só não faz isso por conta própria, e com toda a
razão: acreditar em cabeçalho é uma decisão, não um detalhe. A opção chama-se `trustProxy`, e
nós a expomos como variável de ambiente.

No `src/shared/env/env.schema.ts`, acrescente a variável `TRUST_PROXY` (o arquivo completo
está no Capítulo 9). E no `src/app.ts`, passe o valor ao Fastify:

```ts
  const app = Fastify({
    logger: (options.logger ?? true) ? buildLoggerOptions() : false,

    trustProxy: env.TRUST_PROXY,
  })
```

Refaça a cena, agora com a variável ligada em **1**:

```bash
docker rm -f api
docker run -d --name api --network rede-aula11 -e TRUST_PROXY=1 curso_api
curl http://127.0.0.1:8080/health
docker logs api
```

```
{"req":{"method":"GET","url":"/health","remoteAddress":"172.18.0.1"},"msg":"incoming request"}
```

`172.18.0.1` é o endereço de quem realmente chamou — e não mais o do proxy. O limite de
requisições volta a contar por cliente, e o log volta a servir para investigar.

---

## ⚠️ Capítulo 6: Por que o padrão é desconfiar

Você deve estar se perguntando por que essa opção não vem ligada de fábrica, já que corrige
um problema tão claro. A resposta é o capítulo inteiro.

**Qualquer pessoa pode enviar um `X-Forwarded-For`.** É um cabeçalho comum de HTTP: não tem
assinatura, não tem prova, não tem nada. Ligar o `trustProxy` significa dizer à API "acredite
no que vier escrito ali" — e isso só é seguro quando existe um proxy de verdade na frente
**sobrescrevendo** o cabeçalho.

Sem proxy na frente, o cliente escolhe a própria identidade. Veja a medição, com o mesmo
cabeçalho forjado (`X-Forwarded-For: 9.9.9.9, 8.8.8.8`) enviado direto à API:

| `TRUST_PROXY` | O IP em que a API acredita | Por quê                                   |
| :------------ | :------------------------- | :---------------------------------------- |
| `false`       | `172.17.0.1`               | Ignora o cabeçalho; usa a conexão real    |
| `1`           | `8.8.8.8`                  | Confia em **um** salto: pega o último     |
| `true`        | `9.9.9.9`                  | Confia na cadeia inteira: pega o primeiro |

Repare que o número de saltos **não é uma proteção contra a mentira** — ele só limita quanto
da cadeia forjada a API leva a sério. O que protege é existir um proxy real que reescreve o
cabeçalho antes de a requisição chegar.

### O estrago, medido

A rota `/health/ready` aceita 100 requisições por minuto por IP. Disparamos 120 nas duas
configurações, com e sem cabeçalho forjado, trocando o IP a cada requisição:

```
--- TRUST_PROXY=false ---
sem forjar: {"200":100,"429":20}
forjando:   {"200":100,"429":20}

--- TRUST_PROXY=true ---
sem forjar: {"200":100,"429":20}
forjando:   {"200":120}
```

A última linha é o resumo do capítulo: **120 requisições, nenhuma bloqueada.** Com o
`trustProxy` ligado e nenhum proxy de verdade na frente, o limite de requisições da Aula 09
deixou de existir — bastou trocar de nome a cada chamada.

> [!CAUTION]
> `TRUST_PROXY=true` sem proxy na frente é **pior** do que não configurar nada. Não é uma
> proteção parcial: é a remoção de uma proteção que funcionava.

A regra prática, então:

| Situação                                | Valor                |
| :-------------------------------------- | :------------------- |
| Desenvolvendo na sua máquina            | `false`              |
| Um proxy reverso na frente da API       | `1`                  |
| Dois (por exemplo, CDN e proxy próprio) | `2`                  |
| Não sei quantos                         | Descubra. Não chute. |

---

## 📝 Capítulo 7: O log que você não pode mostrar para ninguém

Faça um teste, com a API rodando em desenvolvimento:

```bash
npm run dev
curl "http://127.0.0.1:3333/health?token=segredo123&pagina=2"
```

Antes desta aula, o terminal registrava isto:

```
"url": "/health?token=segredo123&pagina=2"
```

O token está lá, por extenso. E log não é uma janela: é um **arquivo**, que vai para a
ferramenta de monitoramento, que é copiada no backup, e que muita gente tem permissão de ler
justamente porque log serve para investigar. Um segredo que entra ali não sai mais.

Vamos resolver três coisas de uma vez.

### Crie `src/shared/logger/index.ts`

```ts
/**
 * Configuração do registro de eventos (logger)
 *
 * O Fastify já vem com o Pino ligado. O que faltava era decidir **três** coisas
 * que a configuração padrão deixa em aberto e que só doem depois do deploy:
 *
 *   1. QUANTO registrar — o volume útil em desenvolvimento é ruído em produção.
 *   2. O QUE NUNCA registrar — credencial que entra no log fica gravada para
 *      sempre, no lugar em que mais gente tem acesso.
 *   3. COMO apresentar — máquina lê JSON melhor; gente lê texto melhor.
 *
 * O log é o único lugar em que dá para investigar o que aconteceu depois que
 * aconteceu. Um log que não pode ser aberto por questão de sigilo é um log que
 * não serve para investigar nada.
 */

import type { FastifyRequest, FastifyServerOptions } from 'fastify'
import { env } from '../env/index.ts'

/**
 * Texto que substitui um valor sigiloso no log.
 *
 * É o mesmo que o Pino usa por padrão. Mantemos igual de propósito: quem lê o
 * arquivo encontra sempre a mesma marca, venha ela do `redact` do Pino ou do
 * nosso mascaramento de URL.
 */
export const VALOR_OCULTADO = '[Redacted]'

/**
 * Nomes de parâmetro de query string cujo valor nunca pode ser registrado.
 *
 * Nenhum deles **deveria** trafegar na URL — segredo se manda no corpo ou em
 * cabeçalho. Mas é engano comum, e o custo de um `?token=` gravado é alto demais
 * para depender de ninguém errar.
 *
 * A comparação é feita em minúsculas, então basta listar cada nome uma vez.
 */
export const PARAMETROS_SENSIVEIS = [
  'token',
  'access_token',
  'refresh_token',
  'senha',
  'password',
  'secret',
  'api_key',
  'apikey',
  'authorization',
  'cpf',
]

/**
 * Nível de log padrão de cada ambiente.
 *
 * Em desenvolvimento queremos ver tudo, inclusive o que o Fastify faz por baixo.
 * Em produção, `debug` seria um custo real: log é disco, é rede e é dinheiro, e
 * o volume atrapalha justamente na hora de procurar a linha que importa.
 *
 * Em teste o registro fica **desligado**: sem isso, cada requisição simulada
 * imprimiria várias linhas e o resultado dos testes ficaria ilegível.
 */
export const NIVEL_DE_LOG_POR_AMBIENTE = {
  development: 'debug',
  test: 'silent',
  production: 'info',
} as const

/**
 * Substitui o valor dos parâmetros sensíveis de uma URL pela marca de ocultado.
 *
 * Existe porque o `redact` do Pino **não** resolve este caso: ele age sobre
 * campos de um objeto, e a query string é um pedaço de texto dentro de
 * `req.url`. Sem esta função, um `?token=abc123` seria gravado por extenso.
 *
 * @param url  Caminho com ou sem query string, como o Fastify o entrega.
 * @returns    A mesma URL, com o valor dos parâmetros sensíveis ocultado.
 */
export function mascararUrl(url: string): string {
  const inicioDaQuery = url.indexOf('?')

  if (inicioDaQuery === -1) {
    return url
  }

  const caminho = url.slice(0, inicioDaQuery)

  // Percorremos a query string na mão, em vez de usar o `URLSearchParams`. O
  // motivo apareceu na execução do teste: ao remontar o texto, ele reescreve o
  // valor no formato de URL, e a marca `[Redacted]` virava `%5BRedacted%5D` — o
  // log ficava difícil de ler justamente na linha que mais importa. Fatiando o
  // texto, o que não é sigiloso chega ao log exatamente como veio.
  const parametros = url
    .slice(inicioDaQuery + 1)
    .split('&')
    .map((parametro) => {
      const separador = parametro.indexOf('=')
      const chave = separador === -1 ? parametro : parametro.slice(0, separador)

      if (!PARAMETROS_SENSIVEIS.includes(decodificar(chave).toLowerCase())) {
        return parametro
      }

      return `${chave}=${VALOR_OCULTADO}`
    })

  return `${caminho}?${parametros.join('&')}`
}

/**
 * Decodifica um pedaço de URL sem deixar a aplicação cair por causa dele.
 *
 * `decodeURIComponent` lança exceção diante de texto malformado — um `%zz`
 * digitado por engano, ou enviado de propósito. Uma exceção aqui aconteceria
 * **dentro do serializador do log**, no meio de uma requisição legítima.
 *
 * @param texto Pedaço de URL, possivelmente codificado.
 * @returns     O texto decodificado, ou o original quando não for decodificável.
 */
function decodificar(texto: string): string {
  try {
    return decodeURIComponent(texto)
  } catch {
    return texto
  }
}

/**
 * Monta a configuração do logger conforme o ambiente em que a API está rodando.
 *
 * @returns Configuração pronta para o campo `logger` do Fastify.
 */
export function buildLoggerOptions(): FastifyServerOptions['logger'] {
  const emDesenvolvimento = env.NODE_ENV === 'development'

  return {
    // A variável de ambiente ganha da regra por ambiente. É o que permite subir
    // o volume de um servidor específico, durante uma investigação, sem alterar
    // código nem publicar versão nova.
    level: env.LOG_LEVEL ?? NIVEL_DE_LOG_POR_AMBIENTE[env.NODE_ENV],

    // Ocultação por caminho de campo. Vale para qualquer objeto que alguém
    // registre no log, hoje ou daqui a um ano — inclusive em código que ainda
    // não foi escrito. É a diferença entre proteger o dado e confiar que todo
    // mundo vai lembrar de não registrá-lo.
    redact: {
      paths: [
        'headers.authorization',
        'headers.cookie',
        'req.headers.authorization',
        'req.headers.cookie',
        'request.headers.authorization',
        'request.headers.cookie',
      ],
      censor: VALOR_OCULTADO,
    },

    serializers: {
      /**
       * Decide quais campos da requisição entram no log.
       *
       * Repare no que **não** está aqui: os cabeçalhos. Registrar todos seria
       * gravar `authorization` e `cookie` em toda linha — e a lista de `redact`
       * acima existe para o caso de alguém registrá-los de propósito em outro
       * ponto do código, não para consertar esta decisão.
       */
      req(request: FastifyRequest) {
        return {
          method: request.method,
          url: mascararUrl(request.url),
          host: request.host,
          // `request.ip` é o endereço em que a API acredita. Com `TRUST_PROXY`
          // desligado atrás de um proxy, este campo registra o IP do proxy, e a
          // investigação vai atrás da máquina errada.
          remoteAddress: request.ip,
          remotePort: request.socket.remotePort,
        }
      },
    },

    // O Pino escreve JSON de uma linha só, que é o formato que as ferramentas de
    // monitoramento indexam. Para quem está com o terminal aberto, isso é ruim
    // de ler — então, e **somente** em desenvolvimento, passamos a saída pelo
    // `pino-pretty`, que reformata em texto alinhado.
    //
    // Ele é dependência de desenvolvimento e não existe dentro da imagem Docker,
    // construída com `--omit=dev`. É justamente por isso que esta linha só pode
    // ser produzida quando o ambiente é `development`.
    ...(emDesenvolvimento
      ? {
          transport: {
            target: 'pino-pretty',
            options: {
              translateTime: 'HH:MM:ss',
              ignore: 'pid,hostname',
            },
          },
        }
      : {}),
  }
}
```

### As três decisões, uma a uma

**1. O volume muda com o ambiente.** Em desenvolvimento queremos ver tudo; em produção,
`debug` é custo real — log é disco, é rede e é dinheiro, e o excesso atrapalha justamente na
hora de procurar a linha que importa. Nos testes o registro fica desligado, senão cada
requisição simulada imprimiria várias linhas.

A variável `LOG_LEVEL` existe para o caso específico de precisar subir o volume de um servidor
durante uma investigação, **sem** alterar código e sem publicar versão nova.

**2. O `redact` protege campos.** É uma lista de caminhos que o Pino substitui por
`[Redacted]` em qualquer objeto registrado. Vale para código que ainda nem foi escrito: no dia
em que alguém registrar `{ headers }` para depurar, o `authorization` já sai protegido.

> [!NOTE]
> O serializador que escrevemos **não** registra cabeçalho nenhum, então hoje o `redact` não
> tem o que fazer nas requisições normais. Ele não é decoração: é a rede embaixo do trapézio,
> para o dia em que alguém registrar um objeto que os contenha.

**3. O serializador cuida da URL — e é ele que resolve o nosso caso.** O `redact` age sobre
**campos de objeto**, e a query string é um pedaço de **texto** dentro de `req.url`. Nenhum
caminho de `redact` alcança lá dentro. Por isso a função `mascararUrl` existe.

Rode de novo agora:

```
[14:26:52] INFO: incoming request
    reqId: "req-1"
    req: {
      "method": "GET",
      "url": "/health?token=[Redacted]&pagina=2",
      "host": "127.0.0.1:3333",
      "remoteAddress": "127.0.0.1",
      "remotePort": 57907
    }
```

O `pagina=2` continua lá — perder os parâmetros comuns tornaria o log inútil para o que ele
serve. Só o valor sigiloso sumiu.

### O log legível em desenvolvimento

Repare que a saída acima está colorida e alinhada, e não em JSON de uma linha. Isso é o
`pino-pretty`, que instalamos como dependência de **desenvolvimento**:

```bash
npm install --save-dev pino-pretty
```

Em produção o log continua sendo JSON de uma linha — é o formato que as ferramentas de
monitoramento indexam. O `pino-pretty` nem existe dentro da imagem Docker, que é construída
com `--omit=dev`; é exatamente por isso que aquele trecho da configuração só é produzido
quando o ambiente é `development`.

### Ligue o logger no `src/app.ts`

O `buildApp` passa a montar a configuração em vez de só ligar e desligar:

```ts
    logger: (options.logger ?? true) ? buildLoggerOptions() : false,
```

O arquivo completo está no Capítulo 9.

### E teste, porque isto é regra de negócio

Crie `src/shared/logger/logger.spec.ts`:

```ts
/**
 * Testes da configuração do registro de eventos
 *
 * O log é o único lugar em que dá para investigar o que aconteceu depois que
 * aconteceu. Um segredo que vaza para cá não volta: fica gravado no arquivo, na
 * ferramenta de monitoramento e em toda cópia de backup que já foi feita.
 *
 * Por isso o mascaramento é testado como regra de negócio, e não como detalhe.
 */

import { describe, expect, it } from 'vitest'
import { mascararUrl, VALOR_OCULTADO } from './index.ts'

describe('mascararUrl — o que nunca pode ir para o log', () => {
  it('oculta o valor de um token na query string', () => {
    // O caso que motivou tudo: um token enviado por engano na URL. Sem o
    // mascaramento, ele fica gravado por extenso, para sempre.
    expect(mascararUrl('/health?token=abc123')).toBe(`/health?token=${VALOR_OCULTADO}`)
  })

  it('oculta independentemente de maiúsculas e minúsculas', () => {
    expect(mascararUrl('/health?TOKEN=abc123')).toBe(`/health?TOKEN=${VALOR_OCULTADO}`)
  })

  it('oculta cada parâmetro sensível e preserva os demais', () => {
    // Só o que é sigiloso some. Perder os parâmetros comuns tornaria o log
    // inútil justamente para o que ele serve: entender o que foi pedido.
    expect(mascararUrl('/busca?pagina=2&senha=1234&ordem=nome')).toBe(
      `/busca?pagina=2&senha=${VALOR_OCULTADO}&ordem=nome`,
    )
  })

  it('oculta o CPF, que é dado pessoal e não deve ficar gravado', () => {
    expect(mascararUrl('/protocolos?cpf=12345678900')).toBe(`/protocolos?cpf=${VALOR_OCULTADO}`)
  })

  it('oculta todas as repetições do mesmo parâmetro', () => {
    expect(mascararUrl('/health?token=um&token=dois')).toBe(
      `/health?token=${VALOR_OCULTADO}&token=${VALOR_OCULTADO}`,
    )
  })

  it('não deixa a marca de ocultado ser reescrita em código de URL', () => {
    // Este teste nasceu de uma falha real: a primeira versão remontava a query
    // string com `URLSearchParams`, e a marca `[Redacted]` chegava ao log como
    // `%5BRedacted%5D`.
    expect(mascararUrl('/health?token=abc')).not.toContain('%5B')
  })

  it('não quebra diante de uma URL malformada', () => {
    // `%zz` não é código de URL válido. Uma exceção aqui aconteceria dentro do
    // serializador do log, no meio de uma requisição legítima.
    expect(() => mascararUrl('/busca?%zz=1&token=abc')).not.toThrow()
    expect(mascararUrl('/busca?%zz=1&token=abc')).toContain(VALOR_OCULTADO)
  })

  it('oculta o parâmetro que vem sem valor', () => {
    expect(mascararUrl('/health?token')).toBe(`/health?token=${VALOR_OCULTADO}`)
  })
})

describe('mascararUrl — o que precisa continuar intacto', () => {
  it('devolve a URL sem alteração quando não há query string', () => {
    expect(mascararUrl('/health')).toBe('/health')
    expect(mascararUrl('/health/ready')).toBe('/health/ready')
  })

  it('não inventa query string em URL que não tem', () => {
    // Devolver "/health?" mudaria a linha do log de toda requisição do
    // monitoramento, que é a mais frequente da API.
    expect(mascararUrl('/health')).not.toContain('?')
  })

  it('preserva uma query string sem nada sigiloso', () => {
    expect(mascararUrl('/busca?pagina=2&ordem=nome')).toBe('/busca?pagina=2&ordem=nome')
  })
})
```

Dois desses testes nasceram de falhas reais, encontradas ao executar:

- a primeira versão remontava a query string com `URLSearchParams`, e a marca `[Redacted]`
  chegava ao log como `%5BRedacted%5D` — ilegível justamente na linha mais importante;
- `decodeURIComponent` lança exceção diante de texto malformado como `%zz`, e essa exceção
  aconteceria **dentro do serializador do log**, no meio de uma requisição legítima.

```bash
npm test
```

---

## 🧹 Capítulo 8: desmontando o andaime

O `src/exemplo-demorado.ts` cumpriu o papel dele. Apague-o:

```bash
# PowerShell
Remove-Item src/exemplo-demorado.ts

# Linux / Mac
rm src/exemplo-demorado.ts
```

E tire do `src/app.ts` as duas partes que o referenciam: a linha do `import` e o
`app.register(exemploDemoradoRoutes)` com o comentário acima dele.

Este é o `src/app.ts` final, o estado em que o seu arquivo precisa ficar:

```ts
/**
 * App — Montagem da instância do Fastify
 *
 * Este arquivo é responsável por:
 *   1. Criar a instância do Fastify com suas configurações.
 *   2. Registrar o tratamento centralizado de erros.
 *   3. Registrar as proteções de HTTP.
 *   4. Registrar a documentação da API, quando ela deve existir.
 *   5. Registrar as rotas de cada módulo.
 *
 * A ordem em que essas coisas acontecem não é decorativa: erros, proteções e
 * documentação precisam estar de pé ANTES das rotas para valerem para elas.
 *
 * Separamos a montagem do app (aqui) da inicialização do servidor (`server.ts`)
 * para facilitar os testes automatizados: nos testes importamos apenas o app,
 * sem precisar ocupar uma porta de rede.
 */

import Fastify from 'fastify'
import type { FastifyInstance } from 'fastify'
import { serializerCompiler, validatorCompiler } from 'fastify-type-provider-zod'
import { healthRoutes } from './modules/health/health.routes.ts'
import { registerDocs } from './shared/docs/index.ts'
import { env } from './shared/env/index.ts'
import { errorHandler, notFoundHandler } from './shared/errors/error-handler.ts'
import { buildLoggerOptions } from './shared/logger/index.ts'
import { registerSecurity } from './shared/security/index.ts'
import { configurarMensagensEmPortugues } from './shared/validation/zod-locale.ts'

/**
 * Opções de montagem da aplicação.
 */
export interface BuildAppOptions {
  /**
   * Liga ou desliga o registro de eventos (logger).
   *
   * O padrão é ligado, com a configuração de `src/shared/logger/`. Nos testes
   * automatizados passamos `false`: sem isso, cada requisição simulada
   * imprimiria várias linhas de log e o resultado dos testes ficaria impossível
   * de ler.
   */
  logger?: boolean

  /**
   * Liga ou desliga a documentação da API.
   *
   * O padrão é ligado **fora de produção**. A documentação publica o mapa
   * completo da API — toda rota, todo campo, toda regra de validação —, e hoje
   * esse mapa não tem para quem servir em produção: as únicas rotas são as de
   * saúde. Enquanto for assim, o mapa só interessa a quem está estudando o
   * serviço de fora.
   *
   * A decisão tem gatilho para ser revista: quando existirem rotas de negócio e
   * autenticação, a documentação volta a subir em produção — protegida por
   * login, e não desligada.
   */
  docs?: boolean
}

/**
 * Fábrica da aplicação Fastify.
 *
 * @param options Ajustes opcionais de montagem.
 * @returns Instância do Fastify já configurada, pronta para receber requisições
 *          ou para ser usada em testes.
 */
export function buildApp(options: BuildAppOptions = {}): FastifyInstance {
  const app = Fastify({
    // O Fastify já vem com o Pino, um dos loggers mais rápidos do Node. O que a
    // função `buildLoggerOptions` acrescenta é a parte que o padrão deixa em
    // aberto: quanto registrar em cada ambiente e o que nunca registrar.
    logger: (options.logger ?? true) ? buildLoggerOptions() : false,

    // Declara em quantos proxies confiar para descobrir quem chamou. Com `false`,
    // que é o padrão, `request.ip` é o endereço da conexão — correto quando não
    // há proxy na frente, e errado quando há.
    //
    // Isto vale para tudo que depende de saber quem é o cliente: o limite de
    // requisições, a linha do log e, no futuro, a trilha de auditoria.
    trustProxy: env.TRUST_PROXY,
  })

  // Toda mensagem de validação sai em português, inclusive as que o Zod gera
  // sozinho ao recusar uma requisição.
  configurarMensagensEmPortugues()

  // O Fastify entende schema escrito em JSON Schema. Estas duas linhas ensinam
  // a ele a ler os nossos, que são escritos em Zod — a mesma ferramenta que já
  // valida as variáveis de ambiente.
  //
  // São dois compiladores porque são dois trabalhos opostos:
  //   • o VALIDADOR cuida do que ENTRA: recusa a requisição que não cumpre o
  //     schema, antes de o handler rodar;
  //   • o SERIALIZADOR cuida do que SAI: monta a resposta usando apenas os
  //     campos declarados, descartando qualquer outro.
  app.setValidatorCompiler(validatorCompiler)
  app.setSerializerCompiler(serializerCompiler)

  // Registramos o tratamento de erros ANTES das rotas, de propósito: assim toda
  // rota registrada daqui para baixo já nasce coberta, e ninguém precisa lembrar
  // de tratar erro em cada rota nova.
  app.setErrorHandler(errorHandler)
  app.setNotFoundHandler(notFoundHandler)

  // As proteções de HTTP vêm antes de tudo o que responde: assim toda rota, e
  // também toda resposta de erro, já nasce com os cabeçalhos e sujeita ao limite
  // de requisições.
  registerSecurity(app)

  // A documentação também vem ANTES das rotas, mas por outro motivo: ela é
  // montada escutando cada rota que for registrada. Rota que entrar antes dela
  // não aparece na página — e nada avisa, porque a página abre do mesmo jeito.
  if (options.docs ?? env.NODE_ENV !== 'production') {
    registerDocs(app)
  }

  // No Fastify, cada conjunto de rotas é registrado como um plugin. Isso mantém
  // cada módulo isolado: um erro no registro de um não derruba os outros.
  app.register(healthRoutes)

  return app
}
```

Confirme que não sobrou nada:

```bash
npm run check
```

Se o TypeScript reclamar de `exemploDemoradoRoutes`, sobrou uma das duas referências.

---

## 📄 Capítulo 9: os arquivos que mudaram junto

### `src/shared/env/env.schema.ts`

Duas variáveis novas: `TRUST_PROXY` e `LOG_LEVEL`.

```ts
/**
 * Schema das variáveis de ambiente
 *
 * Este arquivo é o **contrato** da configuração da aplicação: ele declara quais
 * variáveis existem, de que tipo cada uma é e qual o valor padrão quando ela não
 * for informada.
 *
 * Manter o contrato separado da validação (`index.ts`) permite reutilizá-lo nos
 * testes, onde queremos validar um objeto montado à mão em vez do ambiente real.
 */

import { z } from 'zod'

/**
 * Contrato das variáveis de ambiente que esta API precisa para funcionar.
 *
 * As mensagens de erro estão em português porque quem vai lê-las é a nossa
 * equipe, às vezes às pressas, durante um deploy que não subiu.
 */
export const envSchema = z.object({
  /**
   * Em qual ambiente a aplicação está rodando.
   *
   * Restringimos a três valores conhecidos de propósito: um `NODE_ENV=produção`
   * digitado com acento passaria despercebido se aceitássemos texto livre, e a
   * aplicação se comportaria como se estivesse em desenvolvimento no servidor real.
   */
  NODE_ENV: z
    .enum(['development', 'test', 'production'], {
      error: 'deve ser "development", "test" ou "production"',
    })
    .default('development'),

  /**
   * Porta TCP em que o servidor vai escutar.
   *
   * Toda variável de ambiente chega como texto. O `coerce` converte para número
   * antes de validar — e é justamente essa conversão que denuncia um `PORT=abc`,
   * que viraria `NaN` e passaria despercebido sem o schema.
   */
  PORT: z.coerce
    .number({ error: 'deve ser um número inteiro' })
    .int({ error: 'deve ser um número inteiro, sem casas decimais' })
    .min(1, { error: 'deve estar entre 1 e 65535' })
    .max(65535, { error: 'deve estar entre 1 e 65535' })
    .default(3333),

  /**
   * Endereço de rede em que o servidor vai escutar.
   *
   * O padrão `0.0.0.0` significa "todas as interfaces desta máquina", necessário
   * para que a API responda de fora quando estiver dentro de um container.
   *
   * Repare que existem DUAS mensagens: uma no `z.string()` e outra no `.min(1)`.
   * Elas cobrem situações diferentes — a primeira vale quando o valor está
   * ausente ou é de outro tipo, a segunda quando ele veio, mas veio vazio.
   * Sem a primeira, esse caso cairia na mensagem padrão do Zod, em inglês.
   */
  HOST: z
    .string({ error: 'é obrigatória e deve ser um texto' })
    .min(1, { error: 'não pode ficar vazio' })
    .default('0.0.0.0'),

  /**
   * Endereços de sites autorizados a chamar esta API pelo navegador.
   *
   * Vem como uma lista separada por vírgula e sai como array. É uma das poucas
   * configurações que **precisa** ser variável de ambiente de verdade: a lista é
   * genuinamente diferente em cada ambiente — a máquina de quem desenvolve, a
   * homologação, o domínio real — e não há como deduzi-la do `NODE_ENV`.
   *
   * O padrão é **vazio**, ou seja, nenhum site de outra origem autorizado. Um
   * padrão permissivo aqui é o tipo de coisa que nasce "por enquanto" e fica para
   * sempre, porque apertar quebra alguém e afrouxar nunca quebra nada.
   */
  CORS_ORIGINS: z
    .string({ error: 'é obrigatória e deve ser um texto' })
    .default('')
    .transform((valor) =>
      valor
        .split(',')
        .map((origem) => origem.trim())
        // Descarta pedaços vazios: "a,,b" e "a, b," são erros de digitação
        // comuns, e uma origem vazia autorizaria coisa nenhuma de forma confusa.
        .filter((origem) => origem.length > 0),
    ),

  /**
   * Em quantos proxies confiar para descobrir o IP de quem chamou.
   *
   * Atrás de um proxy, o endereço da conexão é o do proxy. O IP real do cliente
   * viaja no cabeçalho `X-Forwarded-For`, e é isso que esta variável libera a
   * API a ler.
   *
   * **Não é uma proteção — é uma declaração de confiança**, e por isso o padrão
   * é `false`. Qualquer pessoa pode enviar um `X-Forwarded-For` inventado: com
   * esta opção ligada sem um proxy de verdade na frente, o cliente troca de
   * identidade a cada requisição e passa por cima do limite de requisições.
   *
   * Valores aceitos:
   *   • `false` — não confiar em ninguém (padrão; correto sem proxy na frente)
   *   • `true`  — confiar na cadeia inteira do cabeçalho
   *   • `1`, `2`, ... — confiar apenas nos N proxies mais próximos da API
   *
   * O número é o valor recomendado em produção: com um proxy na frente,
   * `TRUST_PROXY=1` faz a API ler **só** o salto que o próprio proxy escreveu, e
   * ignorar o que o cliente tenha inventado antes dele.
   */
  TRUST_PROXY: z
    .string({ error: 'é obrigatória e deve ser um texto' })
    .default('false')
    .refine((valor) => valor === 'true' || valor === 'false' || /^[1-9]\d*$/.test(valor), {
      error: 'deve ser "true", "false" ou a quantidade de proxies confiáveis (1 ou mais)',
    })
    .transform((valor): boolean | number => {
      if (valor === 'true') return true
      if (valor === 'false') return false

      return Number(valor)
    }),

  /**
   * Volume de detalhe do registro de eventos.
   *
   * Fica **opcional** de propósito: quando não vem, cada ambiente usa o nível que
   * faz sentido para ele (a tabela está em `src/shared/logger/index.ts`).
   * Declará-la aqui serve para o caso específico de precisar subir o volume de um
   * servidor durante uma investigação — sem alterar código e sem publicar versão.
   */
  LOG_LEVEL: z
    .enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace', 'silent'], {
      error: 'deve ser "fatal", "error", "warn", "info", "debug", "trace" ou "silent"',
    })
    .optional(),
})

/**
 * Tipo derivado automaticamente do schema acima.
 *
 * Escrevemos o contrato uma única vez e o TypeScript deduz o tipo a partir dele.
 * Assim é impossível o tipo e a validação discordarem: adicionar uma variável no
 * schema já a torna conhecida em todo o projeto.
 */
export type Env = z.infer<typeof envSchema>
```

Repare no `TRUST_PROXY`: são três etapas encadeadas — `default`, `refine` e `transform`. A
`refine` recusa qualquer texto que não seja `true`, `false` ou um número inteiro positivo; a
`transform` converte para o tipo certo. Essa ordem importa: sem a `refine`, um
`TRUST_PROXY=sim` viraria `NaN` silenciosamente.

E repare no que **não** é aceito: `0`. Confiar em "zero proxies" é o mesmo que `false`,
escrito de um jeito que ninguém entende ao reler o arquivo seis meses depois.

### `src/shared/env/env.schema.spec.ts`

```ts
/**
 * Testes do contrato de variáveis de ambiente
 *
 * Testamos o `envSchema` diretamente, entregando objetos montados à mão em vez
 * de depender do ambiente real da máquina. É por isso que o contrato mora em um
 * arquivo separado da validação: aqui podemos experimentar qualquer combinação
 * de valores sem alterar o `.env` de ninguém.
 */

import { describe, expect, it } from 'vitest'
import { envSchema } from './env.schema.ts'

/**
 * Devolve a mensagem de erro da variável indicada, ou `undefined` se ela passou.
 */
function mensagemDe(entrada: Record<string, unknown>, variavel: string): string | undefined {
  const resultado = envSchema.safeParse(entrada)

  if (resultado.success) return undefined

  return resultado.error.issues.find((problema) => problema.path[0] === variavel)?.message
}

describe('envSchema — valores válidos', () => {
  it('aceita uma configuração completa e correta', () => {
    const resultado = envSchema.safeParse({
      NODE_ENV: 'production',
      PORT: '8080',
      HOST: '127.0.0.1',
    })

    expect(resultado.success).toBe(true)
  })

  it('converte a porta de texto para número', () => {
    const resultado = envSchema.parse({ PORT: '8080' })

    // Toda variável de ambiente chega como texto. Depois da validação, ela
    // precisa ser um número de verdade — não o texto "8080".
    expect(resultado.PORT).toBe(8080)
    expect(typeof resultado.PORT).toBe('number')
  })

  it('usa os valores padrão quando nada é informado', () => {
    const resultado = envSchema.parse({})

    expect(resultado.NODE_ENV).toBe('development')
    expect(resultado.PORT).toBe(3333)
    expect(resultado.HOST).toBe('0.0.0.0')
  })
})

describe('envSchema — valores inválidos', () => {
  it('recusa uma porta que não é número', () => {
    // O caso real que motivou esta validação: "8O80", com a letra O no lugar do
    // zero. Passa despercebido na leitura e derruba a API na partida.
    expect(envSchema.safeParse({ PORT: '8O80' }).success).toBe(false)
  })

  it('recusa uma porta fora da faixa permitida', () => {
    expect(envSchema.safeParse({ PORT: '0' }).success).toBe(false)
    expect(envSchema.safeParse({ PORT: '99999' }).success).toBe(false)
  })

  it('recusa uma porta com casas decimais', () => {
    expect(envSchema.safeParse({ PORT: '80.5' }).success).toBe(false)
  })

  it('recusa um ambiente fora da lista conhecida', () => {
    // "producao", em português, é o engano mais provável — e o mais perigoso,
    // porque faria a aplicação se comportar como se estivesse em desenvolvimento
    // dentro do servidor de produção.
    expect(envSchema.safeParse({ NODE_ENV: 'producao' }).success).toBe(false)
  })

  it('recusa um endereço vazio', () => {
    expect(envSchema.safeParse({ HOST: '' }).success).toBe(false)
  })
})

describe('envSchema — origens autorizadas (CORS)', () => {
  /**
   * A variável chega como um texto separado por vírgula e precisa sair como
   * lista. Testamos a transformação, e não apenas a validação: é ela que decide
   * quem pode chamar a API pelo navegador, e um erro aqui abre ou fecha portas
   * em silêncio.
   */

  it('não autoriza ninguém quando a variável não é informada', () => {
    const resultado = envSchema.parse({})

    // O padrão precisa ser o fechado. Configuração de segurança que nasce aberta
    // "por enquanto" costuma ficar aberta para sempre.
    expect(resultado.CORS_ORIGINS).toEqual([])
  })

  it('transforma a lista separada por vírgula em array', () => {
    const resultado = envSchema.parse({
      CORS_ORIGINS: 'http://localhost:5173,https://portal.exemplo.gov.br',
    })

    expect(resultado.CORS_ORIGINS).toEqual([
      'http://localhost:5173',
      'https://portal.exemplo.gov.br',
    ])
  })

  it('ignora espaços e vírgulas sobrando', () => {
    // "a, b," e "a,,b" são erros de digitação comuns em arquivo de configuração.
    // Uma origem vazia na lista não autorizaria nada e ainda confundiria a
    // leitura, então ela é descartada.
    const resultado = envSchema.parse({
      CORS_ORIGINS: ' http://um.example , , http://dois.example,',
    })

    expect(resultado.CORS_ORIGINS).toEqual(['http://um.example', 'http://dois.example'])
  })
})

describe('envSchema — mensagens de erro', () => {
  /**
   * Quem lê estas mensagens é a nossa equipe, às vezes às pressas, durante um
   * deploy que não subiu. Uma mensagem em inglês, ou genérica demais, custa
   * minutos preciosos justamente na pior hora.
   *
   * Por isso a mensagem é tratada como parte do comportamento do sistema, e não
   * como detalhe cosmético: ela é testada como qualquer outra regra.
   */

  it('explica em português por que a porta foi recusada', () => {
    expect(mensagemDe({ PORT: '8O80' }, 'PORT')).toBe('deve ser um número inteiro')
  })

  it('explica em português a faixa permitida da porta', () => {
    expect(mensagemDe({ PORT: '99999' }, 'PORT')).toBe('deve estar entre 1 e 65535')
  })

  it('explica em português quais ambientes são aceitos', () => {
    expect(mensagemDe({ NODE_ENV: 'producao' }, 'NODE_ENV')).toBe(
      'deve ser "development", "test" ou "production"',
    )
  })

  it('explica em português quando o endereço vem vazio', () => {
    expect(mensagemDe({ HOST: '' }, 'HOST')).toBe('não pode ficar vazio')
  })

  it('explica em português quando o endereço vem com o tipo errado', () => {
    // Este é o caso que estava faltando (item M-14 do checklist de produção).
    expect(mensagemDe({ HOST: 123 }, 'HOST')).toBe('é obrigatória e deve ser um texto')
  })
})

describe('envSchema — confiança em proxy (TRUST_PROXY)', () => {
  /**
   * Esta variável decide de quem a API acredita ser o IP que ela registra e
   * limita. Um erro aqui não aparece na tela de ninguém: ele aparece meses
   * depois, no bloqueio de todos os clientes de uma vez ou no IP errado dentro
   * de uma investigação.
   */

  it('não confia em ninguém quando a variável não é informada', () => {
    // O padrão precisa ser o desconfiado. Ligar a confiança sem um proxy de
    // verdade na frente é pior do que não ligar.
    expect(envSchema.parse({}).TRUST_PROXY).toBe(false)
  })

  it('aceita ligar e desligar por texto', () => {
    expect(envSchema.parse({ TRUST_PROXY: 'true' }).TRUST_PROXY).toBe(true)
    expect(envSchema.parse({ TRUST_PROXY: 'false' }).TRUST_PROXY).toBe(false)
  })

  it('converte a quantidade de proxies para número', () => {
    const resultado = envSchema.parse({ TRUST_PROXY: '1' })

    // O Fastify trata número e texto de formas diferentes: "1" seria lido como
    // um endereço de rede em que confiar, e não como "um salto".
    expect(resultado.TRUST_PROXY).toBe(1)
    expect(typeof resultado.TRUST_PROXY).toBe('number')
  })

  it('recusa quantidade zero ou negativa', () => {
    // Confiar em "zero proxies" é o mesmo que `false`, escrito de um jeito que
    // ninguém entende ao reler o arquivo de configuração seis meses depois.
    expect(envSchema.safeParse({ TRUST_PROXY: '0' }).success).toBe(false)
    expect(envSchema.safeParse({ TRUST_PROXY: '-1' }).success).toBe(false)
  })

  it('recusa qualquer outro texto', () => {
    expect(envSchema.safeParse({ TRUST_PROXY: 'sim' }).success).toBe(false)
    expect(envSchema.safeParse({ TRUST_PROXY: '10.0.0.1' }).success).toBe(false)
  })

  it('explica em português o que ela aceita', () => {
    expect(mensagemDe({ TRUST_PROXY: 'sim' }, 'TRUST_PROXY')).toBe(
      'deve ser "true", "false" ou a quantidade de proxies confiáveis (1 ou mais)',
    )
  })
})

describe('envSchema — volume do log (LOG_LEVEL)', () => {
  it('fica indefinida quando não é informada, para cada ambiente decidir', () => {
    // Ausência aqui não é falta de configuração: é a instrução de usar o nível
    // padrão do ambiente, que está em `src/shared/logger/index.ts`.
    expect(envSchema.parse({}).LOG_LEVEL).toBeUndefined()
  })

  it('aceita os níveis conhecidos', () => {
    expect(envSchema.parse({ LOG_LEVEL: 'debug' }).LOG_LEVEL).toBe('debug')
    expect(envSchema.parse({ LOG_LEVEL: 'silent' }).LOG_LEVEL).toBe('silent')
  })

  it('recusa um nível inventado', () => {
    // "verbose" existe em outras ferramentas e não existe aqui. Sem esta trava,
    // a API subiria com o nível padrão do Pino e ninguém entenderia por que o
    // log não mudou.
    expect(envSchema.safeParse({ LOG_LEVEL: 'verbose' }).success).toBe(false)
  })
})
```

### `.env.example`

```bash
# =============================================
# Modelo de configuração da API do Curso
# =============================================
#
# Este arquivo É versionado no Git e serve de modelo. Ele nunca deve conter
# senha, token ou qualquer segredo de verdade.
#
# Para começar, copie este arquivo para `.env`:
#
#   Windows (PowerShell):  Copy-Item .env.example .env
#   Linux / Mac:           cp .env.example .env
#
# O `.env` é ignorado pelo Git de propósito: é lá que ficam os valores reais
# de cada máquina, incluindo os segredos.

# ---------------------------------------------
# Ambiente de execução
# ---------------------------------------------
# Valores aceitos: development | test | production
NODE_ENV=development

# ---------------------------------------------
# Servidor HTTP
# ---------------------------------------------
# Porta em que a API vai escutar (1 a 65535)
PORT=3333

# Endereço de rede. Use 0.0.0.0 para aceitar conexões de fora da máquina,
# o que é necessário quando a API roda dentro de um container.
HOST=0.0.0.0

# ---------------------------------------------
# Segurança
# ---------------------------------------------
# Sites autorizados a chamar esta API pelo navegador, separados por vírgula.
# Vazio significa nenhum: é o padrão, e é o padrão certo — configuração de
# segurança que nasce aberta "por enquanto" costuma ficar aberta para sempre.
#
# Exemplo com duas origens:
#   CORS_ORIGINS=http://localhost:5173,https://portal.exemplo.gov.br
CORS_ORIGINS=

# Em quantos proxies confiar para descobrir o IP de quem chamou.
# Valores aceitos: false | true | 1, 2, 3...
#
#   false  -> não confiar em ninguém. É o padrão, e é o certo enquanto não
#             houver um proxy de verdade na frente da API.
#   1      -> confiar apenas no proxy mais próximo. É o valor recomendado em
#             produção quando existe um.
#   true   -> confiar na cadeia inteira do cabeçalho X-Forwarded-For.
#
# Ligar isto sem proxy na frente é pior do que não ligar: qualquer pessoa pode
# enviar um X-Forwarded-For inventado, trocar de identidade a cada requisição e
# passar por cima do limite de requisições.
TRUST_PROXY=false

# ---------------------------------------------
# Registro de eventos (logs)
# ---------------------------------------------
# Volume de detalhe do log, do mais silencioso ao mais falante:
# silent | fatal | error | warn | info | debug | trace
#
# Deixe comentado no dia a dia: sem valor, cada ambiente usa o nível que faz
# sentido para ele (debug em desenvolvimento, info em produção, silencioso nos
# testes). Preencha só para investigar um problema específico.
# LOG_LEVEL=debug
```

Atualize também o seu `.env` local — ele não é versionado, então ninguém faz isso por você.
As duas variáveis têm padrão, então nada quebra se você esquecer; o problema é você não
descobrir que elas existem.

### `src/shared/security/index.ts`

Só o comentário mudou, mas ele mudou porque o mundo mudou: o limite conhecido "a contagem usa
o IP do proxy" deixou de existir, e o que sobrou é outro.

```ts
/**
 * Segurança HTTP
 *
 * Três plugins que resolvem três problemas diferentes. Vale não tratá-los como
 * "as bibliotecas de segurança", porque a diferença entre eles muda o que se
 * pode esperar de cada um:
 *
 *   • HELMET     — pede ao NAVEGADOR que ligue proteções que, por omissão, ficam
 *                  desligadas por compatibilidade histórica da web.
 *   • CORS       — diz ao NAVEGADOR quais sites podem chamar esta API em nome de
 *                  quem está navegando.
 *   • RATE LIMIT — a própria API recusa quem passa do volume combinado.
 *
 * Os dois primeiros são **pedidos ao navegador**. Um programa que fale HTTP
 * direto simplesmente os ignora — e isso não os torna inúteis: quem eles
 * protegem é o cidadão que usa um navegador, que é a vítima real desse tipo de
 * ataque. Mas confundir "o navegador respeita" com "está protegido" é o engano
 * mais comum do assunto.
 *
 * Só o rate limit é uma regra que esta API impõe por conta própria.
 */

import fastifyCors from '@fastify/cors'
import fastifyHelmet from '@fastify/helmet'
import fastifyRateLimit from '@fastify/rate-limit'
import type { FastifyInstance } from 'fastify'
import { env } from '../env/index.ts'

/**
 * Volume aceito de qualquer origem, por endereço de IP.
 *
 * 100 por minuto é folgado para uso legítimo e apertado para um cliente com laço
 * de repetição errado — que é a causa mais frequente de sobrecarga, bem mais do
 * que ataque de verdade.
 *
 * A contagem usa `request.ip`. Atrás de um proxy, esse valor só é o IP de quem
 * chamou porque a variável `TRUST_PROXY` diz à API em quantos proxies confiar —
 * sem ela, todos os clientes viriam com o endereço do proxy e o primeiro a
 * estourar bloquearia os demais.
 *
 * **Limite que continua de pé:** a contagem vive na memória deste processo. Duas
 * cópias da API contam separado, e reiniciar zera o placar. Guardar o número
 * fora do processo exige um armazenamento compartilhado, que o projeto ainda
 * não tem.
 */
export const RATE_LIMIT_GLOBAL = {
  max: 100,
  timeWindow: '1 minute',
}

/**
 * Volume aceito na rota de vida, mais generoso que o global.
 *
 * Quem consulta `/health` é o monitoramento, a cada poucos segundos, para
 * sempre. Com o limite global ele seria bloqueado — e alarme que silencia por
 * rate limit é pior do que não ter alarme.
 *
 * 240 por minuto é uma consulta a cada 250 ms, com folga para mais de um
 * monitor. Repare que é um limite **maior**, e não a ausência de limite: rota
 * isenta é rota sem teto, e um endereço público sem teto é alvo de amplificação.
 */
export const RATE_LIMIT_HEALTH = {
  max: 240,
  timeWindow: '1 minute',
}

/**
 * Registra as três camadas de proteção HTTP.
 *
 * Precisa ser chamada antes do registro das rotas, para que toda rota nasça
 * protegida sem ninguém precisar lembrar.
 *
 * @param app Instância do Fastify que vai receber as proteções.
 */
export function registerSecurity(app: FastifyInstance): void {
  // Sem nenhum ajuste: os padrões do Helmet servem a esta API como estão.
  //
  // A dúvida legítima era a política de conteúdo (CSP), que restringe de onde uma
  // página pode carregar script e estilo, e que poderia deixar a documentação em
  // branco. Foi conferida no navegador: a página do Swagger UI carrega tudo do
  // próprio domínio e não usa script embutido, então ela passa pela política
  // padrão sem nenhuma violação. Nada precisou ser afrouxado.
  app.register(fastifyHelmet)

  app.register(fastifyCors, {
    // Lista fechada, vinda do ambiente. Origem que não está aqui não recebe
    // autorização, e o navegador dela bloqueia a resposta.
    origin: env.CORS_ORIGINS,
  })

  app.register(fastifyRateLimit, {
    ...RATE_LIMIT_GLOBAL,

    // O plugin já responde no mesmo formato de três campos da API, mas com a
    // mensagem em inglês. Reescrevemos só a frase: a API responde em português
    // para quem a chama, e um único erro fora do padrão quebra essa promessa.
    //
    // Usamos o `ttl`, que são os milissegundos que faltam, em vez do texto pronto
    // do plugin: assim a resposta diz quanto falta de verdade, e não repete o
    // tamanho da janela como se ela tivesse acabado de começar.
    errorResponseBuilder: (_request, contexto) => {
      const segundos = Math.ceil(contexto.ttl / 1000)

      return {
        statusCode: 429,
        error: 'Too Many Requests',
        message:
          `Limite de ${contexto.max} requisições por minuto excedido. ` +
          `Tente novamente em ${segundos} ${segundos === 1 ? 'segundo' : 'segundos'}.`,
      }
    },
  })
}
```

### `package.json`

Uma dependência de desenvolvimento nova.

```json
{
  "name": "curso_api",
  "version": "1.0.0",
  "description": "API RESTful backend do curso",
  "main": "dist/server.js",
  "type": "module",
  "engines": {
    "node": ">=24"
  },
  "scripts": {
    "dev": "tsx watch --env-file=.env src/server.ts",
    "prebuild": "node --eval \"require('node:fs').rmSync('dist', { recursive: true, force: true })\"",
    "build": "tsc --project tsconfig.build.json",
    "start": "node --env-file-if-exists=.env dist/server.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "docker:build": "docker build -t curso_api .",
    "docker:run": "docker run --rm -p 3333:3333 --name curso_api curso_api",
    "check": "npm run lint && npm run format:check && npm run test && npm run build"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@fastify/cors": "^11.3.0",
    "@fastify/helmet": "^13.1.0",
    "@fastify/rate-limit": "^11.2.0",
    "@fastify/swagger": "^9.8.1",
    "@fastify/swagger-ui": "^6.1.1",
    "fastify": "^5.12.0",
    "fastify-type-provider-zod": "^7.0.0",
    "zod": "^4.4.3"
  },
  "devDependencies": {
    "@eslint/js": "^10.0.1",
    "@types/node": "^26.2.0",
    "eslint": "^10.8.1",
    "eslint-config-prettier": "^10.1.8",
    "pino-pretty": "^13.1.3",
    "prettier": "^3.9.6",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0",
    "vitest": "^4.1.10"
  }
}
```

### `README.md`

Duas linhas na tabela de variáveis e dois parágrafos na seção do container: um sobre dar prazo
ao `docker stop`, outro sobre configurar o `TRUST_PROXY` atrás de proxy. O README é a primeira
coisa que alguém lê ao entrar no projeto — configuração que só existe no código é configuração
que ninguém usa.

````markdown
# API do Curso

API RESTful do curso, construída com **Fastify + TypeScript**.

## Começando

```bash
npm install     # baixa as dependências
npm run dev     # sobe a API em http://localhost:3333
```

**Pré-requisito:** Node.js na versão registrada no `.nvmrc`.

## Configuração

Antes de subir a API pela primeira vez, crie o seu `.env` a partir do modelo versionado:

```bash
# Windows (PowerShell)
Copy-Item .env.example .env

# Linux / Mac
cp .env.example .env
```

| Variável       | O que controla                                    | Padrão           |
| :------------- | :------------------------------------------------ | :--------------- |
| `NODE_ENV`     | Ambiente: `development`, `test` ou `production`   | `development`    |
| `PORT`         | Porta em que a API escuta                         | `3333`           |
| `HOST`         | Endereço de rede que a API aceita                 | `0.0.0.0`        |
| `CORS_ORIGINS` | Sites que podem chamar a API pelo navegador       | _(nenhum)_       |
| `TRUST_PROXY`  | Em quantos proxies confiar para saber quem chamou | `false`          |
| `LOG_LEVEL`    | Volume do log; vazio usa o padrão do ambiente     | _(por ambiente)_ |

O `.env` **não** é versionado: ele guarda os valores reais de cada máquina, incluindo os
segredos. O `.env.example` é versionado e serve de modelo — ele nunca contém senha de verdade.

## Comandos

| Comando                | O que faz                                          |
| :--------------------- | :------------------------------------------------- |
| `npm run dev`          | Sobe a API recarregando a cada alteração salva     |
| `npm run build`        | Compila o TypeScript para a pasta `dist`           |
| `npm start`            | Executa a versão compilada, como roda em produção  |
| `npm test`             | Roda todos os testes uma vez                       |
| `npm run test:watch`   | Deixa os testes rodando a cada arquivo salvo       |
| `npm run lint`         | Procura problemas de lógica e qualidade no código  |
| `npm run lint:fix`     | Corrige sozinho o que for corrigível               |
| `npm run format`       | Formata todos os arquivos com o Prettier           |
| `npm run format:check` | Confere a formatação sem alterar nada              |
| `npm run check`        | Roda lint, formatação, testes e build em sequência |
| `npm run docker:build` | Constrói a imagem da API                           |
| `npm run docker:run`   | Sobe a API em container, na porta 3333             |

## Rotas

| Método | Rota            | O que devolve                                     |
| :----- | :-------------- | :------------------------------------------------ |
| `GET`  | `/health`       | **Vida:** apenas `{ "status": "ok" }`             |
| `GET`  | `/health/ready` | **Prontidão:** status, uptime, momento e ambiente |

Toda rota declara o contrato da resposta com Zod. Campo que não está no contrato **não sai**,
mesmo que o código o devolva por engano.

## Documentação da API

Com a API rodando fora de produção, a documentação navegável fica em:

```
http://localhost:3333/documentation
```

Ela é **gerada a partir dos schemas do código**, e não escrita à mão: o que está lá é
exatamente o que a API aceita e devolve. Dá para disparar cada rota pela própria página.

| Endereço              | O que é                          |
| :-------------------- | :------------------------------- |
| `/documentation`      | A página navegável               |
| `/documentation/json` | A especificação OpenAPI, em JSON |
| `/documentation/yaml` | A mesma especificação, em YAML   |

**Em produção os três respondem 404**, por decisão registrada: enquanto a API só tiver rotas
de saúde, publicar o mapa completo não ajuda ninguém de fora e ajuda quem estiver mapeando o
serviço. Quando existirem rotas de negócio e autenticação, a documentação volta a subir —
protegida por login.

## Formato de erro

Toda falha responde com o mesmo corpo, qualquer que seja a rota ou o código HTTP:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Endereço não encontrado: GET /rota-que-nao-existe"
}
```

| Campo        | O que é                                            |
| :----------- | :------------------------------------------------- |
| `statusCode` | O código HTTP repetido no corpo                    |
| `error`      | O nome oficial do código HTTP, em inglês           |
| `message`    | A explicação em português, escrita para uma pessoa |

Erro inesperado (código 500) sempre responde com uma mensagem genérica. O detalhe técnico vai
para o log estruturado do servidor, nunca para o cliente.

## Segurança

| Proteção       | O que faz                                                   |
| :------------- | :---------------------------------------------------------- |
| **Helmet**     | Liga os cabeçalhos de segurança que o navegador respeita    |
| **CORS**       | Só as origens em `CORS_ORIGINS` podem chamar pelo navegador |
| **Rate limit** | 100 requisições por minuto, por IP                          |

A rota `/health` tem limite próprio, de 240 por minuto, para não bloquear o monitoramento.

**Limite conhecido:** a contagem usa o IP da conexão. Atrás de um proxy, todos os clientes são
contados como um só. Será corrigido quando a API passar a rodar atrás de proxy.

## Rodando por container

Requer o Docker Desktop instalado e em execução.

```bash
npm run docker:build   # constrói a imagem
npm run docker:run     # sobe a API na porta 3333
```

A imagem é multi-estágio: o código-fonte e as ferramentas de desenvolvimento ficam fora da
imagem final. O processo roda como usuário sem privilégios (`node`), e o Docker verifica a
saúde da API pela rota `/health` a cada 30 segundos — `docker ps` mostra `(healthy)`.

**Ao derrubar o container, dê prazo.** A API trata o `SIGTERM`: ela para de aceitar conexão
nova, espera as requisições em andamento responderem e só então sai, com código 0. Isso leva
o tempo da requisição mais lenta, e o `docker stop` sem `-t` não espera tudo isso —
medimos cerca de 3,4 s de prazo padrão, menos que o prazo interno de 10 s da própria API.
Quando o prazo de fora é menor, quem decide é o `SIGKILL`, que corta a requisição no meio:

```bash
docker stop -t 30 curso_api
```

**Atrás de um proxy, configure o `TRUST_PROXY`.** Sem ele, `request.ip` é o endereço do
proxy: o limite de requisições passa a contar todos os clientes como um só, e o log registra
sempre o mesmo IP. Com um proxy na frente, o valor é `1`. Sem proxy nenhum, é `false` — e
ligar sem proxy é pior do que não ligar, porque aí qualquer cliente escolhe o próprio IP
enviando um `X-Forwarded-For` inventado.

Dentro do container a configuração chega por variável de ambiente, e não por arquivo `.env`.
````

### `requisicoes/health.http`

Duas requisições novas: uma com segredo na URL, para ver o mascaramento acontecer, e uma com
`X-Forwarded-For` forjado, para ver a diferença que o `TRUST_PROXY` faz.

```http
# Requisições da funcionalidade de Health Check
#
# Este arquivo é lido pela extensão REST Client (humao.rest-client).
# Com o servidor rodando (`npm run dev`), clique em "Send Request" logo acima
# de cada requisição para executá-la sem sair do editor.
#
# Como fica versionado no Git, todo o time compartilha exatamente as mesmas
# requisições de teste — diferente do Postman, onde cada um monta as suas.

@host = http://localhost:3333

### Verificar se a API está no ar (rota de vida)
GET {{host}}/health

### Verificar se a API está pronta para atender (rota de prontidão)
GET {{host}}/health/ready

### Conferir que um endereço inexistente responde no formato de erro da API
GET {{host}}/rota-que-nao-existe

### Baixar a especificação OpenAPI da API (a mesma que a página /documentation lê)
GET {{host}}/documentation/json

### Conferir que um segredo enviado na URL NÃO chega ao log
# A resposta é a de sempre. O que muda está no terminal do `npm run dev`:
# a linha do log registra `token=[Redacted]`, e não o valor enviado.
GET {{host}}/health?token=segredo-que-nao-pode-ser-gravado&pagina=2

### Fingir vir de outro IP, para ver a diferença que o TRUST_PROXY faz
# Com TRUST_PROXY=false (o padrão), a API ignora o cabeçalho e registra o
# endereço real da conexão. Com TRUST_PROXY=true, ela acredita no que veio aqui.
GET {{host}}/health/ready
X-Forwarded-For: 9.9.9.9
```

---

## 💾 Fechando o ciclo: mande para o GitHub

```bash
git add .
git commit -m "feat: desliga com calma, confia no proxy certo e protege o log"
git push
```

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Precisa terminar sem erro, com todos os testes passando.

### 2. O container sai com código 0

```bash
docker build -t curso_api .
docker rm -f curso_api
docker run -d -p 3333:3333 --name curso_api curso_api
docker stop -t 30 curso_api
docker inspect curso_api --format "{{.State.ExitCode}}"
```

Precisa responder `0`. Se responder `137`, o `SIGTERM` não está sendo tratado.

### 3. O log da despedida aparece

```bash
docker logs curso_api
```

Precisa terminar com `Servidor encerrado. Nenhuma requisição foi cortada.`

### 4. O segredo não vai para o log

Com `npm run dev` rodando:

```bash
curl "http://127.0.0.1:3333/health?token=segredo123"
```

O terminal precisa mostrar `token=[Redacted]`.

### 5. O `Ctrl+C` também encerra direito

Com `npm run dev` rodando, aperte `Ctrl+C`. Precisa aparecer a linha
`Sinal de desligamento recebido` antes de o terminal voltar.

### 6. A variável recusa valor inválido

```bash
# PowerShell
$env:TRUST_PROXY="sim"; npm run dev
```

A API precisa **não subir**, dizendo que `TRUST_PROXY` deve ser `"true"`, `"false"` ou a
quantidade de proxies confiáveis. Apague a variável depois (`Remove-Item Env:TRUST_PROXY`).

---

## 🚨 Erros Comuns

### "Implementei tudo e o container continua saindo com 137"

Você provavelmente usou `docker stop` sem `-t`. O prazo padrão medido foi de ~3,4 segundos, e
a requisição do exemplo leva 5. **O log vai dizer "encerrando com calma" mesmo assim** — o que
denuncia é o código de saída.

### "O `Ctrl+C` não faz nada e eu preciso fechar o terminal"

Confira se você registrou os ouvintes **antes** do `await app.listen(...)`. Se o sinal chegar
durante a subida, precisa haver quem o trate.

### "Ligue o `trustProxy` e o limite de requisições parou de funcionar"

É o Capítulo 6 acontecendo com você. `TRUST_PROXY=true` sem um proxy real na frente permite
que qualquer cliente escolha o próprio IP. Volte para `false`, ou use `1` **com** proxy.

### "O `pino-pretty` não é encontrado no container"

Ele é dependência de desenvolvimento, e a imagem instala com `--omit=dev`. Isso é o esperado.
Se o erro apareceu, é porque o container está rodando com `NODE_ENV=development` — confira o
`-e NODE_ENV` do seu `docker run`.

### "O meu `docker stop` demorou 10 segundos, não 3,4"

Ótimo: significa que a sua versão do Docker usa o padrão documentado. O número não importa —
a conclusão é a mesma, passe o `-t`.

---

## 🏋️ Exercícios

### 1. Descubra o prazo padrão da sua máquina

Meça, na sua instalação do Docker, quanto tempo o `docker stop` sem `-t` espera antes do
`SIGKILL`. Use um container que ignore sinais, como o `node:24-slim` do Capítulo 2, e cronometre.

### 2. Faça a API estourar o próprio prazo

Sem alterar o `server.ts`, faça o log registrar a linha
`O encerramento passou do prazo`. Explique, em uma frase, o que foi preciso para isso.

### 3. Prove que o `CPF` não vai para o log

A lista `PARAMETROS_SENSIVEIS` inclui `cpf`. Escreva o teste que prova isso para uma URL com
dois parâmetros, sendo um deles o CPF, e rode.

### 4. Descubra o que acontece com dois proxies

Com `TRUST_PROXY=1` e um cabeçalho `X-Forwarded-For: 1.1.1.1, 2.2.2.2, 3.3.3.3`, qual IP a API
vai registrar? Preveja antes de testar, depois teste e compare.

Os gabaritos comentados estão em [`exercicios/11-gabarito.md`](./exercicios/11-gabarito.md).

---

## 📌 O que fica para a próxima aula

Esta aula deixou uma pergunta no ar de propósito: montar a cena do proxy exigiu criar uma rede
na mão, subir dois containers com nomes combinados e apontar um volume para um arquivo de
configuração. Funciona, mas ninguém quer digitar isso todo dia — e ninguém vai lembrar da
ordem daqui a duas semanas.

A **Aula 12** resolve isso com o Docker Compose: o mesmo ambiente inteiro descrito em um
arquivo versionado, e um comando só para subir tudo. É o que vai tornar possível, na aula
seguinte, ter um banco de dados de verdade sem ninguém instalar MySQL na mão.

Fica também registrado o que **não** foi resolvido aqui: a contagem do limite de requisições
continua vivendo na memória do processo. Duas cópias da API contam separado, e reiniciar zera
o placar. Guardar esse número fora do processo exige um armazenamento compartilhado, e nada se
instala antes de ser usado.
