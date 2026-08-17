# 📖 Aula 08: Documentação Viva com Swagger

> **Aula anterior:** [07 — Validação de Entrada e Contrato de Resposta](./07-validacao-e-contrato.md)

Na aula passada você escreveu schemas para duas coisas: impedir que entre lixo e impedir que
saia o que não devia.

Hoje você vai descobrir que aqueles mesmos schemas fazem uma terceira coisa, e que ela vem
**de graça**. Sem escrever contrato de novo. Sem manter dois textos em dia.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Abrir uma página no navegador que mostra **todas** as rotas da sua API, com os campos, os
  tipos e os valores aceitos por cada uma.
- Disparar uma requisição de verdade a partir dessa página, sem `curl`, sem Postman, sem
  arquivo `.http`.
- Entender por que essa documentação **não consegue** ficar desatualizada.
- Decidir, com argumento, se a documentação deve ficar disponível em produção — e escrever
  essa decisão em código, em vez de deixá-la em um comentário.
- Escrever um teste que prova que uma página **não existe**.

---

## 📋 Pré-requisitos

- Aula 07 concluída, com o `npm run check` passando.
- O projeto rodando com `npm run dev`.

Não instale nada ainda. A primeira coisa é sentir o problema.

---

## 💣 Capítulo 1: Descreva a sua API para alguém

Imagine que outro órgão vai consumir a sua API. Chega um e-mail:

> _"Bom dia. Precisamos integrar com a API de vocês. Quais rotas existem, e o que cada uma
> devolve?"_

Pare por um minuto e responda de verdade. Com o projeto do jeito que ele está hoje, quais são
as suas opções?

| Opção                                     | O problema dela                                                                                   |
| :---------------------------------------- | :------------------------------------------------------------------------------------------------ |
| "Abre o código e lê `health.routes.ts`"   | Quem consome a API não tem — e não deveria ter — acesso ao seu código                             |
| Escrever um documento e mandar por e-mail | No dia em que a rota mudar, existem duas verdades: o código e o anexo. E o anexo não muda sozinho |
| Colar um exemplo de resposta na conversa  | Envelhece na primeira alteração, e ninguém avisa a pessoa que recebeu                             |

Repare no que essas três têm em comum: **todas criam uma segunda cópia da verdade.** E cópia
que precisa ser atualizada à mão é cópia que um dia vai estar errada — normalmente no pior
momento, quando alguém a estiver usando para tomar uma decisão.

> [!IMPORTANT]
> A pergunta desta aula não é "como escrever uma boa documentação?". É **"como fazer com que
> exista uma documentação só, que não possa discordar do código?"**

E a resposta já está no seu projeto, escrita na aula passada.

---

## 📖 Capítulo 2: A especificação que o mundo inteiro já combinou

Existe um formato padrão para descrever uma API HTTP. Ele se chama **OpenAPI**, e é um
documento — em JSON ou YAML — que diz, de forma que qualquer ferramenta entende:

- quais caminhos existem (`/health`, `/health/ready`);
- qual método HTTP cada um aceita (`GET`, `POST`…);
- o que precisa ser enviado, e de que tipo;
- o que volta em cada código de resposta, campo por campo.

**Por que isso importa mais do que um documento bem escrito:** OpenAPI é lido por máquina.
A partir de um arquivo OpenAPI, ferramentas geram página de documentação, código de cliente
em várias linguagens, coleção de testes e validação automática de contrato. Um texto em
português, por melhor que seja, não faz nada disso.

### A analogia da planta baixa

Um texto descrevendo uma casa — _"tem três quartos, a cozinha é ampla"_ — serve para uma
conversa. Uma **planta baixa** serve para construir: o eletricista, o encanador e o pedreiro
leem o mesmo desenho e cada um extrai dele o que precisa, sem ligar um para o outro.

OpenAPI é a planta baixa da sua API. E, como você vai ver, ela já está desenhada — só falta
alguém imprimir.

---

## 📦 Capítulo 3: Instalando os dois plugins

```bash
npm install @fastify/swagger @fastify/swagger-ui
```

Saída esperada:

