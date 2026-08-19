# 🛡️ Aula 06: Validação de Entrada e Contrato de Resposta

Bem-vindos à **Aula 06**! 🎉

Na Aula 05 vocês fizeram a API parar de contar o que não deve **quando algo dá errado**. Hoje
vamos cuidar das duas portas da API quando tudo dá certo: **o que entra** e **o que sai**.

São problemas opostos, e a mesma ferramenta resolve os dois — o Zod, que vocês já conhecem da
Aula 04, onde ele validou as variáveis de ambiente.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar por que uma API nunca deve confiar no que chega de fora.
- Declarar o **contrato de resposta** de uma rota e provar que campo não declarado não sai.
- Separar a rota de **vida** da rota de **prontidão**, e justificar por que são duas.
- Validar o que entra em uma rota, e ver a requisição errada ser recusada **antes** do seu
  código rodar.
- Fazer toda mensagem de erro de validação sair em português.

## 📋 Pré-requisitos

- Aulas 01 a 05 concluídas.
- `npm run check` passando no seu projeto.
- A extensão **REST Client**, que você já usou na Aula 05.

---

## 💣 Capítulo 1: O vazamento que ninguém autorizou

Vamos começar vendo o problema acontecer.

Abra `src/modules/health/health.service.ts` e acrescente **uma linha** ao objeto devolvido —
imagine que foi um colega, apressado, tentando facilitar uma investigação:

```typescript
  getStatus(): HealthStatus {
    return {
      status: 'ok',
      uptime: process.uptime(),
      timestamp: new Date().toISOString(),
      environment: process.env.NODE_ENV ?? 'development',
      caminhoDoServidor: process.cwd(), // ← a linha nova
    }
  }
```

O TypeScript vai reclamar, porque a interface `HealthStatus` não tem esse campo. Acrescente
lá também, para o exemplo funcionar:

```typescript
export interface HealthStatus {
  status: string
  uptime: number
  timestamp: string
  environment: string
  caminhoDoServidor: string // ← só para o experimento
}
```

Suba a API e chame a rota — use a requisição `GET /health` que já está no seu
`requisicoes/health.http`:

```bash
npm run dev
```

A resposta agora inclui o caminho completo da pasta onde a API está rodando no servidor:

```json
{
  "status": "ok",
  "uptime": 12.34,
  "timestamp": "2026-08-19T17:00:00.000Z",
  "environment": "development",
  "caminhoDoServidor": "C:/projeto/curso_api"
}
```

**Pare e pense no que acabou de acontecer.** Ninguém decidiu publicar essa informação.
Ninguém revisou. Ninguém foi avisado. Bastou alguém acrescentar um campo em um arquivo
interno, e ele saiu pela internet.

Em um sistema de verdade, esse campo poderia ser o nome do servidor, a versão de uma
biblioteca com falha conhecida, ou o caminho de um arquivo de configuração. Cada um deles é
uma peça que ajuda quem está tentando invadir.

> [!IMPORTANT]
> **A ideia central desta aula:** o problema não é o campo. É o fato de a resposta ser
> **tudo o que o código devolver**, em vez de ser **exatamente o que foi combinado**.
>
> Uma proteção que depende de todo mundo lembrar já falhou. Só ainda não sabemos quando.

**Desfaça as duas alterações** antes de continuar — apague a linha do `return` e a linha da
`interface`. Pare o servidor com `Ctrl + C`.

---

## 📖 Capítulo 2: Duas portas, dois problemas

Toda API tem duas portas, e as ameaças de cada uma são diferentes:

| Porta          | Pergunta                                  | O que acontece se ninguém cuidar                                            |
| :------------- | :---------------------------------------- | :-------------------------------------------------------------------------- |
| **Entrada** 📥 | O que essa pessoa está mandando para mim? | Seu código roda com dado que ele não esperava, e quebra de formas criativas |
| **Saída** 📤   | O que estou devolvendo para essa pessoa?  | Informação interna vaza sem ninguém decidir — o que acabamos de ver         |

O que vamos fazer é declarar, para cada rota, **o formato do que ela aceita e o formato do
que ela devolve**. Esse par de declarações tem um nome: **contrato**.

### A analogia da alfândega

Pense na sua API como um país, e no schema como a alfândega.

Sem alfândega, entra o que quiser e sai o que quiser — inclusive coisas que ninguém pretendia
exportar. Com alfândega, existe uma lista do que pode passar em cada direção. O que não está
na lista **não passa**, e ninguém precisa ficar vigiando bagagem por bagagem.

O detalhe que importa: a alfândega não pergunta se você teve boas intenções. Ela confere a
lista.

---

## 📦 Capítulo 3: Instalando a ponte entre o Fastify e o Zod

O Fastify entende schema no formato JSON Schema. Os nossos são escritos em Zod. Falta uma
ponte:

```bash
npm install fastify-type-provider-zod
```

Repare que **não** usamos `-D`. Esta biblioteca é usada quando a API está no ar, validando
requisição de verdade: ela é dependência de produção, como o Fastify e o Zod.

> [!NOTE]
> **Por que não instalamos o Zod agora?** Porque ele já está no projeto desde a Aula 04. A
> mesma ferramenta que garante que a configuração está correta na partida vai garantir, agora,
> que cada requisição está correta. Uma ferramenta a menos para aprender.

Seu `package.json` fica assim — a única linha nova é a da biblioteca em `dependencies`:

