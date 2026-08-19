# 🛡️ Aula 05: Tratamento de Erros — O que a API Nunca Deve Contar

Bem-vindos à **Aula 05**! 🎉

Até aqui, todas as aulas cuidaram do caminho em que tudo dá certo. Hoje é o contrário:
vamos cuidar do caminho em que tudo dá errado.

E vamos fazer isso na ordem inversa da que estamos acostumados. Pela primeira vez, você vai
**ver o problema com os próprios olhos antes de existir a solução**. Não vamos acreditar em
mim: vamos provocar a falha, olhar o que saiu pela internet, e só então consertar.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar por que uma mensagem de erro pode ser um problema de **segurança**.
- Separar erro **esperado** de erro **inesperado**, e justificar por que só um dos dois pode
  ser mostrado ao cliente.
- Registrar um tratamento de erro global no Fastify, que cobre toda rota criada dali em
  diante.
- Definir um formato único de resposta de erro para a API inteira.
- Guardar as requisições que provam cada caso em um arquivo `.http` versionado, para
  reconferir quando quiser.
- Publicar esse formato no `README.md`, como contrato de quem consome a API.

## 📋 Pré-requisitos

- Aulas 01 a 04 concluídas.
- `npm run check` passando.
- A extensão **REST Client**, instalada na Aula 01. Vamos usá-la de verdade hoje.
- Nada para instalar nesta aula. Tudo o que vamos usar já está no projeto.

---

## 📖 Capítulo 1: O problema, visto de perto

### A pergunta que ninguém faz

Quando escrevemos uma rota, pensamos no que ela devolve quando dá certo. Quase nunca
paramos para perguntar:

> _"E quando der errado, o que exatamente o cliente vai receber?"_

A resposta hoje é: **o que o Fastify decidir**. E o que ele decide, por padrão, é devolver a
mensagem crua da exceção.

### Vamos ver isso acontecer

Não vamos acreditar na minha palavra. Abra `src/app.ts` e acrescente esta rota
temporária, logo antes do `return app`:

```typescript
// ⚠️ TEMPORÁRIA — vamos apagar ainda nesta aula.
app.get('/vazamento', async () => {
  throw new Error("Unknown column 'cpf' in field list: SELECT * FROM cidadaos")
})
```

Essa mensagem não foi inventada por capricho: é o formato exato do que um banco de dados
MySQL devolve quando uma consulta erra o nome de uma coluna. Guarde isso — é o tipo de erro
que vai acontecer de verdade no dia a dia, assim que a API passar a falar com um banco.

Agora suba a API:

```bash
npm run dev
```

E acesse `http://localhost:3333/vazamento` no navegador. Você vai ver:

```json
{
  "statusCode": 500,
  "error": "Internal Server Error",
  "message": "Unknown column 'cpf' in field list: SELECT * FROM cidadaos"
}
```

### Leia de novo o que acabou de sair pela internet

Aquela linha entregou, para **qualquer pessoa** que acessasse o endereço:

| O que vazou     | O que isso conta para quem quer atacar               |
| :-------------- | :--------------------------------------------------- |
| `cidadaos`      | O nome de uma tabela do banco                        |
| `cpf`           | O nome de uma coluna, e que ela guarda dado sensível |
| `SELECT * FROM` | Que o sistema monta consultas SQL, e em qual formato |
| A frase inteira | Que o banco é MySQL, pelo formato característico     |

> [!CAUTION]
> Nenhuma dessas informações, sozinha, invade nada. Juntas, elas são o mapa. Um ataque
> começa exatamente assim: alguém provoca erros de propósito, um atrás do outro, e vai
> montando a planta do prédio pelas mensagens que voltam. Isso tem nome — chama-se
> **enumeração**, e é a primeira fase de praticamente todo ataque a sistema web.

### A analogia do prédio

Imagine que alguém toca o interfone do prédio e pede para falar com o setor financeiro. Há
duas respostas possíveis:

- 🔴 _"O financeiro fica na sala 302, terceiro andar, mas o Roberto saiu para almoçar e a
  porta dos fundos está com a fechadura quebrada desde ontem."_
- 🟢 _"Não posso ajudar com isso. Já avisei a portaria."_

As duas respostas negam o pedido. Só uma delas não entrega a planta do prédio junto.

**Este é o trabalho desta aula: trocar a primeira resposta pela segunda — sem que a portaria
deixe de ser avisada.**

---

## 🧠 Capítulo 2: A ideia central

### Quem escreveu a mensagem?

Existe uma pergunta simples que resolve o problema inteiro:

> **A mensagem deste erro foi escrita por nós, para ser lida por quem consome a API?**

Só isso. Compare:

| Mensagem                                             | Quem escreveu  | Para quem           | Pode sair? |
| :--------------------------------------------------- | :------------- | :------------------ | :--------: |
| `Protocolo não encontrado`                           | Nós            | O cidadão           |     ✅     |
| `CPF inválido`                                       | Nós            | O cidadão           |     ✅     |
| `Unknown column 'cpf' in field list`                 | O MySQL        | Um desenvolvedor    |     ❌     |
| `ECONNREFUSED 127.0.0.1:3306`                        | O sistema      | Um administrador    |     ❌     |
| `Cannot read properties of undefined (reading 'id')` | O próprio Node | Quem escreveu o bug |     ❌     |