```
added 18 packages, and audited 266 packages in 3s

89 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

### Por que são dois pacotes, e não um

Esta separação vai importar bastante no Capítulo 8, então vale entender agora:

| Pacote                | O que faz                                                   | Publica endereço? |
| :-------------------- | :---------------------------------------------------------- | :---------------- |
| `@fastify/swagger`    | **Gera** o documento OpenAPI a partir dos schemas das rotas | **Não**           |
| `@fastify/swagger-ui` | **Serve** a página bonita e o documento em JSON e YAML      | **Sim**           |

O primeiro é o desenhista: produz a planta e a deixa guardada na memória do processo. O
segundo é a parede onde a planta fica pendurada.

> [!NOTE]
> Isso não é curiosidade de bastidor. Se você instalasse **só** o `@fastify/swagger`, a
> especificação seria gerada e **nenhum endereço existiria** — nem a página, nem o `/json`.
> Testamos: sem o `swagger-ui`, `/documentation/json` responde 404.

---

## 🌉 Capítulo 4: Ligando a documentação ao projeto

Crie a pasta `src/shared/docs/` e, dentro dela, o arquivo `index.ts`.

Repare no lugar: **`shared/`, não `modules/`**. A documentação não pertence a nenhuma
funcionalidade — ela descreve todas. É a mesma lógica que colocou o tratamento de erros e a
leitura das variáveis de ambiente ali.

```typescript
/**
 * Documentação da API (OpenAPI + Swagger UI)
 *
 * Este arquivo não descreve a API à mão. Ele liga duas peças que já existem:
 * os schemas escritos em Zod, que a API usa para validar e serializar, e um
 * formato padrão de descrição de API chamado **OpenAPI**.
 *
 * O resultado é uma documentação que **não pode ficar desatualizada**: ela é
 * gerada a partir do mesmo schema que o servidor usa para responder. Mudou o
 * contrato, mudou a documentação, sem ninguém lembrar de nada.
 *
 * São dois plugins, com trabalhos diferentes:
 *
 *   • `@fastify/swagger`    — GERA a especificação. Não publica endereço nenhum.
 *   • `@fastify/swagger-ui` — SERVE a página e a especificação em texto.
 *
 * Saber de quem é cada endereço importa na hora de desligar: quem publica
 * `/documentation`, `/documentation/json` e `/documentation/yaml` é o segundo.
 */

import fastifySwagger from '@fastify/swagger'
import fastifySwaggerUi from '@fastify/swagger-ui'
import type { FastifyInstance } from 'fastify'
import { jsonSchemaTransform } from 'fastify-type-provider-zod'

/** Endereço em que a documentação fica disponível. */
export const DOCS_ROUTE_PREFIX = '/documentation'

/**
 * Registra a documentação da API.
 *
 * Precisa ser chamada **antes** do registro das rotas. O `@fastify/swagger` não
 * lê o código-fonte: ele escuta o evento que o Fastify dispara a cada rota
 * registrada e vai juntando os schemas. Rota que entra antes dele estar de pé
 * simplesmente não é vista — e o defeito é silencioso, porque a página abre
 * normalmente, só que vazia.
 *
 * @param app Instância do Fastify que vai receber a documentação.
 */
export function registerDocs(app: FastifyInstance): void {
  app.register(fastifySwagger, {
    openapi: {
      info: {
        title: 'API do Curso',
        description:
          'API de atendimento ao cidadão. Esta página é gerada a partir dos schemas ' +
          'do próprio código: o que está aqui é exatamente o que a API aceita e devolve.',
        version: '1.0.0',
      },
      tags: [
        {
          name: 'saude',
          description: 'Rotas de monitoramento, consultadas por quem observa a API',
        },
        { name: 'interna', description: 'Rotas de uso interno da equipe que opera a API' },
      ],
    },

    // Esta linha é a ponte. Os nossos schemas são escritos em Zod, e o formato
    // OpenAPI é outro; o `jsonSchemaTransform` traduz um no outro. Sem ela, o
    // plugin não entenderia nenhum dos schemas do projeto.
    transform: jsonSchemaTransform,
  })

  app.register(fastifySwaggerUi, {
    routePrefix: DOCS_ROUTE_PREFIX,
  })
}
```

### A linha que faz a mágica

```typescript
transform: jsonSchemaTransform,
```

Os seus schemas são escritos em **Zod**. O formato OpenAPI é outro. O `jsonSchemaTransform`
— que vem da `fastify-type-provider-zod`, o pacote que você já instalou na Aula 07 — traduz
um no outro.

Sem essa linha, o plugin registraria as rotas e não saberia dizer **nada** sobre elas.

---

## 🔌 Capítulo 5: Ligando no `app.ts`

Agora o `app.ts` precisa chamar a função. E precisa chamá-la **no lugar certo**.

Abra `src/app.ts` e deixe-o exatamente assim:

```typescript
/**
 * App — Montagem da instância do Fastify
 *
 * Este arquivo é responsável por:
 *   1. Criar a instância do Fastify com suas configurações.
 *   2. Registrar o tratamento centralizado de erros.
 *   3. Registrar plugins globais.
 *   4. Registrar a documentação da API, quando ela deve existir.
 *   5. Registrar as rotas de cada módulo.
 *
 * A ordem em que essas coisas acontecem não é decorativa: o tratamento de erros
 * e a documentação precisam estar de pé ANTES das rotas para valerem para elas.
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
import { configurarMensagensEmPortugues } from './shared/validation/zod-locale.ts'

/**
 * Opções de montagem da aplicação.
 */