```json
{
  "name": "curso_api",
  "version": "1.0.0",
  "description": "API RESTful backend do curso",
  "main": "dist/server.js",
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "dev": "tsx watch --env-file=.env src/server.ts",
    "prebuild": "node --eval \"require('node:fs').rmSync('dist', { recursive: true, force: true })\"",
    "build": "tsc",
    "start": "node --env-file-if-exists=.env dist/server.js",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "check": "npm run lint && npm run format:check && npm run build"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "fastify": "^5.12.0",
    "fastify-type-provider-zod": "^7.0.0",
    "zod": "^4.4.3"
  },
  "devDependencies": {
    "@eslint/js": "^10.0.1",
    "@types/node": "^26.2.0",
    "eslint": "^10.8.1",
    "eslint-config-prettier": "^10.1.8",
    "prettier": "^3.9.6",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0"
  }
}
```

_(Os números das versões podem variar um pouco — o npm instala o mais recente dentro da faixa
permitida. O que precisa bater são os **nomes** dos pacotes e os **scripts**.)_

> [!TIP]
> Repare em `dependencies` × `devDependencies`. Três coisas rodam no servidor de produção: o
> **Fastify**, que atende as requisições, o **Zod**, que valida, e a **ponte** entre os dois.
> Todo o resto — compilador, linter, formatador — é ferramenta nossa, de desenvolvimento.
>
> Essa separação não é burocracia: é o que mantém o que vai para o servidor pequeno e com
> menos superfície de ataque.

---

## 🇧🇷 Capítulo 4: As mensagens em português

Antes de escrever o primeiro schema, um ajuste pequeno com efeito grande.

O Zod escreve as mensagens de erro em inglês:

```
Invalid input: expected number, received NaN
```

Nossa API responde em português. Se metade das mensagens sair em inglês, quem consome a API
precisa lidar com duas línguas — e o cidadão do outro lado, se a mensagem chegar até a tela
dele, não entende nenhuma delas.

O Zod 4 já vem com tradução pronta. Crie o arquivo `src/shared/validation/zod-locale.ts`:

```typescript
/**
 * Mensagens de validação em português
 *
 * O Zod escreve as mensagens de erro em inglês por padrão: _"Invalid input:
 * expected number, received NaN"_. Como as respostas desta API são em português,
 * uma mensagem em inglês faria a API responder em duas línguas — dependendo de
 * quem gerou o erro, o que é confuso para quem consome.
 *
 * O Zod 4 traz traduções prontas. Configurar a nossa custa uma linha, e vale
 * para **toda** validação do projeto: a das variáveis de ambiente e a de cada
 * requisição que chega.
 *
 * Antes:  Invalid input: expected number, received NaN
 * Depois: Tipo inválido: esperado número, recebido NaN
 */

import { z } from 'zod'

/**
 * Aplica o idioma português às mensagens de erro do Zod.
 *
 * Precisa rodar **antes** da primeira validação. O projeto tem dois pontos de
 * entrada — a leitura das variáveis de ambiente, que acontece na partida, e a
 * montagem da API — e esta função é chamada nos dois. Chamar mais de uma vez não
 * causa problema: a última configuração simplesmente substitui a anterior.
 */
export function configurarMensagensEmPortugues(): void {
  z.config(z.locales.pt())
}
```

Agora ligue essa configuração no primeiro lugar que valida alguma coisa: a leitura das
variáveis de ambiente. Abra `src/shared/env/index.ts` e deixe **exatamente** assim:

```typescript
/**
 * Configuração validada da aplicação
 *
 * Este é o **único** lugar do projeto autorizado a ler `process.env`. Qualquer
 * outro arquivo importa o objeto `env` daqui, já validado e já tipado.
 *
 * A validação acontece no momento do import, ou seja, durante a partida da
 * aplicação. Se algo estiver errado, a API **não sobe**.
 *
 * Isso se chama "falhar rápido" (fail fast) e é uma decisão deliberada: é muito
 * melhor a aplicação recusar a partida com uma mensagem clara, aos olhos de quem
 * está fazendo o deploy, do que subir com configuração errada e quebrar depois,
 * durante a requisição de um cidadão.
 */

import { configurarMensagensEmPortugues } from '../validation/zod-locale.ts'
import { envSchema } from './env.schema.ts'

// Precisa vir antes do `safeParse`: é o que faz a mensagem de configuração
// inválida sair em português, e não em inglês.
configurarMensagensEmPortugues()

const resultado = envSchema.safeParse(process.env)

if (!resultado.success) {
  // Usamos `console.error` aqui, e apenas aqui, porque neste ponto o logger do
  // Fastify ainda não existe: o app só é montado depois que a configuração é
  // lida. É a exceção que confirma a regra, e por isso ela vem documentada.
  // eslint-disable-next-line no-console -- o logger ainda não existe nesta etapa da partida
  console.error('\n❌ Configuração inválida. A API não foi iniciada.\n')

  for (const problema of resultado.error.issues) {
    const variavel = problema.path.join('.')
    const valorRecebido = process.env[variavel]

    // eslint-disable-next-line no-console -- o logger ainda não existe nesta etapa da partida
    console.error(
      `   ${variavel}: ${problema.message}` +
        (valorRecebido === undefined ? '' : ` (recebido: "${valorRecebido}")`),
    )
  }

  // eslint-disable-next-line no-console -- o logger ainda não existe nesta etapa da partida
  console.error('\n   Confira o seu arquivo .env. O modelo está em .env.example.\n')

  process.exit(1)
}

/**
 * Configuração da aplicação, validada e pronta para uso.
 *
 * Diferente de `process.env`, aqui os tipos são reais: `env.PORT` é `number`, não
 * `string | undefined`. Isso elimina a checagem manual que alguém sempre esquece
 * de fazer em algum canto do código.
 */
export const env = resultado.data
```