Repare no padrão: as mensagens seguras são as que **nós** escrevemos. As perigosas vieram de
fora — de bibliotecas, do banco, do sistema operacional. Nenhuma delas foi escrita pensando
em quem está do outro lado da internet.

### Os dois tipos de erro

Vamos dar nome a essa divisão:

- **Erro esperado** — a regra de negócio previu. "Protocolo não encontrado" não é um bug: é
  o sistema funcionando. Alguém pediu algo que não existe, e nós respondemos exatamente
  isso.
- **Erro inesperado** — ninguém previu. Banco fora do ar, bug no nosso código, biblioteca
  que quebrou. Aqui não sabemos nem o que a mensagem contém.

Para o erro esperado, temos a mensagem pronta. Para o inesperado, a única resposta honesta é
uma frase genérica.

> [!IMPORTANT]
> O detalhe do erro inesperado **não é jogado fora**. Ele vai inteiro, com stack trace, para
> o log estruturado — que é onde a equipe investiga. O que muda não é a existência da
> informação: é **quem tem acesso a ela**.

### E em desenvolvimento? Posso mostrar o detalhe?

É comum ver projetos que mostram o stack trace na resposta quando não estão em produção.
**Não vamos fazer isso**, e o motivo vale a pena entender:

Se o comportamento em desenvolvimento for diferente do de produção, então o caminho que roda
em produção é justamente o que ninguém vê no dia a dia. Ele passa meses sem ser exercitado,
até falhar na hora errada.

Um único comportamento, igual em todo lugar, é um comportamento que está sempre sendo
exercitado.

---

## 🧪 Capítulo 3: A prova antes do código

Aqui a aula muda de ritmo. Em vez de escrever a solução e depois conferir, vamos escrever a
**exigência** primeiro, ver o projeto reprovar, e só então resolver.

Isso vale para qualquer coisa que você for construir, e o ciclo é sempre o mesmo:

```mermaid
flowchart LR
    A["🔴 Escreve a exigência<br/>e vê o projeto reprovar"] --> B["🟢 Escreve o código<br/>até ele atender"]
    B --> C["🔵 Melhora o código<br/>reconferindo a cada passo"]
    C --> A
```

O passo que parece bobo — **ver a reprovação** — é o mais importante dos três. Uma exigência
que nunca reprovou não provou nada: pode estar conferindo a coisa errada, ou nem estar sendo
conferida.

### Passo 1: Escrevendo a exigência em um arquivo `.http`

Na Aula 01 você criou `requisicoes/health.http` e instalou a extensão **REST Client**. Aquilo
não foi um enfeite: é assim que se guarda uma requisição de teste dentro do repositório, para
qualquer pessoa do time repetir exatamente a mesma chamada.

Vamos usar de novo. Crie o arquivo `requisicoes/erros.http`:

```http
# Requisições que provam o tratamento de erros
#
# Cada bloco abaixo é uma requisição que você dispara clicando em
# "Send Request", logo acima dela, com a API rodando (`npm run dev`).
#
# Este arquivo é versionado de propósito. Ele é a memória do time sobre COMO
# se confere que o tratamento de erros continua funcionando — quem chegar
# daqui a um ano não precisa adivinhar quais endereços visitar.

@host = http://localhost:3333

### 1. Falha inesperada — a resposta NÃO pode conter "cidadaos" nem "SELECT"
GET {{host}}/vazamento

### 2. Endereço que não existe — precisa vir no formato de erro da API
GET {{host}}/endereco-que-nao-existe

### 3. Método errado em rota que existe — também é "não encontrado"
POST {{host}}/health

### 4. O caminho feliz — precisa continuar respondendo igual
GET {{host}}/health
```

**O que cada parte faz:**

| Trecho             | Para que serve                                                                |
| :----------------- | :---------------------------------------------------------------------------- |
| `@host = ...`      | Uma variável. Trocar o endereço num lugar só troca em todas as requisições    |
| `###`              | Separa uma requisição da outra. É o que faz aparecer o botão **Send Request** |
| `{{host}}`         | Usa o valor da variável                                                       |
| Comentário com `#` | Explica **o que se espera**, que é a parte que o arquivo sozinho não conta    |

### Passo 2: Vendo o projeto reprovar

Com a API no ar (`npm run dev`), abra `requisicoes/erros.http` no VS Code e clique em
**Send Request** na requisição **1**.

A resposta abre numa aba ao lado:

```json
{
  "statusCode": 500,
  "error": "Internal Server Error",
  "message": "Unknown column 'cpf' in field list: SELECT * FROM cidadaos"
}
```

Ali está `cidadaos`. Ali está `SELECT`. **A exigência número 1 reprovou.**

Dispare também a **2**:

```json
{
  "message": "Route GET:/endereco-que-nao-existe not found",
  "error": "Not Found",
  "statusCode": 404
}
```

Repare que essa mensagem está em **inglês**, fala em "Route" e usa um formato que não é
nosso — é o padrão do Fastify. Reprovou também.

A exigência está certa. O código é que ainda não está. **Agora** podemos escrever a
solução — e vamos saber com precisão quando ela ficou pronta.

> [!TIP]
> Deixe as duas abas abertas, lado a lado: o `.http` de um lado e a resposta do outro. Ao
> longo do capítulo seguinte você vai reenviar as mesmas requisições e ver a resposta mudar,
> sem sair do editor.

---

## 🏗️ Capítulo 4: Construindo a solução

### Passo 3: A classe `AppError`

Precisamos de um jeito de dizer, no código: _"esta mensagem é minha, pode sair"_.

A forma mais direta é criar um **tipo de erro próprio**. Quando o handler receber um erro,
basta perguntar se ele é desse tipo.

Crie a pasta `src/shared/errors/` e, dentro dela, o arquivo `app-error.ts`:

```typescript
/**
 * AppError — Erro esperado da aplicação
 *
 * Este arquivo existe para responder uma única pergunta, toda vez que algo dá
 * errado: **esta mensagem pode ser lida por quem está do outro lado da API?**
 *
 * A resposta depende de quem escreveu a mensagem:
 *
 *   - Se fomos nós ("Protocolo não encontrado"), ela foi escrita justamente para
 *     ser lida por quem consome a API. Pode sair inteira.
 *   - Se veio de uma biblioteca ou do banco de dados ("Unknown column 'cpf' in
 *     field list"), ela foi escrita para desenvolvedores e revela a estrutura
 *     interna do sistema. Não pode sair de jeito nenhum.
 *
 * Lançar `AppError` é a forma de dizer "esta mensagem é minha e é segura".
 * Qualquer outro erro é tratado como inesperado por padrão — o lado seguro.
 */

/**
 * Erro previsto pela regra de negócio, com a mensagem já pronta para o cliente.
 *
 * @example `throw new AppError('Protocolo não encontrado', 404)`
 */
export class AppError extends Error {
  /**
   * Código de status HTTP que a resposta deve carregar.
   *
   * Fica junto da mensagem de propósito: quem lança o erro é quem sabe se aquilo
   * é um "não encontrei" (404) ou um "você mandou errado" (400). Deixar essa
   * decisão para o handler global obrigaria ele a adivinhar.
   */
  public readonly statusCode: number

  /**
   * @param message    Mensagem em português, escrita para ser lida por quem consome a API.
   * @param statusCode Código HTTP da resposta. O padrão 400 cobre o caso mais comum,
   *                   que é a requisição ter chegado com algo errado.
   */
  constructor(message: string, statusCode = 400) {
    super(message)

    // Sem esta linha, `error.name` sairia como "Error" em todo log e em toda
    // ferramenta de monitoramento, e não haveria como distinguir um erro nosso
    // de um erro vindo de fora.
    this.name = 'AppError'

    this.statusCode = statusCode
  }
}
```

**Explicando:**

1. `extends Error` — `AppError` **é** um `Error`. Ele continua funcionando com `throw`, com
   `try/catch` e com tudo o mais que já existe. Só ganhou um campo a mais.
2. `super(message)` — entrega a mensagem para o `Error` que está por baixo. É ele quem
   preenche o `.message` e o stack trace.
3. `public readonly statusCode` — `readonly` significa que ninguém pode trocar o código
   depois que o erro foi criado. Um erro que muda de identidade no meio do caminho é uma
   fonte de bug muito difícil de rastrear.
4. `statusCode = 400` — valor padrão. Se alguém esquecer de escolher, cai no 400
   ("você mandou algo errado"), que é o palpite mais seguro para um erro previsto.

> [!NOTE]
> **Por que uma classe, e não um objeto solto?**
>
> Porque `instanceof` só funciona com classes. É ele que vai permitir, no próximo arquivo,
> perguntar "este erro é nosso?" com uma linha só — sem inventar convenções tipo "se tiver a
> propriedade `ehNosso: true`", que qualquer biblioteca poderia imitar por acidente.

### Passo 4: O handler global

Agora o arquivo que decide o que sai. Crie `src/shared/errors/error-handler.ts`:

```typescript
/**
 * Tratamento centralizado de erros
 *
 * Sem este arquivo, cada falha dentro de uma rota é respondida pelo comportamento
 * padrão do Fastify: código 500 com a mensagem crua da exceção no corpo. Com
 * banco de dados, essa mensagem crua carrega nome de tabela, nome de coluna e
 * trecho de consulta — um mapa da estrutura interna, entregue ao navegador de
 * quem quer que tenha feito a requisição.
 *
 * A decisão central aqui é uma só, e vale para toda a API:
 *
 *   **O cliente recebe a mensagem apenas quando fomos nós que a escrevemos.**
 *
 * O detalhe técnico nunca se perde: ele vai para o log estruturado, que é onde a
 * equipe procura. O que muda é quem tem acesso a ele.
 */

import { STATUS_CODES } from 'node:http'
import type { FastifyError, FastifyReply, FastifyRequest } from 'fastify'
import { AppError } from './app-error.ts'

/**
 * Formato único de resposta de erro desta API.
 *
 * Ter um formato só é o que permite a quem consome a API escrever **um** trecho
 * de código para ler erro, em vez de um para cada situação.
 */
interface RespostaDeErro {
  /** Código HTTP repetido no corpo, para quem lê o JSON sem olhar o cabeçalho. */
  statusCode: number
  /** Nome oficial do código HTTP, em inglês (ex.: "Not Found"). */
  error: string
  /** Mensagem em português, escrita para ser lida por uma pessoa. */
  message: string
}

/**
 * Monta o corpo da resposta de erro no formato único da API.
 *
 * @param statusCode Código HTTP da resposta.
 * @param message    Mensagem já considerada segura para sair.
 */
function montarResposta(statusCode: number, message: string): RespostaDeErro {
  return {
    statusCode,
    // O Node já traz a tabela oficial de nomes dos códigos HTTP. Escrever a
    // nossa daria manutenção e correria o risco de divergir do padrão.
    error: STATUS_CODES[statusCode] ?? 'Error',
    message,
  }
}

/**
 * Handler global de erros, registrado uma única vez em `buildApp()`.
 *
 * Registrar aqui, e não em cada rota, é o que garante que toda rota criada de
 * agora em diante já nasça protegida — ninguém precisa lembrar de nada.
 *
 * @param error   O erro que foi lançado em algum ponto da requisição.
 * @param request A requisição, usada para registrar o erro no log.
 * @param reply   A resposta que será enviada ao cliente.
 * @returns       A resposta já enviada, no formato único de erro.
 */
export function errorHandler(
  error: FastifyError,
  request: FastifyRequest,
  reply: FastifyReply,
): FastifyReply {
  // 1º caso: erro que nós mesmos lançamos. A mensagem foi escrita para o cliente,
  // então ela sai inteira, com o código que quem a lançou escolheu.
  if (error instanceof AppError) {
    return reply.status(error.statusCode).send(montarResposta(error.statusCode, error.message))
  }

  // 2º caso: erro que o próprio Fastify levantou sobre a requisição recebida
  // (corpo mal formado, tipo de conteúdo não suportado, campo inválido). A
  // mensagem fala do que o cliente enviou, não do que existe dentro do servidor.
  // Os códigos abaixo de 500 são exatamente os que significam "o problema veio
  // de fora".
  if (error.statusCode !== undefined && error.statusCode < 500) {
    return reply.status(error.statusCode).send(montarResposta(error.statusCode, error.message))
  }

  // 3º caso: qualquer outra coisa. Bug no nosso código, banco fora do ar,
  // biblioteca que quebrou. Não sabemos o que essa mensagem contém, então
  // partimos do princípio de que ela não pode sair.
  //
  // O erro inteiro, com stack trace, é registrado no log estruturado. É lá que a
  // equipe investiga — e é o único lugar onde esse detalhe deve existir.
  request.log.error({ err: error }, 'Erro não tratado durante a requisição')

  return reply
    .status(500)
    .send(montarResposta(500, 'Erro interno do servidor. A equipe já foi avisada.'))
}

/**
 * Handler para endereços que não existem nesta API.
 *
 * O Fastify já responde 404 sozinho, mas em um formato diferente do nosso. Sem
 * este handler, a API teria dois formatos de erro conviventes, e quem a consome
 * precisaria tratar os dois.
 *
 * @param request A requisição que não encontrou rota.
 * @param reply   A resposta que será enviada ao cliente.
 * @returns       A resposta 404 no formato único de erro.
 */
export function notFoundHandler(request: FastifyRequest, reply: FastifyReply): FastifyReply {
  // Repetimos o método e o caminho na mensagem porque isso é informação que o
  // próprio cliente acabou de enviar: não revela nada que ele já não saiba.
  return reply
    .status(404)
    .send(montarResposta(404, `Endereço não encontrado: ${request.method} ${request.url}`))
}
```

**Explicando os três casos, na ordem em que aparecem:**

1. **`error instanceof AppError`** — fomos nós. A mensagem sai inteira, com o código que quem
   lançou o erro escolheu.

2. **`error.statusCode < 500`** — este caso merece atenção. São os erros que o **próprio
   Fastify** levanta sobre a requisição que chegou: corpo em JSON quebrado, tipo de conteúdo
   que a rota não aceita, campo obrigatório faltando.

   Por que essas mensagens podem sair? Porque elas falam sobre **o que o cliente enviou**, e
   não sobre o que existe dentro do servidor. "O corpo da requisição não é um JSON válido" não
   revela nada — quem mandou o JSON quebrado foi ele.

   E a divisão em 500 não é arbitrária: na especificação do HTTP, os códigos `4xx` significam
   literalmente _"o problema está do seu lado"_ e os `5xx`, _"o problema está do meu lado"_.
   Estamos usando uma linha que o próprio protocolo já desenhou.