export interface BuildAppOptions {
  /**
   * Liga ou desliga o registro de eventos (logger).
   *
   * O padrão é ligado. Nos testes automatizados passamos `false`: sem isso, cada
   * requisição simulada imprimiria várias linhas de log e o resultado dos testes
   * ficaria impossível de ler.
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
    // O Fastify já vem com o Pino, um dos loggers mais rápidos do Node.
    // Deixá-lo ligado nos dá o registro de cada requisição sem escrever uma linha.
    logger: options.logger ?? true,
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

### A ordem não é decoração

Esta é a parte da aula que vale guardar para o resto da carreira:

```typescript
registerDocs(app) // ← precisa vir ANTES
app.register(healthRoutes) // ← das rotas
```

O `@fastify/swagger` **não lê o seu código-fonte**. Ele fica escutando um aviso que o Fastify
dispara toda vez que uma rota é registrada, e vai anotando o que passa.

Se as rotas subirem primeiro, ele não escuta nada.

E agora repare no tipo de falha que isso produz: **nenhum erro.** Nenhuma mensagem vermelha.
A página abre normalmente, com o título certo — e vazia. Você olharia para ela, veria que "a
página está lá", e não perceberia.

> [!WARNING]
> Guarde este padrão: **o defeito mais perigoso não é o que quebra, é o que continua
> parecendo certo.** No Capítulo 9 você vai escrever um teste cuja única função é impedir
> exatamente isso.

---

## 👀 Capítulo 6: Veja o que você já tinha

Suba a API:

```bash
npm run dev
```

Abra no navegador:

```
http://localhost:3333/documentation
```

E olhe.

As duas rotas estão lá. Os campos estão lá. Os tipos estão lá. `environment` mostra os três
valores aceitos, um a um. **Você não escreveu uma linha de documentação.**

Tudo isso veio do `health.schema.ts` que você escreveu na aula passada para outro motivo
completamente diferente — impedir vazamento de campo.

### Clique em "Try it out"

Escolha `GET /health`, clique em **Try it out** e depois em **Execute**.

A página dispara a requisição de verdade e mostra a resposta.

> [!IMPORTANT]
> Pare um segundo nisso. Esta página não é um **texto sobre** a API. É a API, com uma capa.
> Quem está integrando com você consegue testar cada rota antes de escrever a primeira linha
> de código do lado dele.

### E o documento por trás da página

A página é bonita, mas o que interessa às ferramentas é o documento. Acesse:

```
http://localhost:3333/documentation/json
```

É o OpenAPI cru. É esse arquivo que outras ferramentas leem para gerar código de cliente,
coleção de testes e validação de contrato.

---

## 🏷️ Capítulo 7: Deixando a documentação boa de ler

Funcionar não é o mesmo que estar bom. Abra a página de novo e repare em duas coisas feias:

1. As rotas não têm título nenhum — só o caminho.
2. Ao lado do código `200` está escrito **"Default Response"**, que não informa nada.

Vamos consertar as duas, e cada conserto ensina uma coisa diferente.

### Passo 1: Descrições nos campos e nas respostas

Abra `src/modules/health/health.schema.ts` e deixe-o assim:

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
 *
 * Há ainda um terceiro uso, de graça: estes schemas são a fonte da documentação
 * da API. Por isso cada campo tem um `.describe()` — o comentário acima dele
 * serve a quem lê o código; a descrição serve a quem consome a API, e essa
 * pessoa nunca vai abrir este arquivo.
 */

import { z } from 'zod'

/**
 * Resposta de `GET /health` — a rota de vida.
 *
 * Deliberadamente mínima. Quem consulta esta rota é o monitoramento externo, a
 * cada poucos segundos, e a única pergunta que ele faz é "a API responde?".
 * Uptime e ambiente são informação de dentro de casa: ficam na rota de prontidão.
 */
export const healthResponseSchema = z
  .object({
    status: z.literal('ok').describe('Sempre "ok". Se a API não estiver de pé, não há resposta.'),
  })
  // O `.describe()` no objeto inteiro vira o texto que a documentação mostra ao
  // lado do código 200. Sem ele, a página exibe "Default Response", que não
  // informa nada a quem está lendo para decidir se usa a rota.
  .describe('A API está de pé.')

/**
 * Resposta de `GET /health/ready` — a rota de prontidão.
 *
 * Detalhada, para quem opera a API. Quando houver banco de dados, é aqui que
 * entra a checagem de conexão: "estou de pé" e "estou pronto para atender" são
 * perguntas diferentes, e a segunda é a que decide se o tráfego pode chegar.
 */
export const readinessResponseSchema = z
  .object({
    status: z.literal('ok').describe('Sempre "ok" quando a API está pronta para atender.'),

    /** Há quantos segundos este processo está no ar. */
    uptime: z.number().describe('Há quantos segundos este processo está no ar.'),

    /** Momento em que a resposta foi gerada, no padrão ISO 8601. */
    timestamp: z.string().describe('Momento em que a resposta foi gerada, no padrão ISO 8601.'),

    /** Ambiente em que a API está rodando: development, test ou production. */
    environment: z
      .enum(['development', 'test', 'production'])
      .describe('Ambiente em que a API está rodando.'),
  })
  .describe('A API está pronta para atender, com os detalhes de operação.')

/** Formato da resposta da rota de vida, derivado do schema acima. */
export type HealthResponse = z.infer<typeof healthResponseSchema>

/** Formato da resposta da rota de prontidão, derivado do schema acima. */
export type ReadinessResponse = z.infer<typeof readinessResponseSchema>
```

> [!IMPORTANT]
> **Comentário e descrição não são a mesma coisa, e por isso os dois existem.**
>
> O comentário `/** Há quantos segundos... */` serve a quem abre este arquivo — alguém do
> time, mexendo no código. O `.describe()` serve a quem consome a API pela internet, e essa
> pessoa **nunca** vai abrir este arquivo.
>
> Escrever os dois parece repetição. Não é: são dois públicos diferentes, e um dia os textos
> vão precisar ser diferentes também.

### Passo 2: Título, explicação e etiqueta em cada rota

Abra `src/modules/health/health.routes.ts` e deixe-o assim:

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
 *
 * O mesmo `schema` alimenta a documentação da API. Os campos `summary`,
 * `description` e `tags` existem só para ela: o Fastify os ignora ao responder.
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
    {
      schema: {
        // `summary`, `description` e `tags` não mudam nada em execução: o
        // Fastify os ignora. Quem os lê é o gerador da documentação.
        summary: 'A API está de pé?',
        description:
          'Responde o mínimo possível, de propósito. É a rota que o monitoramento ' +
          'consulta a cada poucos segundos, e a única coisa que ele precisa saber é ' +
          'se houve resposta.',
        tags: ['saude'],
        response: { 200: healthResponseSchema },
      },
    },
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
    {
      schema: {
        summary: 'A API está pronta para atender?',
        description:
          'Uso interno. Devolve informação de operação — há quanto tempo o processo ' +
          'está no ar e em qual ambiente roda.\n\n' +
          'Hoje esta rota é aberta, porque o projeto ainda não tem autenticação. ' +
          'Quando houver login, ela passa a exigir usuário autenticado. Deixamos ' +
          'isso escrito aqui de propósito: dívida anotada é dívida que alguém paga.',
        tags: ['interna'],
        response: { 200: readinessResponseSchema },
      },
    },
    async (request, reply) => {
      return healthController.handleReadiness(request, reply)
    },
  )
}
```

Recarregue a página. Agora as rotas têm título, explicação, e estão separadas em dois grupos:
**saude** e **interna**.

### A dívida escrita na cara de quem vai usar

Leia com atenção a descrição do `/health/ready`:

> _"Hoje esta rota é aberta, porque o projeto ainda não tem autenticação. Quando houver login,
> ela passa a exigir usuário autenticado."_

Isso está na documentação **pública** da API. De propósito.

Era possível fazer o contrário. O `@fastify/swagger` sabe esconder uma rota:

```typescript
// NÃO faça isto aqui. É só para você saber que existe.
{ schema: { hide: true, response: { 200: readinessResponseSchema } } }
```

Mas repare no que `hide: true` faz e no que **não** faz: ele tira a rota da documentação e
**não tira a rota do ar**. O `/health/ready` continuaria respondendo normalmente para quem
digitasse o endereço.

> [!CAUTION]
> Esconder uma porta que continua destrancada não é segurança. É esquecer onde você deixou o
> problema.
>
> A rota está aberta hoje porque o projeto ainda não tem login. Isso é uma dívida, e dívida
> **anotada em lugar visível** é dívida que alguém paga. Dívida escondida vira surpresa.

**Então quando `hide: true` é legítimo?** Quando a rota realmente não faz parte do contrato
público: um endpoint de manutenção que só o próprio servidor chama, ou uma rota antiga em
processo de remoção, que você não quer que ninguém novo comece a usar.

A pergunta que separa os dois casos: _"estou escondendo porque não é da conta de quem lê, ou
estou escondendo porque tenho um problema que não quero encarar?"_

---

## 🚪 Capítulo 8: Documentação também é uma porta

Agora a pergunta séria da aula.

Você acabou de publicar, num único endereço, **o mapa completo da sua API**: todas as rotas,
todos os campos, todos os tipos, todas as regras de validação, em formato que um programa lê
sozinho.

Isso é ótimo para quem vai integrar com você.

E é ótimo para quem está estudando como atacar o seu serviço. **É o mesmo arquivo.**

Lembre da Aula 06: você tratou os erros centralmente justamente para a API parar de entregar
detalhe interno para o cliente. Seria estranho fechar aquela porta e abrir esta sem pensar.

### A conta, no estado de hoje

| Lado da balança                         | Quanto pesa **hoje**                                          |
| :-------------------------------------- | :------------------------------------------------------------ |
| Ganho: alguém de fora consegue integrar | **Zero.** As únicas rotas são de saúde. Não há o que integrar |
| Risco: o mapa fica público              | Pequeno, mas maior que zero                                   |

Quando um lado vale zero, não há dilema. **A documentação não sobe em produção** — por
enquanto.

E "por enquanto" é a palavra importante: no dia em que existir a primeira rota de negócio, o
ganho deixa de ser zero, e a decisão certa passa a ser outra — publicar a documentação
**protegida por login**, em vez de desligada.

> [!NOTE]
> Repare no formato desta decisão. Ela não é "documentação é perigosa, então nunca". É "hoje
> o ganho é zero, então não; quando o ganho existir, revemos". **Decisão com prazo e com
> gatilho é diferente de proibição.**

### Como isso vira código

Você já escreveu a linha, no Capítulo 5:

```typescript
if (options.docs ?? env.NODE_ENV !== 'production') {
  registerDocs(app)
}
```

Duas coisas acontecem aí, e vale separar:

O `env.NODE_ENV !== 'production'` é a **regra**: fora de produção, documentação ligada.

O `options.docs ??` é a **exceção controlada**: se alguém passar `docs` explicitamente, esse
valor vence. É o mesmo desenho do `logger`, que você criou na Aula 05 — e existe pela mesma
razão: **sem ele, não haveria como testar as duas situações.**

> [!IMPORTANT]
> Guarde esta ideia, porque ela vai aparecer o curso inteiro: **uma decisão que você não
> consegue testar é uma intenção, não uma garantia.**
>
> Se a documentação simplesmente lesse `NODE_ENV` e pronto, o teste do Capítulo 9 seria
> impossível de escrever, e "não sobe em produção" seria uma frase de comentário — do tipo que
> continua lá, escrita e bonita, meses depois de alguém ter mudado o comportamento.

### Veja o desligamento acontecer

Compile e rode como produção:

```bash
npm run build
```

E agora suba em modo de produção. **A sintaxe muda conforme o terminal:**

```powershell
# Windows (PowerShell)
$env:NODE_ENV="production"; npm start
```

```bash
# Linux / Mac / Git Bash
NODE_ENV=production npm start
```

Com o servidor no ar, acesse `http://localhost:3333/documentation`. Resultado real:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Endereço não encontrado: GET /documentation"
}
```

Três coisas para reparar nessa resposta:

1. **A página sumiu de verdade** — e o `/documentation/json` e o `/documentation/yaml` junto.
   Não é uma tela de "acesso negado" com o conteúdo atrás.
2. **O erro saiu no formato de sempre**, os mesmos três campos da Aula 06. Você não escreveu
   uma linha de tratamento para isso. O endereço simplesmente deixou de existir, e endereço
   inexistente já tinha dono.
3. **`/health` e `/health/ready` continuam respondendo.** Só a documentação saiu.

> [!WARNING]
> **No PowerShell, a variável continua valendo naquela janela.** Se você seguir a aula e o
> `npm run dev` começar a se comportar como produção, é isso. Feche o terminal e abra outro,
> ou rode `$env:NODE_ENV="development"`.

---

## 🧪 Capítulo 9: Testando uma coisa que não existe

Crie `src/shared/docs/docs.spec.ts`:

```typescript
/**
 * Testes da documentação da API
 *
 * Dois grupos, e eles provam coisas opostas.
 *
 * O primeiro prova que a documentação **existe e está certa** — e o teste que
 * mais importa ali é o que confere se as rotas chegaram à especificação. O
 * gerador não lê o código-fonte: ele escuta cada rota sendo registrada. Se um
 * dia a ordem de registro mudar no `app.ts`, ele deixa de ouvir e a página passa
 * a abrir **vazia**, sem erro, sem aviso e sem ninguém perceber. Este teste é o
 * que transforma esse silêncio em falha visível.
 *
 * O segundo grupo prova que a documentação **não existe** quando não deve. Sem
 * ele, "a documentação não sobe em produção" seria só uma intenção escrita num
 * comentário, e comentário nenhum impede um deploy.
 */