**Ganho imediato:** aquele erro da Aula 04, quando você digitou `8O80` com a letra O, agora
sai em português. Se quiser conferir, é só repetir o experimento de lá.

---

## 📜 Capítulo 5: Declarando o contrato de saída

Chegou o momento principal.

Crie o arquivo `src/modules/health/health.schema.ts`:

```typescript
/**
 * Contratos de resposta do Health Check
 *
 * Um schema declarado aqui faz duas coisas ao mesmo tempo, e a segunda é a que
 * mais importa:
 *
 *   1. Acelera a serialização. O Fastify compila um serializador específico para
 *      este formato, em vez de usar o `JSON.stringify` genérico.
 *   2. **Remove da resposta todo campo que não está declarado.** Se alguém
 *      acrescentar um dado ao retorno do service — a versão do Node, o caminho
 *      de um diretório, o nome do servidor —, ele não sai por acidente.
 *
 * O segundo ponto é o que transforma "tome cuidado com o que você devolve" em
 * uma garantia que não depende de ninguém tomar cuidado.
 */

import { z } from 'zod'

/**
 * Resposta de `GET /health` — a rota de vida.
 *
 * Deliberadamente mínima. Quem consulta esta rota é o monitoramento externo, a
 * cada poucos segundos, e a única pergunta que ele faz é "a API responde?".
 * Uptime e ambiente são informação de dentro de casa: ficam na rota de prontidão.
 */
export const healthResponseSchema = z.object({
  status: z.literal('ok'),
})

/**
 * Resposta de `GET /health/ready` — a rota de prontidão.
 *
 * Detalhada, para quem opera a API. Quando houver banco de dados, é aqui que
 * entra a checagem de conexão: "estou de pé" e "estou pronto para atender" são
 * perguntas diferentes, e a segunda é a que decide se o tráfego pode chegar.
 */
export const readinessResponseSchema = z.object({
  status: z.literal('ok'),

  /** Há quantos segundos este processo está no ar. */
  uptime: z.number(),

  /** Momento em que a resposta foi gerada, no padrão ISO 8601. */
  timestamp: z.string(),

  /** Ambiente em que a API está rodando: development, test ou production. */
  environment: z.enum(['development', 'test', 'production']),
})

/** Formato da resposta da rota de vida, derivado do schema acima. */
export type HealthResponse = z.infer<typeof healthResponseSchema>

/** Formato da resposta da rota de prontidão, derivado do schema acima. */
export type ReadinessResponse = z.infer<typeof readinessResponseSchema>
```

**Duas coisas merecem atenção:**

**1. `z.literal('ok')` em vez de `z.string()`.** Não estamos dizendo "o status é um texto".
Estamos dizendo "o status é exatamente `ok`". Quando existir um estado diferente — por
exemplo, `degradado` —, ele será acrescentado ao contrato de propósito, e não por acidente.

**2. `z.infer` no fim.** Repare que **não** escrevemos a interface `HealthStatus` à mão. O
tipo é derivado do schema. Isso elimina a possibilidade clássica de o tipo e a validação
discordarem: existe uma definição só, e o TypeScript lê dela.

---

## 🔀 Capítulo 6: Duas rotas, duas perguntas diferentes

Hoje o `/health` responde tudo para todo mundo. Vamos separar.

| Rota            | Pergunta                            | Quem consulta         | O que devolve                     |
| :-------------- | :---------------------------------- | :-------------------- | :-------------------------------- |
| `/health`       | A API está **de pé**?               | Monitoramento externo | Só `status`                       |
| `/health/ready` | A API está **pronta para atender**? | Quem opera o sistema  | Status, uptime, momento, ambiente |

**Por que isso não é frescura:** quem monitora consulta a rota a cada poucos segundos, e a
única coisa que ele faz com a resposta é decidir entre "responde" e "não responde". Entregar
junto o tempo de vida do processo e o ambiente é dar informação de graça para quem estiver
observando de fora, sem ganho nenhum para quem precisa dela.

E há uma diferença conceitual que aparece mais adiante: um processo pode estar **de pé** e
ainda assim **não pronto** — por exemplo, subiu mas ainda não conseguiu falar com o banco de
dados. Quem decide se o tráfego pode chegar precisa da segunda resposta, não da primeira.

### Passo 1: O service, com um método para cada pergunta

Deixe `src/modules/health/health.service.ts` **exatamente** assim:

```typescript
/**
 * HealthService
 *
 * Concentra a lógica de negócio da funcionalidade de Health Check (checagem de
 * saúde). É ele quem sabe COMO montar a resposta; o controller apenas pede.
 *
 * Separar a lógica (aqui) da camada HTTP (controller) mantém cada uma com uma
 * responsabilidade só: esta classe não sabe que existe HTTP, e é por isso que
 * uma mudança na forma de responder não a alcança.
 */

import { env } from '../../shared/env/index.ts'
import type { HealthResponse, ReadinessResponse } from './health.schema.ts'

export class HealthService {
  /**
   * Responde à pergunta "a API está de pé?".
   *
   * Devolve o mínimo possível, de propósito: esta é a resposta que o
   * monitoramento externo consulta a cada poucos segundos, e ele não precisa
   * saber mais nada para decidir se a API responde.
   *
   * @returns Apenas o status da aplicação.
   */
  getStatus(): HealthResponse {
    return { status: 'ok' }
  }

  /**
   * Responde à pergunta "a API está pronta para atender?".
   *
   * Estar de pé e estar pronto são coisas diferentes: um processo pode ter
   * subido e ainda não conseguir falar com o banco de dados. Quando houver
   * banco, é aqui que essa checagem entra.
   *
   * @returns Status, há quanto tempo a API está no ar, o momento da resposta e
   *          em qual ambiente ela está rodando.
   */
  getReadiness(): ReadinessResponse {
    return {
      status: 'ok',

      // `process.uptime()` devolve há quantos segundos este processo está no ar.
      // Ferramentas de monitoramento usam esse número para detectar quando a API
      // está reiniciando sozinha em looping.
      uptime: process.uptime(),

      timestamp: new Date().toISOString(),

      // Saber em qual ambiente a resposta foi gerada evita a confusão clássica de
      // investigar um problema em produção olhando para a instância de homologação.
      // O valor vem da configuração já validada, e não de `process.env` direto.
      environment: env.NODE_ENV,
    }
  }
}
```