3. **Todo o resto** — não sabemos o que é. Frase genérica na resposta, erro inteiro no log.

> [!TIP]
> **Por que `STATUS_CODES` do `node:http`?**
>
> É a tabela oficial que traduz `404` para `"Not Found"`, e ela já vem dentro do Node — não
> instalamos nada. Escrever a nossa própria tabela significaria mantê-la, e correr o risco de
> ela discordar do padrão. Sempre que a plataforma já resolveu um problema, use a solução
> dela.

### Passo 5: Ligando os handlers no `app.ts`

Os dois arquivos existem, mas ninguém os usa ainda. É o `app.ts` que os coloca em serviço.

Abra `src/app.ts` e deixe **exatamente** assim. Repare que a rota `/vazamento` do Capítulo 1
**continua aqui** — vamos precisar dela mais um pouco, para provar que o handler funciona, e
ela sai no Capítulo 6:

```typescript
/**
 * App — Montagem da instância do Fastify
 *
 * Este arquivo é responsável por:
 *   1. Criar a instância do Fastify com suas configurações.
 *   2. Registrar o tratamento centralizado de erros.
 *   3. Registrar plugins globais.
 *   4. Registrar as rotas de cada módulo.
 *
 * Separamos a montagem do app (aqui) da inicialização do servidor (`server.ts`)
 * porque são duas responsabilidades diferentes: uma decide quais rotas e
 * plugins a aplicação tem, a outra decide em qual endereço ela atende. Quem
 * monta a aplicação não precisa saber nada sobre rede.
 */

import Fastify from 'fastify'
import type { FastifyInstance } from 'fastify'
import { healthRoutes } from './modules/health/health.routes.ts'
import { errorHandler, notFoundHandler } from './shared/errors/error-handler.ts'

/**
 * Fábrica da aplicação Fastify.
 *
 * @returns Instância do Fastify já configurada, pronta para receber requisições.
 */
export function buildApp(): FastifyInstance {
  const app = Fastify({
    // O Fastify já vem com o Pino, um dos loggers mais rápidos do Node.
    // Deixá-lo ligado nos dá o registro de cada requisição sem escrever uma linha.
    logger: true,
  })

  // Registramos o tratamento de erros ANTES das rotas, de propósito: assim toda
  // rota registrada daqui para baixo já nasce coberta, e ninguém precisa lembrar
  // de tratar erro em cada rota nova.
  app.setErrorHandler(errorHandler)
  app.setNotFoundHandler(notFoundHandler)

  // No Fastify, cada conjunto de rotas é registrado como um plugin. Isso mantém
  // cada módulo isolado: um erro no registro de um não derruba os outros.
  app.register(healthRoutes)

  // ⚠️ TEMPORÁRIA — sai no Capítulo 6 desta aula.
  app.get('/vazamento', async () => {
    throw new Error("Unknown column 'cpf' in field list: SELECT * FROM cidadaos")
  })

  return app
}
```

**Duas linhas novas, e é isso.** Vale medir o que elas compraram:

- `app.setErrorHandler(errorHandler)` — o Fastify aceita **um** handler de erro por
  instância. A partir daqui, toda exceção lançada em qualquer rota passa por ele.
- `app.setNotFoundHandler(notFoundHandler)` — o mesmo, para endereços que não existem.

> [!IMPORTANT]
> Repare no que **não** precisamos fazer: sair rota por rota acrescentando `try/catch`.
>
> Essa é a diferença entre tratamento **centralizado** e tratamento espalhado. Espalhado,
> a proteção depende de alguém lembrar em cada rota nova — e um esquecimento em uma única
> rota basta para vazar. Centralizado, a rota nasce protegida porque o `buildApp()` é o
> caminho por onde toda rota passa.

### Passo 6: Vendo a exigência ser atendida

Pare a API (`Ctrl+C`) e suba de novo:

```bash
npm run dev
```

Volte ao `requisicoes/erros.http` e dispare a requisição **1** outra vez. Agora:

```json
{
  "statusCode": 500,
  "error": "Internal Server Error",
  "message": "Erro interno do servidor. A equipe já foi avisada."
}
```

Nem `cidadaos`, nem `SELECT`, nem sinal do MySQL. **A exigência número 1 passou.**

Agora olhe o **terminal** onde o `npm run dev` está rodando. O detalhe não sumiu — ele está
lá inteiro, com stack trace, na linha que o handler registrou:

```
ERROR: Erro não tratado durante a requisição
    err: {
      "type": "Error",
      "message": "Unknown column 'cpf' in field list: SELECT * FROM cidadaos",
      "stack": ...
    }
```

**Esta é a aula inteira em uma tela.** A mesma informação, em dois lugares diferentes: frase
genérica para quem está fora, detalhe completo para quem opera o sistema. 🔴 → 🟢