import { describe, expect, it } from 'vitest'
import { buildApp } from '../../app.ts'

/** Formato mínimo do que a especificação devolve, para não usar `any` nos testes. */
interface EspecificacaoOpenApi {
  openapi: string
  paths: Record<string, Record<string, { summary?: string; tags?: string[] }>>
}

describe('Documentação ligada', () => {
  it('publica a especificação em /documentation/json', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })

    expect(resposta.statusCode).toBe(200)

    await app.close()
  })

  it('inclui as rotas que existem de verdade', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })
    const especificacao = resposta.json<EspecificacaoOpenApi>()

    // Comparamos a lista inteira, e não apenas a presença de cada uma. Assim o
    // teste também falha se uma rota nova entrar sem documentação, ou se alguma
    // desaparecer da especificação sem desaparecer da API.
    expect(Object.keys(especificacao.paths)).toEqual(['/health', '/health/ready'])

    await app.close()
  })

  it('descreve os quatro campos de /health/ready', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })

    // Este teste prova que o schema escrito em Zod atravessou a tradução para o
    // formato OpenAPI. Se a ponte quebrar, a rota continua aparecendo — só que
    // sem nenhum detalhe, o que é pior do que não aparecer.
    for (const campo of ['status', 'uptime', 'timestamp', 'environment']) {
      expect(resposta.body).toContain(campo)
    }

    await app.close()
  })

  it('separa a rota interna da rota pública por etiqueta', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })
    const { paths } = resposta.json<EspecificacaoOpenApi>()

    expect(paths['/health']?.get?.tags).toEqual(['saude'])
    expect(paths['/health/ready']?.get?.tags).toEqual(['interna'])

    await app.close()
  })
})