Repare que a interface `HealthStatus` **sumiu** deste arquivo. Ela virou o tipo derivado do
schema, importado de `health.schema.ts`. Uma definição só.

### Passo 2: O controller, com um handler para cada rota

Deixe `src/modules/health/health.controller.ts` assim:

```typescript
/**
 * HealthController
 *
 * Recebe a requisição HTTP, pede o trabalho ao service e devolve a resposta.
 *
 * Regra de ouro do projeto: o controller NUNCA contém lógica de negócio.
 * Nada de cálculo, nada de `if` de regra. Ele só faz três coisas:
 *   1. Recebe a requisição
 *   2. Chama o service
 *   3. Devolve a resposta
 */

import type { FastifyReply, FastifyRequest } from 'fastify'
import type { HealthService } from './health.service.ts'

export class HealthController {
  /**
   * Recebe o service pronto, por parâmetro do construtor, em vez de criá-lo aqui
   * dentro. Isso se chama injeção de dependência: quem cria o controller é quem
   * decide qual service ele recebe, e trocar essa peça não exige abrir esta classe.
   */
  constructor(private readonly healthService: HealthService) {}

  /**
   * Responde à requisição `GET /health` — a rota de vida.
   *
   * @param _request Requisição recebida. O prefixo `_` marca que não a usamos
   *                 nesta rota — ela não recebe parâmetro nenhum.
   * @param reply    Objeto usado para devolver a resposta ao cliente.
   */
  async handle(_request: FastifyRequest, reply: FastifyReply): Promise<FastifyReply> {
    const status = this.healthService.getStatus()

    // Declaramos o 200 explicitamente. O Fastify assumiria 200 sozinho, mas
    // deixar escrito torna a intenção óbvia para quem ler o código depois.
    return reply.status(200).send(status)
  }

  /**
   * Responde à requisição `GET /health/ready` — a rota de prontidão.
   *
   * @param _request Requisição recebida, também não usada aqui.
   * @param reply    Objeto usado para devolver a resposta ao cliente.
   */
  async handleReadiness(_request: FastifyRequest, reply: FastifyReply): Promise<FastifyReply> {
    const readiness = this.healthService.getReadiness()

    return reply.status(200).send(readiness)
  }
}
```

### Passo 3: As rotas, agora com contrato

Deixe `src/modules/health/health.routes.ts` assim:

```typescript
/**
 * Health Routes
 *
 * Define as rotas da funcionalidade de Health Check e as liga ao controller.
 *
 * No Fastify, um arquivo de rotas é um plugin: uma função que recebe a instância
 * do app e registra os caminhos dentro dela.
 *
 * A partir da declaração de `schema` em cada rota, o Fastify passa a garantir o
 * contrato de saída: campo que não está no schema não sai na resposta, mesmo que
 * o service o devolva.
 */

import type { FastifyInstance } from 'fastify'
import type { ZodTypeProvider } from 'fastify-type-provider-zod'
import { HealthController } from './health.controller.ts'
import { healthResponseSchema, readinessResponseSchema } from './health.schema.ts'
import { HealthService } from './health.service.ts'

/**
 * Plugin de rotas do Health Check.
 *
 * @param app Instância do Fastify, entregue automaticamente pelo `app.register()`.
 */
export async function healthRoutes(app: FastifyInstance): Promise<void> {
  // Montamos a "corrente" de dependências à mão: o controller precisa do service,
  // então criamos o service primeiro e o entregamos ao controller.
  // Com um módulo só, fazer isso manualmente é simples e deixa tudo visível.
  const healthService = new HealthService()
  const healthController = new HealthController(healthService)

  // `withTypeProvider` avisa ao TypeScript que os schemas desta instância são
  // escritos em Zod. É o que faz o editor saber, dentro do handler, exatamente
  // qual formato pode ser devolvido — e reclamar antes de rodar, se não for.
  const rotas = app.withTypeProvider<ZodTypeProvider>()

  /**
   * Rota de vida: a API está de pé?
   *
   * Resposta deliberadamente mínima. Esta é a rota que o monitoramento externo
   * consulta a cada poucos segundos.
   */
  rotas.get(
    '/health',
    { schema: { response: { 200: healthResponseSchema } } },
    async (request, reply) => {
      return healthController.handle(request, reply)
    },
  )

  /**
   * Rota de prontidão: a API está pronta para atender?
   *
   * Detalhada, para quem opera a API.
   */
  rotas.get(
    '/health/ready',
    { schema: { response: { 200: readinessResponseSchema } } },
    async (request, reply) => {
      return healthController.handleReadiness(request, reply)
    },
  )
}
```

### Passo 4: Ensinando o Fastify a ler os nossos schemas