Dispare também a **2** e a **3**. As duas agora respondem no formato da API:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Endereço não encontrado: GET /endereco-que-nao-existe"
}
```

E a **4**, o caminho feliz, precisa responder **exatamente como antes**. Tratamento de erro
que muda o caminho feliz é tratamento de erro quebrado.

---

## 🧩 Capítulo 5: O 404 no mesmo formato

Antes desta aula, um endereço inexistente devolvia o formato do Fastify. Agora devolve o
nosso. Compare o que o cidadão recebe ao errar o endereço:

| Antes                                   | Agora                                            |
| :-------------------------------------- | :----------------------------------------------- |
| `"message": "Route GET:/xyz not found"` | `"message": "Endereço não encontrado: GET /xyz"` |

A diferença parece pequena e não é. Sem o `notFoundHandler`, a API teria **dois** formatos de
erro conviventes: um para 404 e outro para todo o resto. Quem consome a API precisaria
escrever dois trechos de código para ler erro, e descobriria isso do jeito ruim — em
produção, com o sistema já integrado.

> [!NOTE]
> **Por que a mensagem repete o endereço que a pessoa pediu?**
>
> Porque essa é a única informação da resposta que **não é nossa**: foi ela quem acabou de
> digitar. Devolver o que a pessoa enviou não revela nada, e ajuda muito a encontrar um erro
> de digitação. A régua continua a mesma — o que não pode sair é o que está do lado de
> dentro.

### A requisição 3 merece um parágrafo

Você disparou `POST /health`, e a rota `/health` **existe**. Mesmo assim, a resposta foi 404.

Isso não é bug. Para o Fastify, uma rota é a dupla **método + caminho**. `GET /health` existe;
`POST /health` não existe, do mesmo jeito que `/xyz` não existe. Um caminho certo com o método
errado é tão inexistente quanto um caminho inventado — e é bom que seja assim, porque a
alternativa seria a API aceitar `POST` em rota que só sabe responder `GET`.

---

## 🧹 Capítulo 6: Desmontando o andaime

A rota `/vazamento` cumpriu o papel dela: mostrar o problema com os seus próprios olhos, e
depois provar que a solução funciona. Agora ela precisa sair.

**Por que não deixar?** Porque ela é uma rota de mentira dentro de uma API de verdade. Ela
aparece na lista de rotas, alguém a encontra daqui a seis meses e não sabe se pode apagar, e
no dia em que a API for para um servidor ela vai junto — um endereço público que existe só
para quebrar.

**Andaime é para ser montado e desmontado.** Deixar andaime de pé é como deixar a escada
encostada na janela depois que a obra acabou.

### Passo 7: `src/app.ts`, versão final da aula

Apague o bloco da rota temporária. Este é o arquivo completo, como ele fica ao final da
Aula 05:

```typescript
/**
 * App — Montagem da instância do Fastify
 *
 * Este arquivo é responsável por:
 *   1. Criar a instância do Fastify com suas configurações.
 *   2. Registrar o tratamento centralizado de erros.
 *   3. Registrar plugins globais.
 *   4. Registrar as rotas de cada módulo.
 *
 * Separamos a montagem do app (aqui) da inicialização do servidor (`server.ts`)
 * porque são duas responsabilidades diferentes: uma decide quais rotas e
 * plugins a aplicação tem, a outra decide em qual endereço ela atende. Quem
 * monta a aplicação não precisa saber nada sobre rede.
 */

import Fastify from 'fastify'
import type { FastifyInstance } from 'fastify'
import { healthRoutes } from './modules/health/health.routes.ts'
import { errorHandler, notFoundHandler } from './shared/errors/error-handler.ts'

/**
 * Fábrica da aplicação Fastify.
 *
 * @returns Instância do Fastify já configurada, pronta para receber requisições.
 */
export function buildApp(): FastifyInstance {
  const app = Fastify({
    // O Fastify já vem com o Pino, um dos loggers mais rápidos do Node.
    // Deixá-lo ligado nos dá o registro de cada requisição sem escrever uma linha.
    logger: true,
  })

  // Registramos o tratamento de erros ANTES das rotas, de propósito: assim toda
  // rota registrada daqui para baixo já nasce coberta, e ninguém precisa lembrar
  // de tratar erro em cada rota nova.
  app.setErrorHandler(errorHandler)
  app.setNotFoundHandler(notFoundHandler)

  // No Fastify, cada conjunto de rotas é registrado como um plugin. Isso mantém
  // cada módulo isolado: um erro no registro de um não derruba os outros.
  app.register(healthRoutes)

  return app
}
```

### Passo 8: `requisicoes/erros.http`, versão final da aula

A requisição 1 apontava para uma rota que não existe mais. Ela precisa acompanhar. Deixe o
arquivo **exatamente** assim:

```http
# Requisições que provam o tratamento de erros
#
# Cada bloco abaixo é uma requisição que você dispara clicando em
# "Send Request", logo acima dela, com a API rodando (`npm run dev`).
#
# Este arquivo é versionado de propósito. Ele é a memória do time sobre COMO
# se confere que o tratamento de erros continua funcionando — quem chegar
# daqui a um ano não precisa adivinhar quais endereços visitar.