describe('Documentação desligada', () => {
  it('não publica nenhum dos três endereços', async () => {
    const app = buildApp({ logger: false, docs: false })

    // São três endereços, e todos vêm do mesmo plugin. Conferimos os três porque
    // desligar dois e esquecer o terceiro deixaria a especificação disponível
    // para quem soubesse o caminho — que é exatamente o que queríamos evitar.
    for (const url of ['/documentation', '/documentation/json', '/documentation/yaml']) {
      const resposta = await app.inject({ method: 'GET', url })

      expect(resposta.statusCode).toBe(404)
    }

    await app.close()
  })

  it('devolve esse 404 no formato de erro único da API', async () => {
    const app = buildApp({ logger: false, docs: false })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })

    // A rota deixou de existir, e o endereço passou a ser tratado como qualquer
    // outro endereço inexistente: cai no `notFoundHandler` global, sem precisar
    // de nenhum tratamento próprio.
    expect(resposta.json()).toEqual({
      statusCode: 404,
      error: 'Not Found',
      message: 'Endereço não encontrado: GET /documentation/json',
    })

    await app.close()
  })
})
```

Rode:

```bash
npm test
```

```
 Test Files  4 passed (4)
      Tests  40 passed (40)
```

Eram 34 na aula passada; agora são **40**.

### Por que estes testes são diferentes dos outros

Quase todo teste que você escreveu até aqui responde _"isto funciona?"_. Os deste arquivo
respondem duas perguntas mais difíceis.

**"Isto continua sendo montado do jeito certo?"** — O teste que compara a lista de rotas não
testa o Fastify nem o Swagger; testa **a ordem do seu `app.ts`**. É a única coisa no projeto
que percebe se alguém, daqui a seis meses, mover o `registerDocs(app)` duas linhas para baixo.

**"Isto continua não existindo?"** — Testar ausência parece estranho até você perceber que a
decisão do Capítulo 8 é uma decisão de segurança. E decisão de segurança que ninguém verifica
sobrevive exatamente até a primeira refatoração distraída.

> [!IMPORTANT]
> Toda vez que você tomar uma decisão do tipo _"isto **não** deve acontecer"_, pergunte: **o
> que quebraria se ela deixasse de valer?** Se a resposta for "nada, ninguém ia perceber",
> falta um teste.

---

## 📄 Capítulo 10: Atualizando o README e as requisições

Abra `requisicoes/health.http` e acrescente a última linha:

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
```