Falta ligar a ponte. Deixe `src/app.ts` **exatamente** assim:

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
import { serializerCompiler, validatorCompiler } from 'fastify-type-provider-zod'
import { healthRoutes } from './modules/health/health.routes.ts'
import { errorHandler, notFoundHandler } from './shared/errors/error-handler.ts'
import { configurarMensagensEmPortugues } from './shared/validation/zod-locale.ts'

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

  // No Fastify, cada conjunto de rotas é registrado como um plugin. Isso mantém
  // cada módulo isolado: um erro no registro de um não derruba os outros.
  app.register(healthRoutes)

  return app
}
```

### Passo 5: A prova

Suba a API e chame as duas rotas:

```bash
npm run dev
```

```
GET http://localhost:3333/health
GET http://localhost:3333/health/ready
```

```json
{ "status": "ok" }
```

```json
{
  "status": "ok",
  "uptime": 9.86,
  "timestamp": "2026-08-19T17:06:45.921Z",
  "environment": "development"
}
```

**Agora repita o experimento do Capítulo 1.** Acrescente de volta o `caminhoDoServidor` ao
`getStatus()` — desta vez o TypeScript vai reclamar antes mesmo de você rodar, porque o tipo
vem do schema. Ignore o aviso por um instante e chame `/health` de novo:

```json
{ "status": "ok" }
```

**O campo não saiu.** Não porque alguém lembrou de tirá-lo, mas porque ele não está no
contrato. É essa a diferença entre "tome cuidado" e "não tem como".

Desfaça o experimento antes de seguir.

---

## 📥 Capítulo 7: A outra porta — validando o que entra

O `/health` não recebe nada, então precisamos de uma rota de exemplo para ver a entrada sendo
validada. Vamos criar uma **descartável**, e apagá-la no fim do capítulo.

> [!CAUTION]
> Assim como o `exemplo-bagunca.ts` da Aula 03 e a rota `/vazamento` da Aula 05, esta rota
> existe só para o experimento. Nunca deixe rota de mentira em uma API de verdade.

Abra `src/modules/health/health.routes.ts` e acrescente, **temporariamente**, antes do
fechamento da função:

```typescript
  // ⚠️ TEMPORÁRIA — apagar no fim do capítulo.
  rotas.get(
    '/exemplo/protocolo/:numero',
    {
      schema: {
        params: z.object({ numero: z.coerce.number().int().positive() }),
        response: { 200: z.object({ numero: z.number() }) },
      },
    },
    async (request, reply) => {
      // `request.params.numero` já chega aqui como NÚMERO, e já validado.
      return reply.send({ numero: request.params.numero })
    },
  )
```

Você vai precisar do import do Zod no topo do arquivo, também temporário:

```typescript
import { z } from 'zod'
```

Agora acrescente as quatro requisições ao seu `requisicoes/erros.http`, também
temporariamente, e dispare uma a uma:

```http
### TEMPORÁRIO — experimento do Capítulo 7 da Aula 06
GET {{host}}/exemplo/protocolo/123

### TEMPORÁRIO — "abc" não vira número
GET {{host}}/exemplo/protocolo/abc

### TEMPORÁRIO — é número, mas não é positivo
GET {{host}}/exemplo/protocolo/-5

### TEMPORÁRIO — é número, mas não é inteiro
GET {{host}}/exemplo/protocolo/3.7
```

| Requisição                   | O que acontece                       |
| :--------------------------- | :----------------------------------- |
| `GET /exemplo/protocolo/123` | `200` com `{"numero":123}`           |
| `GET /exemplo/protocolo/abc` | `400` — "abc" não vira número        |
| `GET /exemplo/protocolo/-5`  | `400` — é número, mas não é positivo |
| `GET /exemplo/protocolo/3.7` | `400` — é número, mas não é inteiro  |

**Três coisas para reparar, e a terceira é a mais importante:**

**1. O `z.coerce`.** Tudo que vem no endereço é texto: `"123"`, com aspas. O `coerce` converte
antes de validar — o mesmo recurso que vocês usaram na Aula 04 para a variável `PORT`.

**2. O tipo dentro do handler.** Passe o mouse sobre `request.params.numero` no editor. O
TypeScript diz `number`, não `string` e nem `any`. Ninguém precisou avisá-lo: o tipo veio do
schema.

**3. O seu código não roda com dado inválido.** Quando alguém chama `/exemplo/protocolo/abc`,
a função do handler **nem é executada**. O Fastify recusa antes. Essa é a diferença entre
validar na porta e validar lá dentro: no segundo caso, você já teria dado inválido circulando
pelo seu código.

> [!TIP]
> **Como provar a afirmação 3, e não só acreditar nela.** Acrescente um
> `app.log.info('o handler rodou')` como primeira linha do handler. Chame `/123` — a linha
> aparece no terminal. Chame `/abc` — ela **não** aparece. A recusa aconteceu antes.
>
> É a mesma ideia da Aula 05: não acredite, provoque e olhe.

**Não apague ainda** — o próximo capítulo usa essa rota. Ele avisa quando ela sai.

---

## 🗣️ Capítulo 8: A mensagem que o cliente recebe

Chame a rota inválida e olhe a resposta com atenção:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "params/numero Tipo inválido: esperado número, recebido NaN"
}
```

Está em português — o Capítulo 4 resolveu isso. Mas repare no começo da mensagem:
`params/numero`. Isso é o formato interno da ferramenta, com a barra e o nome técnico da parte
da requisição. Quem consome a nossa API não deveria precisar aprender esse formato.

Vamos melhorar, no lugar onde toda mensagem de erro já passa: o handler da Aula 05.