@host = http://localhost:3333

### 1. Endereço que não existe — precisa vir no formato de erro da API
# Esperado: 404, e a mensagem em português começando por "Endereço não encontrado"
GET {{host}}/endereco-que-nao-existe

### 2. Método errado em rota que existe — também é "não encontrado"
# Esperado: 404. Para o Fastify, rota é método + caminho.
POST {{host}}/health

### 3. O caminho feliz — precisa continuar respondendo igual
# Esperado: 200, com status, uptime e ambiente.
GET {{host}}/health
```

> [!NOTE]
> **E a prova do vazamento, some?**
>
> A requisição some, porque a rota que ela chamava era andaime. O que **não** some é a régua:
> nenhuma resposta desta API pode conter nome de tabela, nome de coluna ou trecho de consulta.
>
> Guarde essa frase. Quando a API passar a falar com um banco de dados de verdade, ela vira a
> primeira coisa a conferir a cada rota nova — e aí a prova volta, sobre erro de verdade em
> vez de erro encomendado.

---

## 📄 Capítulo 7: O contrato de erro no README

Você acabou de definir um **contrato**: toda falha desta API responde com os mesmos três
campos, sempre. Quem consome a API — o time do aplicativo, o portal do cidadão, outro órgão —
precisa saber disso para escrever **um** trecho de código que lê erro, em vez de um para cada
situação.

Contrato que não está escrito não é contrato: é hábito. E hábito se perde na primeira troca
de equipe.

Acrescente a seção `## Formato de erro` ao seu `README.md`, depois de `## Rotas`. Este é o
arquivo completo, como ele fica ao final da Aula 05:

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

| Variável   | O que controla                                  | Padrão        |
| :--------- | :---------------------------------------------- | :------------ |
| `NODE_ENV` | Ambiente: `development`, `test` ou `production` | `development` |
| `PORT`     | Porta em que a API escuta                       | `3333`        |
| `HOST`     | Endereço de rede que a API aceita               | `0.0.0.0`     |

O `.env` **não** é versionado: ele guarda os valores reais de cada máquina, incluindo os
segredos. O `.env.example` é versionado e serve de modelo — ele nunca contém senha de verdade.

## Comandos

| Comando                | O que faz                                         |
| :--------------------- | :------------------------------------------------ |
| `npm run dev`          | Sobe a API recarregando a cada alteração salva    |
| `npm run build`        | Compila o TypeScript para a pasta `dist`          |
| `npm start`            | Executa a versão compilada, como roda em produção |
| `npm run lint`         | Procura problemas de lógica e qualidade no código |
| `npm run lint:fix`     | Corrige sozinho o que for corrigível              |
| `npm run format`       | Formata todos os arquivos com o Prettier          |
| `npm run format:check` | Confere a formatação sem alterar nada             |
| `npm run check`        | Roda lint, formatação e build em sequência        |

## Rotas

| Método | Rota      | O que devolve                              |
| :----- | :-------- | :----------------------------------------- |
| `GET`  | `/health` | O estado da API: status, uptime e ambiente |

As requisições de exemplo de cada rota estão em `requisicoes/`, prontas para disparar pelo
VS Code com a extensão REST Client.

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
````

---

## 💾 Fechando o ciclo: mande para o GitHub

Hoje você criou `src/shared/errors/` com dois arquivos, criou `requisicoes/erros.http`,
alterou o `app.ts` e documentou o contrato de erro no README. Feche o ciclo da Aula 02:

```bash
git add .
git commit -m "feat: adiciona tratamento centralizado de erros"
git push
```

Confira no GitHub, no navegador, que a pasta `src/shared/errors/` chegou lá com os dois
arquivos.

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Precisa terminar sem nenhum erro. É o comando que resume, em uma resposta só, se o projeto
está em ordem.

### 2. As três requisições do `.http` respondem o esperado

Com `npm run dev` rodando, abra `requisicoes/erros.http` e dispare as três, em ordem:

| #   | Requisição                     | Código esperado | O que precisa vir na `message`          |
| --- | :----------------------------- | :-------------: | :-------------------------------------- |
| 1   | `GET /endereco-que-nao-existe` |     **404**     | `Endereço não encontrado: GET /...`     |
| 2   | `POST /health`                 |     **404**     | `Endereço não encontrado: POST /health` |
| 3   | `GET /health`                  |     **200**     | _(não é erro — traz status e uptime)_   |

### 3. Nenhuma resposta está em inglês

Antes desta aula, o 404 vinha com `"Route GET:/xyz not found"`. Se você ainda vê essa frase, o
`setNotFoundHandler` não foi registrado.

### 4. A `/health` continua igual

`http://localhost:3333/health` precisa responder exatamente como antes da aula. Tratamento de
erro que muda o caminho feliz é tratamento de erro quebrado.

### 5. A rota `/vazamento` não existe mais