E o `README.md`, que ganha a seção `## Documentação da API`. Este é o arquivo completo ao
final da Aula 08:

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

| Endereço                | O que é                                    |
| :---------------------- | :----------------------------------------- |
| `/documentation`        | A página navegável                         |
| `/documentation/json`   | A especificação OpenAPI, em JSON           |
| `/documentation/yaml`   | A mesma especificação, em YAML             |

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
````

### O `package.json`, por inteiro

Dois pacotes novos entraram hoje. Este é o arquivo completo:

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
    "build": "tsc --project tsconfig.build.json",
    "start": "node --env-file-if-exists=.env dist/server.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "check": "npm run lint && npm run format:check && npm run test && npm run build"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
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
    "prettier": "^3.9.6",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0",
    "vitest": "^4.1.10"
  }
}
```

> [!NOTE]
> As versões podem estar diferentes das suas. Isso não é problema — o `npm install` sempre
> baixa a versão mais recente compatível.

---

## 💾 Fechando o ciclo: mande para o GitHub

Hoje você criou `src/shared/docs/` com dois arquivos, alterou o `app.ts`, o schema e as rotas,
e documentou tudo no README. Feche o ciclo da Aula 02:

```bash
git add .
git commit -m "feat: adiciona documentação OpenAPI com Swagger"
git push
```

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Precisa terminar sem nenhum erro, com os **40 testes** passando.

### 2. A página abre e não está vazia

Com `npm run dev` rodando, abra `http://localhost:3333/documentation`.