Abra `src/shared/errors/error-handler.ts` e acrescente, logo antes da função `errorHandler`:

```typescript
/**
 * Nome, em português, de cada parte da requisição que pode ser validada.
 *
 * O Fastify informa qual delas falhou em `validationContext`. Sem esta tradução,
 * a resposta diria "querystring" para quem só quer saber que errou um parâmetro
 * do endereço.
 */
const PARTES_DA_REQUISICAO: Record<string, string> = {
  body: 'no corpo da requisição',
  params: 'no endereço',
  querystring: 'nos parâmetros do endereço',
  headers: 'nos cabeçalhos',
}

/**
 * Monta a mensagem de um erro de validação, campo por campo.
 *
 * As mensagens de cada campo já vêm em português: o `zod-locale.ts` configura o
 * idioma do Zod na partida. Aqui só montamos a frase em volta delas.
 *
 * @param error O erro de validação levantado pelo Fastify.
 * @returns     Uma frase única, legível, com todos os campos que falharam.
 */
function montarMensagemDeValidacao(error: FastifyError): string {
  const parte = PARTES_DA_REQUISICAO[error.validationContext ?? ''] ?? 'na requisição'

  const campos = (error.validation ?? []).map((problema) => {
    // O `instancePath` vem no formato "/numero" ou "/endereco/cidade". Vira
    // "numero" e "endereco.cidade", que é como uma pessoa se refere ao campo.
    const campo = problema.instancePath.replace(/^\//, '').replaceAll('/', '.')

    return campo === '' ? problema.message : `${campo} — ${problema.message}`
  })

  return campos.length === 0
    ? `Dados inválidos ${parte}.`
    : `Dados inválidos ${parte}: ${campos.join('; ')}`
}
```

E, dentro da função `errorHandler`, acrescente um caso **entre** o primeiro e o segundo:

```typescript
  // 2º caso: a requisição não cumpriu o schema declarado na rota. O Fastify
  // recusou antes de o handler rodar — o código da rota nem chegou a executar
  // com dado inválido, que é exatamente o ponto de validar na entrada.
  //
  // Reescrevemos a mensagem porque a original vem no formato da ferramenta
  // ("params/numero Tipo inválido: …"), com a barra e o nome interno da parte da
  // requisição. Quem consome a API não deveria precisar aprender esse formato.
  if (error.code === 'FST_ERR_VALIDATION') {
    return reply.status(400).send(montarResposta(400, montarMensagemDeValidacao(error)))
  }
```

Renumere os comentários dos casos seguintes: o que era o 2º vira o 3º, e o que era o 3º vira
o 4º.

O arquivo completo, ao final desta aula, fica assim:

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
 * Nome, em português, de cada parte da requisição que pode ser validada.
 *
 * O Fastify informa qual delas falhou em `validationContext`. Sem esta tradução,
 * a resposta diria "querystring" para quem só quer saber que errou um parâmetro
 * do endereço.
 */
const PARTES_DA_REQUISICAO: Record<string, string> = {
  body: 'no corpo da requisição',
  params: 'no endereço',
  querystring: 'nos parâmetros do endereço',
  headers: 'nos cabeçalhos',
}

/**
 * Monta a mensagem de um erro de validação, campo por campo.
 *
 * As mensagens de cada campo já vêm em português: o `zod-locale.ts` configura o
 * idioma do Zod na partida. Aqui só montamos a frase em volta delas.
 *
 * @param error O erro de validação levantado pelo Fastify.
 * @returns     Uma frase única, legível, com todos os campos que falharam.
 */