`http://localhost:3333/vazamento` precisa devolver **404**, no formato da API. Se ela ainda
devolver 500, o Passo 7 não foi feito — e você está prestes a mandar um andaime para o GitHub.

### 6. O README documenta o contrato

Abra o seu `README.md`: ele precisa ter a seção `## Formato de erro`, e o JSON de exemplo
dela precisa ser igual ao que você acabou de ver na aba de resposta. Se os dois divergirem,
quem manda é a resposta de verdade — corrija o README.

---

## 🚨 Erros Comuns

**"Property 'statusCode' does not exist on type 'Error'"**
Você usou `error.statusCode` sem antes verificar `error instanceof AppError`. O TypeScript
está certo: um `Error` comum não tem esse campo. A verificação não é burocracia — é ela que
diz ao compilador (e a você) com qual tipo de erro está lidando.

**A resposta continua mostrando `cidadaos` depois do Passo 5**
Duas causas possíveis, nesta ordem: você não reiniciou o `npm run dev` depois de salvar, ou as
duas linhas `app.setErrorHandler` / `app.setNotFoundHandler` ficaram **depois** do
`app.register`. Elas precisam vir antes.

**`Cannot find module './app-error.ts'`**
Falta a extensão `.ts` no import, ou o arquivo está em outra pasta. Neste projeto, **todo
import é relativo e leva `.ts`** — inclusive entre arquivos da mesma pasta.

**O 404 voltou ao formato antigo**
O `setNotFoundHandler` não foi registrado, ou foi registrado dentro de um plugin. Ele precisa
estar direto na instância principal, no `buildApp()`.

**"Meu erro esperado está saindo como 500"**
Você lançou `new Error(...)` em vez de `new AppError(...)`. Essa é justamente a proteção
funcionando: o handler só confia na mensagem quando ela vem marcada como nossa.

**O botão "Send Request" não aparece no `.http`**
A extensão REST Client não está instalada, ou o arquivo não foi salvo com a extensão `.http`.
O VS Code decide o que fazer com o arquivo pelo nome dele.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/05-gabarito.md`](./exercicios/05-gabarito.md).

**1. Prove que o log recebe o que a resposta esconde**
Crie de novo, temporariamente, a rota `/vazamento` no `app.ts`. Suba com `npm run dev` e
dispare a requisição pelo `.http`. Compare o que aparece na **aba de resposta** com o que
aparece no **terminal**. Onde está o stack trace? Onde está o nome da tabela? Apague a rota
depois.

**2. Um erro esperado de verdade**
Crie uma rota `GET /protocolo/:numero` que lance `new AppError('Protocolo não encontrado', 404)`
quando o número for `0`, e devolva `{ numero }` em qualquer outro caso. Acrescente as duas
requisições ao `erros.http` e dispare as duas. A mensagem apareceu inteira? Por quê? Apague a
rota e as requisições depois.

**3. O código que você não escolheu**
Lance um `new AppError('Acesso negado', 403)` em uma rota temporária. Qual valor veio no campo
`error` da resposta? Você escreveu essa palavra em algum lugar? O que isso diz sobre o
`STATUS_CODES` do Node?

**4. Investigue o 2º caso**
Acrescente ao `erros.http` uma requisição `POST` para `/health` com o cabeçalho
`Content-Type: application/json` e um corpo em JSON quebrado: `{ "isto": "não fecha"`. Qual
código voltou? A mensagem saiu inteira ou foi engolida? Em qual dos três casos do
`errorHandler` ela caiu — e por que ela é segura mesmo saindo inteira?

**5. Pergunta para responder por escrito**
Por que o handler global foi registrado **antes** das rotas no `buildApp()`? O que aconteceria
se, daqui a um ano, alguém criasse um módulo novo e esquecesse de tratar erros dentro dele?

---

## 🎯 Resumo e Próximos Passos

Hoje a API parou de contar coisas que não deve.

O que ficou pronto:

- Toda falha passa por **um** lugar só, e nenhuma rota precisa se defender sozinha.
- O detalhe interno vai para o log e **nunca** para a resposta — em qualquer ambiente.
- Erro esperado e erro inesperado ficaram separados por um critério claro: quem escreveu a
  mensagem.
- A API inteira responde erro em um formato único, inclusive o 404.
- As requisições que provam isso ficaram versionadas, em `requisicoes/erros.http`.

E um método que vale muito além desta aula: **escreva a exigência antes**. Ver o projeto
reprovar é o que prova que a sua exigência está conferindo alguma coisa de verdade. Uma
verificação que nunca reprovou é só um ritual.

**E agora?**

Repare no que ainda não temos. A API se defende bem quando **ela** falha — mas continua
aceitando qualquer coisa que chegue de fora, e continua devolvendo o objeto que o código
montar, campo por campo, sem ninguém conferir.

Na **Aula 06** o Zod vai fechar as duas pontas: **validar o que entra** e **declarar o que
sai**. E a validação de entrada cai, de graça, no handler que você acabou de escrever — quando
ela falhar, o erro já vai sair no formato certo, sem uma linha nova.

Até a próxima! 🚀