| O que conferir                  | Esperado                                    |
| :------------------------------ | :------------------------------------------ |
| Quantas rotas aparecem          | **Duas**                                    |
| Os grupos                       | `saude` e `interna`, separados              |
| Ao lado do `200` de `/health`   | "A API está de pé.", não "Default Response" |
| Os valores de `environment`     | Os três, listados um a um                   |
| **Try it out** em `GET /health` | Responde `{"status":"ok"}` de verdade       |

> [!WARNING]
> **Se a página abrir com "No operations defined in spec!"**, o `registerDocs(app)` está
> **depois** do `app.register(healthRoutes)` no seu `app.ts`. Troque a ordem.

### 3. A documentação some em produção

```bash
npm run build
```

```powershell
# Windows (PowerShell)
$env:NODE_ENV="production"; npm start
```

```bash
# Linux / Mac / Git Bash
NODE_ENV=production npm start
```

| Endereço              | Esperado                                          |
| :-------------------- | :------------------------------------------------ |
| `/health`             | `{"status":"ok"}`                                 |
| `/health/ready`       | Os quatro campos, com `environment: "production"` |
| `/documentation`      | **404**, no formato de erro da API                |
| `/documentation/json` | **404**                                           |
| `/documentation/yaml` | **404**                                           |

Depois, feche esse terminal (ou volte a variável para `development`) antes de seguir.