function montarMensagemDeValidacao(error: FastifyError): string {
  const parte = PARTES_DA_REQUISICAO[error.validationContext ?? ''] ?? 'na requisição'

  const campos = (error.validation ?? []).map((problema) => {
    // O `instancePath` vem no formato "/numero" ou "/endereco/cidade". Vira
    // "numero" e "endereco.cidade", que é como uma pessoa se refere ao campo.
    const campo = problema.instancePath.replace(/^\//, '').replaceAll('/', '.')

    return campo === '' ? problema.message : `${campo} — ${problema.message}`
  })

  return campos.length === 0
    ? `Dados inválidos ${parte}.`
    : `Dados inválidos ${parte}: ${campos.join('; ')}`
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

  // 2º caso: a requisição não cumpriu o schema declarado na rota. O Fastify
  // recusou antes de o handler rodar — o código da rota nem chegou a executar
  // com dado inválido, que é exatamente o ponto de validar na entrada.
  //
  // Reescrevemos a mensagem porque a original vem no formato da ferramenta
  // ("params/numero Tipo inválido: …"), com a barra e o nome interno da parte da
  // requisição. Quem consome a API não deveria precisar aprender esse formato.
  if (error.code === 'FST_ERR_VALIDATION') {
    return reply.status(400).send(montarResposta(400, montarMensagemDeValidacao(error)))
  }

  // 3º caso: outro erro que o próprio Fastify levantou sobre a requisição
  // recebida (corpo mal formado, tipo de conteúdo não suportado). A mensagem
  // fala do que o cliente enviou, não do que existe dentro do servidor. Os
  // códigos abaixo de 500 são exatamente os que significam "o problema veio de
  // fora".
  if (error.statusCode !== undefined && error.statusCode < 500) {
    return reply.status(error.statusCode).send(montarResposta(error.statusCode, error.message))
  }

  // 4º caso: qualquer outra coisa. Bug no nosso código, banco fora do ar,
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

Agora a resposta fica assim:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Dados inválidos no endereço: numero — Tipo inválido: esperado número, recebido NaN"
}
```

> [!NOTE]
> **Repare no que NÃO precisou ser feito.** Não criamos um tratamento de erro para a rota
> nova. Não escrevemos `try/catch` em lugar nenhum. O erro de validação caiu no mesmo handler
> que já existia desde a Aula 05, e saiu no mesmo formato de todos os outros erros da API.
>
> Foi exatamente isso que a Aula 05 prometeu. Quando a estrutura está certa, a funcionalidade
> nova chega quase de graça.

### Um limite honesto da tradução automática

Chame `/exemplo/protocolo/-5` e leia a mensagem:

```
Dados inválidos no endereço: numero — Muito pequeno: esperado que number fosse >0
```

Repare no `number`, em inglês, no meio de uma frase em português. A tradução do Zod cobre a
estrutura da mensagem, mas o nome do tipo vem do próprio Zod.

Quando a mensagem automática não for boa o suficiente, escreva a sua no schema:

```typescript
numero: z.coerce.number({ error: 'precisa ser um número' }).int().positive({
  error: 'precisa ser maior que zero',
}),
```

**A regra prática:** deixe a mensagem automática onde ela já resolve, e escreva a sua onde o
cidadão vai ler.

### Agora sim: desmonte o andaime

A rota `/exemplo/protocolo/:numero` cumpriu o papel. **Apague**, nesta ordem:

1. O bloco da rota temporária, em `health.routes.ts`.
2. O `import { z } from 'zod'` no topo daquele arquivo.
3. Os quatro blocos marcados como `### TEMPORÁRIO` no `requisicoes/erros.http`.

Confira com `git diff` que não sobrou nenhum `/exemplo/` no projeto. É a mesma disciplina da
Aula 05: andaime é para ser montado **e desmontado**.

---

## 🧪 Capítulo 9: A prova do contrato, guardada no repositório

O experimento mais importante desta aula é o do Capítulo 1: **um campo que ninguém declarou
não sai na resposta**. Vamos guardar a forma de reconferir isso, para você não depender da
memória daqui a três meses.

Deixe `requisicoes/health.http` **exatamente** assim:

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
# Esperado: 200 com {"status":"ok"} e NADA MAIS.
#
# Este "nada mais" é o contrato de saída em ação. Se você algum dia vir um campo
# a mais aqui, não procure quem o escreveu: procure o `schema` da rota em
# `health.routes.ts`. Ou ele sumiu, ou alguém o alterou.
GET {{host}}/health

### Verificar se a API está pronta para atender (rota de prontidão)
# Esperado: 200 com exatamente quatro campos — status, uptime, timestamp e
# environment. Nem três, nem cinco.
GET {{host}}/health/ready

### Conferir que um endereço inexistente responde no formato de erro da API
# Esperado: 404 com statusCode, error e message, em português.
GET {{host}}/rota-que-nao-existe
```

### O procedimento de conferência do contrato

Este é o roteiro para provar, em um minuto, que a alfândega está de pé. Rode-o sempre que
mexer em um schema:

| #   | O que fazer                                                            | O que precisa acontecer                  |
| --- | :--------------------------------------------------------------------- | :--------------------------------------- |
| 1   | Acrescente `versaoDoNode: process.version` ao retorno de `getStatus()` | O editor sublinha em vermelho na hora    |
| 2   | Acrescente `versaoDoNode: z.string()` ao `healthResponseSchema`        | O vermelho some                          |
| 3   | Suba a API e chame `GET /health`                                       | O campo **aparece** — está no contrato   |
| 4   | Remova a linha do **schema**, mas deixe a do **service**               | O editor volta a reclamar                |
| 5   | Suba a API e chame `GET /health` de novo                               | O campo **sumiu** — não está no contrato |
| 6   | Desfaça tudo                                                           | `git diff` limpo                         |

O passo 5 é a aula inteira. O código continuava devolvendo o campo; o contrato é que decidiu
que ele não sai.

> [!IMPORTANT]
> **Repare no que este procedimento testa: uma ausência.**
>
> É o tipo de coisa que ninguém confere sozinho, porque não dá sintoma. Um campo que vaza não
> quebra nada, não gera erro, não aparece no log. Ele só fica lá, sendo entregue para quem
> pedir, até o dia em que alguém repara.
>
> **Ao conferir segurança, procure a ausência da informação — não a presença do formato.**

---

## 📄 Capítulo 10: Atualizando o README

Quem acrescenta rota, atualiza a documentação. No `README.md`, a seção `## Rotas` agora tem
as duas:

```markdown
## Rotas

| Método | Rota            | O que devolve                                     |
| :----- | :-------------- | :------------------------------------------------ |
| `GET`  | `/health`       | **Vida:** apenas `{ "status": "ok" }`             |
| `GET`  | `/health/ready` | **Prontidão:** status, uptime, momento e ambiente |

Toda rota declara o contrato da resposta com Zod. Campo que não está no contrato **não sai**,
mesmo que o código o devolva por engano.

As requisições de exemplo de cada rota estão em `requisicoes/`, prontas para disparar pelo
VS Code com a extensão REST Client.
```

---

## 💾 Fechando o ciclo: mande para o GitHub

```bash
git add .
git commit -m "feat: valida entrada e declara contrato de resposta com Zod"
git push
```

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Precisa terminar sem nenhum erro.

### 2. As duas rotas respondem o que devem

Com `npm run dev` rodando, dispare pelo `requisicoes/health.http`:

| Rota            | Resposta esperada                   |
| :-------------- | :---------------------------------- |
| `/health`       | `{"status":"ok"}` — **e nada mais** |
| `/health/ready` | Os quatro campos                    |

### 3. O campo extra não vaza

Faça o procedimento de seis passos do Capítulo 9. Se o campo aparecer no passo 5, o schema não
está sendo aplicado — confira se o `setSerializerCompiler` está no `app.ts` e se a rota declara
`schema`.

### 4. A mensagem de validação sai em português e sem barra

Se você ainda vê `params/numero` no começo da mensagem, o caso `FST_ERR_VALIDATION` não foi
acrescentado ao `errorHandler`, ou foi acrescentado **depois** do caso `statusCode < 500` — e
aí aquele o captura antes.

### 5. A rota descartável foi apagada

```bash
git status
git diff
```

Não deve haver nenhuma rota `/exemplo/...` no seu `health.routes.ts`, nenhum bloco
`### TEMPORÁRIO` no `erros.http`, e nenhum `import { z }` sobrando nas rotas.

---

## 🚨 Erros Comuns

**`Cannot find module 'fastify-type-provider-zod'`**
Faltou instalar. Rode `npm install fastify-type-provider-zod` na raiz do projeto.

**O campo extra continua aparecendo na resposta**
Duas causas possíveis: o `app.setSerializerCompiler(serializerCompiler)` não foi adicionado ao
`app.ts`, ou a rota não declara `schema`. Sem os dois, o Fastify volta a devolver o objeto
inteiro.

**`request.params` aparece como `unknown` ou `any` no editor**
Faltou o `withTypeProvider<ZodTypeProvider>()`. Sem ele, o Fastify valida em tempo de execução
mas o TypeScript não sabe o formato.

**A mensagem de erro sai em inglês**
A função `configurarMensagensEmPortugues()` não foi chamada, ou foi chamada depois da
validação. Ela precisa rodar antes — por isso está no `env/index.ts` e no `buildApp()`.

**`z.locales is not a function`**
Sua versão do Zod é anterior à 4. Confira com `npm list zod`; este projeto usa a 4.

**O `health.service.ts` não compila depois do Passo 1**
A interface `HealthStatus` saiu daquele arquivo e virou tipo derivado do schema. Se o editor
ainda a procura, é porque o import de `health.schema.ts` não foi acrescentado.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/06-gabarito.md`](./exercicios/06-gabarito.md).

**1. Descubra o que o contrato não protege**
Troque, no `readinessResponseSchema`, o tipo de `uptime` de `z.number()` para `z.string()`.
Chame `/health/ready`. O que acontece? A API devolve erro, converte o valor ou ignora? Depois
desfaça.

**2. Escreva uma mensagem melhor**
Recrie a rota descartável do Capítulo 7, agora com mensagens próprias no schema (`{ error:
'...' }`). Compare a resposta com a versão automática. Em qual situação vale o trabalho de
escrever a sua?

**3. Duas violações de uma vez**
Na rota descartável, acrescente também uma `querystring` obrigatória. Faça uma requisição que
erre **os dois** ao mesmo tempo. A resposta menciona os dois problemas? Por que o Fastify
parou no primeiro contexto?

**4. O contrato como documentação**
Sem rodar nada, leia apenas `health.schema.ts` e responda: quantos campos a rota de prontidão
devolve, e quais valores `environment` aceita? Compare com o esforço de descobrir isso lendo o
service. É esse o segundo ganho do contrato.

**5. A ordem dos casos importa**
No `errorHandler`, mova o caso `FST_ERR_VALIDATION` para **depois** do caso
`statusCode < 500`. Chame `/exemplo/protocolo/abc`. A mensagem voltou ao formato com barra?
Por quê? O que isso ensina sobre a ordem de uma sequência de `if`?

---

## 🎯 Resumo e Próximos Passos

Hoje a API passou a ter portas com alfândega nos dois sentidos.

O que ficou pronto:

- Toda resposta é **exatamente** o que o contrato declara — campo a mais não sai, mesmo que o
  código o devolva.
- `/health` e `/health/ready` separadas, cada uma respondendo à sua pergunta.
- Entrada validada antes de o seu código rodar, com tipos derivados do mesmo schema.
- Mensagens de validação em português, no formato único de erro da API.
- O procedimento de conferência do contrato, guardado junto das requisições.

E um princípio que vale além desta aula: **prefira a garantia à disciplina**. Toda vez que
você puder escolher entre "todo mundo precisa lembrar de X" e "não tem como esquecer de X", a
segunda opção é a que continua funcionando daqui a um ano, com outra equipe.

**E agora?**

Esta é a última aula desta trilha, e vale olhar para trás: em seis aulas vocês saíram de uma
pasta vazia e chegaram a uma API que sobe, é padronizada por ferramenta, lê configuração
validada, não conta ao mundo o que não deve quando falha, não confia no que chega de fora e
devolve exatamente o que foi combinado.

Isso é um assunto inteiro fechado. A trilha continua depois — os schemas que vocês escreveram
hoje têm um segundo uso, quase de graça: eles descrevem a API inteira, e dá para gerar a
partir deles uma **documentação navegável** que não consegue ficar desatualizada. Depois vêm
as defesas contra quem abusa da API, o empacotamento que faz ela rodar igual em qualquer
máquina, o banco de dados, e a primeira rota que existe para alguém de fora.

Cada um desses degraus resolve uma dor específica. Eles chegam quando estas seis aulas
estiverem firmes — e "firmes" quer dizer que você consegue explicar cada arquivo do seu
projeto para outra pessoa.

Até lá, o melhor exercício é o mais simples: abra o projeto amanhã e leia o código que você
escreveu hoje. Se estiver claro, você escreveu bem.

Parabéns pela trilha concluída! 🚀