---

## 🚨 Erros Comuns

**A página abre, mas diz "No operations defined in spec!"**
O `registerDocs(app)` está depois do registro das rotas no `app.ts`. O gerador só enxerga
rota que é registrada **depois** dele.

**`Cannot find module '@fastify/swagger'`**
Faltou instalar. Rode `npm install @fastify/swagger @fastify/swagger-ui` na raiz do projeto.

**A página abre, mas os campos aparecem vazios ou sem tipo**
Faltou o `transform: jsonSchemaTransform` no registro do `@fastify/swagger`. Sem ele o plugin
não entende schema escrito em Zod.

**`/documentation` responde 404 em desenvolvimento**
Seu `NODE_ENV` está como `production`. No PowerShell a variável continua valendo na janela
inteira depois que você a define — feche o terminal ou rode `$env:NODE_ENV="development"`.

**Ao lado do 200 continua escrito "Default Response"**
O `.describe()` foi colocado só nos campos, e não no objeto inteiro. Ele precisa estar depois
do `z.object({ ... })`, encadeado.

**O `npm run dev` passou a se comportar como produção**
Mesma causa do erro acima: a variável de ambiente ficou definida na janela do terminal.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/08-gabarito.md`](./exercicios/08-gabarito.md).

**1. Prove que a ordem importa**
No `app.ts`, mova o bloco `if (options.docs ...)` para **depois** do
`app.register(healthRoutes)`. Rode `npm test` e depois abra a página. Qual dos dois avisou
primeiro que algo estava errado — o teste ou o navegador? Depois desfaça.

**2. Uma rota nova sem documentação**
Acrescente uma rota qualquer (`/versao`, devolvendo `{ versao: '1.0.0' }`) com `schema` de
resposta, e rode `npm test`. Qual teste falha, e por quê? Ele estava certo em falhar?

**3. O `hide: true` na prática**
Coloque `hide: true` no schema do `/health/ready` e recarregue a página. A rota sumiu da
documentação — agora chame `http://localhost:3333/health/ready` no navegador. O que isso
mostra sobre "esconder" como estratégia de segurança? Depois desfaça.

**4. Documentação sem página**
Comente **apenas** o `app.register(fastifySwaggerUi, ...)` no `src/shared/docs/index.ts` e
tente acessar os três endereços. Quais respondem, e o que isso confirma sobre a divisão de
trabalho entre os dois plugins? Depois desfaça.

---

## 🎯 Resumo e Próximos Passos

Hoje a sua API ganhou uma documentação que **não pode mentir**, porque não é um segundo texto:
é o mesmo schema, olhado de outro ângulo.

O que ficou pronto:

- Página navegável e executável em `/documentation`, gerada a partir do código.
- A especificação OpenAPI em `/documentation/json` e `/documentation/yaml`, pronta para
  qualquer ferramenta consumir.
- Rotas com título, explicação e grupo, e campos com descrição própria.
- Documentação desligada em produção, com o motivo escrito e o gatilho para rever.
- 40 testes, incluindo dois que provam que uma coisa **não existe**.

E três ideias que valem além desta aula:

1. **A melhor documentação é a que não é escrita** — é derivada de algo que já precisava
   existir. Documentação que depende de alguém lembrar de atualizar sempre perde a corrida.
2. **Esconder não é proteger.** Se a porta está destrancada, tirá-la da planta só faz você
   esquecer que ela existe.
3. **Decisão que ninguém verifica é intenção.** Se você decidiu que algo não deve acontecer,
   escreva o teste que quebra quando voltar a acontecer.

**E agora?**

Você acabou de passar uma aula inteira pensando em quem está do outro lado da API — e
descobriu que nem todo mundo do outro lado tem boa intenção.

Essa é exatamente a pergunta da próxima aula, agora sem a parte fácil. Hoje a sua API aceita
qualquer quantidade de requisições, de qualquer origem, sem nenhum limite. Um script de dez
linhas consegue fazer milhares de chamadas por segundo, e nada no seu código diz "chega".

Na Aula 09 você vai colocar as três proteções que toda API pública precisa ter antes de
existir na internet: cabeçalhos de segurança, controle de quem pode chamar de outro site, e
limite de requisições por origem.
