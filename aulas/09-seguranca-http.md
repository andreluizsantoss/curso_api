# 🛡️ Aula 09: Segurança HTTP

> **Aula anterior:** [08 — Documentação Viva com Swagger](./08-documentacao-da-api.md)

Na aula passada você publicou o mapa completo da sua API e parou para pensar em quem está do
outro lado.

Hoje a pergunta continua, com a parte difícil: **e quando quem está do outro lado não tem boa
intenção?**

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Derrubar a própria API com um laço de repetição de cinco linhas — e depois impedir isso.
- Explicar a diferença entre uma proteção que **a API impõe** e uma que ela apenas **pede ao
  navegador**. É a distinção que separa quem entende segurança de quem instala bibliotecas.
- Decidir quais sites podem chamar a sua API pelo navegador, e ver o bloqueio acontecer.
- Escrever um limite conhecido do seu próprio código, em vez de torcer para ninguém notar.

---

## 📋 Pré-requisitos

- Aula 08 concluída, com o `npm run check` passando.
- O `npm run dev` funcionando.

---

## 💣 Capítulo 1: Derrube a sua própria API

Antes de instalar qualquer coisa, veja o tamanho do problema. Suba a API:

```bash
npm run dev
```

Em **outro** terminal, na pasta do projeto, crie um arquivo `ataque.js`:

```javascript
// Arquivo descartável: vamos apagá-lo no fim da aula.
// Ele não faz nada de sofisticado — é o que qualquer pessoa escreveria em 2 minutos.

let respondidas = 0

const inicio = Date.now()

const tentativas = Array.from({ length: 2000 }, async () => {
  const resposta = await fetch('http://localhost:3333/health/ready')
  if (resposta.ok) respondidas++
})

await Promise.all(tentativas)

const segundos = ((Date.now() - inicio) / 1000).toFixed(1)

console.log(`${respondidas} requisições respondidas em ${segundos}s`)
```

Rode:

```bash
node ataque.js
```

**Duas mil requisições. Todas respondidas.** Sem reclamação, sem registro de alarme, sem
limite nenhum.

E repare no que isso significa em duas situações bem diferentes:

| Quem faz                                | O que acontece                                          |
| :-------------------------------------- | :------------------------------------------------------ |
| Alguém querendo derrubar o serviço      | Consegue, e não precisa de nenhuma técnica especial     |
| Um cliente com laço de repetição errado | Faz o mesmo estrago, **sem querer**, e nem fica sabendo |

> [!IMPORTANT]
> O segundo caso é o mais comum na vida real, e é o que mais derruba API em produção. Não é
> preciso um atacante: basta um sistema parceiro com um `while` mal escrito.

---

## 📖 Capítulo 2: São três problemas, não um

"Segurança HTTP" costuma ser tratada como uma caixa só, com três bibliotecas dentro. São três
problemas diferentes, e a diferença muda o que você pode esperar de cada solução.

| Plugin                | Protege contra                                      | **Quem obedece** |
| :-------------------- | :-------------------------------------------------- | :--------------- |
| `@fastify/helmet`     | O navegador operar no modo mais permissivo          | **O navegador**  |
| `@fastify/cors`       | Outro site chamar a API em nome do cidadão          | **O navegador**  |
| `@fastify/rate-limit` | Volume: abuso, força bruta, cliente com laço errado | **A API**        |

Leia a última coluna de novo. É a lição central desta aula:

> [!CAUTION]
> **Helmet e CORS são pedidos que a sua API faz ao navegador.**
>
> O `ataque.js` que você acabou de rodar não é um navegador. Ele ignora os dois completamente,
> hoje e depois de você instalá-los. Nada do que você fizer com Helmet ou CORS vai impedi-lo.
>
> Só o rate limit é uma regra que a **sua API** impõe por conta própria.

Isso não torna os outros dois inúteis — longe disso. Eles protegem **o cidadão que usa um
navegador**, que é a vítima real de clickjacking e de um site de terceiro chamando a API em
nome dele. São proteções para uma pessoa, não para o servidor.

Mas confundir "o navegador respeita" com "está protegido" é o erro conceitual mais comum da
área, e vale desarmá-lo agora, antes de escrever a primeira linha.

### A analogia do prédio

| Proteção   | No prédio                                                                        |
| :--------- | :------------------------------------------------------------------------------- |
| Helmet     | As placas: "não corra", "elevador não é para carga". Quem lê e respeita, obedece |
| CORS       | A lista da portaria: quem pode subir e para qual apartamento                     |
| Rate limit | A catraca: passou de tantas vezes, ela trava. Não adianta discutir               |

Placa e lista de portaria dependem de alguém colaborar. A catraca não.

---

## 📦 Capítulo 3: Instalando os três

```bash
npm install @fastify/helmet @fastify/cors @fastify/rate-limit
```

```
added 5 packages, and audited 271 packages in 3s

93 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

---

## 🚦 Capítulo 4: A catraca — limite de requisições

Começamos pelo rate limit, por um motivo prático: é o único dos três que você consegue
**provar sozinho**, com o mesmo `ataque.js` do Capítulo 1.

Crie a pasta `src/shared/security/` e, dentro dela, o arquivo `index.ts`:

```typescript
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
 * **Limite conhecido, e anotado de propósito:** a contagem usa `request.ip`.
 * Atrás de um proxy, esse valor é o IP do proxy, e não o de quem chamou — todos
 * os clientes viram um só, e o primeiro a estourar bloqueia os demais. A correção
 * é ligar o `trustProxy` do Fastify, e ela entra quando existir um proxy de
 * verdade na frente para provar o comportamento.
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

> [!NOTE]
> O arquivo já traz os três plugins. Vamos entender o CORS e o Helmet nos capítulos
> seguintes — mas eles ficam escritos de uma vez porque as três proteções pertencem ao mesmo
> lugar, e quebrar o arquivo em três etapas deixaria você com um arquivo incompleto no disco
> entre um capítulo e outro.

### Por que os limites são constantes exportadas

Repare que `RATE_LIMIT_GLOBAL` e `RATE_LIMIT_HEALTH` são exportados, em vez de números soltos
dentro do registro.

Dois motivos, e o segundo é o que importa:

1. A rota `/health` vai precisar do valor dela, em outro arquivo.
2. **Os testes vão comparar os dois.** Você vai escrever um teste que verifica que o limite da
   rota de vida é _maior_ que o global — e não `false`. Com os números escondidos dentro do
   registro, esse teste seria impossível de escrever.

---

## 🔌 Capítulo 5: Ligando no `app.ts`

Abra `src/app.ts` e deixe-o exatamente assim:

```typescript
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
import { registerSecurity } from './shared/security/index.ts'
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

Uma única linha nova de código (`registerSecurity(app)`), e ela está no mesmo lugar de sempre:
**antes das rotas**. É o terceiro capítulo seguido em que essa ordem importa, e sempre pelo
mesmo motivo — o que vem antes vale para tudo que vier depois, sem ninguém precisar lembrar.

---

## 🌐 Capítulo 6: A lista da portaria — CORS

O `security/index.ts` que você escreveu lê `env.CORS_ORIGINS`, e essa variável ainda não
existe. Vamos criá-la.

### O que o CORS resolve, na prática

Imagine que o cidadão está logado no portal da prefeitura, numa aba. Em outra aba, ele abre um
site qualquer. Sem CORS, o JavaScript **daquele site** pode chamar a sua API, e o navegador
manda os cookies do cidadão junto — porque a requisição parte do navegador dele.

O CORS é a sua API dizendo ao navegador: _"só entregue a minha resposta ao script se ele vier
de um destes endereços."_

### Passo 1: a variável de ambiente

Abra `src/shared/env/env.schema.ts` e deixe-o assim:

```typescript
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

> [!IMPORTANT]
> Repare no `.transform()`. Ele é a primeira vez no curso em que o schema **não apenas valida,
> mas converte**: entra o texto `"a,b"`, sai a lista `['a', 'b']`.
>
> O ganho é que o resto do projeto nunca vê o texto. Ninguém precisa lembrar de fazer
> `.split(',')` — e "ninguém precisa lembrar" é a frase que você já viu em todas as decisões
> boas deste curso.

### Passo 2: o modelo, no `.env.example`

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
```

Acrescente a mesma linha ao seu `.env` (que não é versionado):

```bash
CORS_ORIGINS=
```

---

## 🩺 Capítulo 7: A exceção do monitoramento

Existe um problema no que acabamos de fazer, e ele é sutil.

Quem consulta `/health` é o sistema de monitoramento — a cada poucos segundos, para sempre,
sem parar. Com o limite global de 100 por minuto, uma consulta a cada 3 segundos dá 20 por
minuto e cabe. Mas basta um segundo monitor, ou uma frequência maior, e o alarme começa a
receber **429**.

> [!CAUTION]
> Um alarme que silencia porque bateu no rate limit é pior do que não ter alarme. Você fica
> sem saber que a API caiu **e** achando que está tudo bem.

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
import { RATE_LIMIT_HEALTH } from '../../shared/security/index.ts'
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

      // Esta rota tem um limite de requisições próprio, mais alto que o do resto
      // da API. Quem a consulta é o monitoramento, a cada poucos segundos, para
      // sempre — com o limite global ele seria bloqueado, e alarme silenciado por
      // rate limit é pior do que não ter alarme.
      config: { rateLimit: RATE_LIMIT_HEALTH },
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

### A tentação que recusamos

O plugin permite isentar uma rota por completo:

```typescript
config: { rateLimit: false } // NÃO fizemos isto — e o motivo importa
```

Seria mais simples de escrever e resolveria o problema do monitoramento. Mas "isento" e "sem
teto" são a mesma coisa: um endereço público sem limite nenhum é convite para ser usado como
alvo de amplificação.

> Um limite **maior** resolve o problema do monitoramento sem criar um problema novo. Sempre
> que a saída fácil for "remover a proteção aqui", pergunte se dá para **afrouxar** em vez de
> **desligar**.

---

## 🎬 Capítulo 8: Veja funcionar

Suba a API de novo (`npm run dev`) e rode o `ataque.js` mais uma vez:

```bash
node ataque.js
```

Agora as requisições começam a ser recusadas. Faça uma chamada normal para ver a resposta:

```
{
  "statusCode": 429,
  "error": "Too Many Requests",
  "message": "Limite de 100 requisições por minuto excedido. Tente novamente em 60 segundos."
}
```

Três coisas para reparar:

1. **429** é o código HTTP de "requisições demais" — a faixa `4xx`, que a Aula 06 ensinou a
   ler como "o problema está do seu lado".
2. **Os três campos são os mesmos** de todos os outros erros da API. O plugin já responde
   nesse formato; nós só reescrevemos a frase, que vinha em inglês.
3. **A frase diz quanto falta de verdade**, em segundos, e não "1 minuto" fixo.

E o `/health`, que tem o limite próprio, continua respondendo normalmente enquanto o
`/health/ready` já está bloqueado. Teste os dois lado a lado.

### Os cabeçalhos do Helmet

Abra qualquer rota no navegador, aperte **F12**, vá na aba **Network**, clique na requisição e
olhe os **Response Headers**. Alguns dos que apareceram:

| Cabeçalho                   | O que pede ao navegador                                  |
| :-------------------------- | :------------------------------------------------------- |
| `x-content-type-options`    | Não adivinhe o tipo do arquivo pelo conteúdo             |
| `x-frame-options`           | Não deixe outro site embutir esta página numa moldura    |
| `strict-transport-security` | Só fale comigo por HTTPS daqui em diante                 |
| `content-security-policy`   | De onde uma página pode carregar script, estilo e imagem |
| `referrer-policy`           | Não conte para o próximo site de onde o cidadão veio     |

Nenhum deles exigiu configuração. **Este é o argumento do Helmet:** são proteções que já
existem no navegador há anos e ficam desligadas por omissão, para não quebrar sites antigos.
Ele só as liga.

> [!TIP]
> Confira que a documentação da Aula 08 continua abrindo: `http://localhost:3333/documentation`.
>
> Ela é o único lugar do projeto que **é uma página**, e a política de conteúdo do Helmet
> poderia deixá-la em branco. Não deixa — a página carrega tudo do próprio domínio, que é
> exatamente o que a política padrão permite. Mas isso foi **conferido**, não presumido.

---

## 🧪 Capítulo 9: Os testes

Crie `src/shared/security/security.spec.ts`:

```typescript
/**
 * Testes da segurança HTTP
 *
 * Os três plugins protegem coisas diferentes, e os testes seguem essa divisão.
 *
 * Repare no que o `remoteAddress` faz aqui: ele diz ao `inject` de qual IP a
 * requisição está vindo. Sem isso, todos os testes deste arquivo dividiriam a
 * mesma cota de requisições e passariam a falhar uns por causa dos outros — na
 * ordem em que rodassem, que nem sempre é a mesma. Cada teste usa um IP próprio.
 */

import { describe, expect, it } from 'vitest'
import { buildApp } from '../../app.ts'
import { RATE_LIMIT_GLOBAL, RATE_LIMIT_HEALTH } from './index.ts'

describe('Cabeçalhos de segurança (Helmet)', () => {
  it('manda o navegador não adivinhar o tipo do conteúdo', async () => {
    const app = buildApp({ logger: false, docs: false })

    const resposta = await app.inject({
      method: 'GET',
      url: '/health',
      remoteAddress: '10.0.0.1',
    })

    // Sem este cabeçalho, o navegador tenta "adivinhar" o tipo de um arquivo
    // pelo conteúdo, e um texto qualquer pode acabar executado como script.
    expect(resposta.headers['x-content-type-options']).toBe('nosniff')

    await app.close()
  })

  it('impede que a API seja embutida em um site de outra pessoa', async () => {
    const app = buildApp({ logger: false, docs: false })

    const resposta = await app.inject({
      method: 'GET',
      url: '/health',
      remoteAddress: '10.0.0.2',
    })

    // É a proteção contra clickjacking: um site coloca a nossa página numa
    // moldura invisível e engana o cidadão para clicar onde ele não vê.
    expect(resposta.headers['x-frame-options']).toBe('SAMEORIGIN')

    await app.close()
  })

  it('protege também as respostas de erro', async () => {
    const app = buildApp({ logger: false, docs: false })

    const resposta = await app.inject({
      method: 'GET',
      url: '/endereco-que-nao-existe',
      remoteAddress: '10.0.0.3',
    })

    // Este é o teste que costuma faltar. Resposta de erro é uma resposta como
    // qualquer outra, e o esquecimento clássico é proteger só o caminho feliz.
    expect(resposta.statusCode).toBe(404)
    expect(resposta.headers['x-content-type-options']).toBe('nosniff')

    await app.close()
  })
})

describe('Limite de requisições (Rate Limit)', () => {
  it('recusa quem passa do limite, com o código 429', async () => {
    const app = buildApp({ logger: false, docs: false })
    let ultimaResposta

    // Uma a mais que o permitido: as primeiras passam, a seguinte é recusada.
    for (let tentativa = 0; tentativa <= RATE_LIMIT_GLOBAL.max; tentativa++) {
      ultimaResposta = await app.inject({
        method: 'GET',
        url: '/health/ready',
        remoteAddress: '10.0.0.4',
      })
    }

    expect(ultimaResposta?.statusCode).toBe(429)

    await app.close()
  })

  it('recusa em português, no formato de erro único da API', async () => {
    const app = buildApp({ logger: false, docs: false })
    let ultimaResposta

    for (let tentativa = 0; tentativa <= RATE_LIMIT_GLOBAL.max; tentativa++) {
      ultimaResposta = await app.inject({
        method: 'GET',
        url: '/health/ready',
        remoteAddress: '10.0.0.5',
      })
    }

    const corpo = ultimaResposta?.json()

    // O plugin já responde nos mesmos três campos, mas com a frase em inglês.
    // Uma resposta fora do padrão bastaria para quebrar a promessa da API de
    // falar uma língua só com quem a chama.
    expect(corpo).toMatchObject({ statusCode: 429, error: 'Too Many Requests' })
    expect(corpo.message).toContain('Limite de 100 requisições por minuto excedido')
    expect(corpo.message).toMatch(/Tente novamente em \d+ segundos?\./)

    await app.close()
  })

  it('deixa o monitoramento consultar /health além do limite global', async () => {
    const app = buildApp({ logger: false, docs: false })
    let bloqueadas = 0

    // Vinte requisições a mais do que o limite global permitiria. Se a exceção
    // da rota de vida não existisse, as últimas seriam recusadas — e o alarme
    // do monitoramento silenciaria justamente sob carga.
    for (let tentativa = 0; tentativa < RATE_LIMIT_GLOBAL.max + 20; tentativa++) {
      const resposta = await app.inject({
        method: 'GET',
        url: '/health',
        remoteAddress: '10.0.0.6',
      })

      if (resposta.statusCode !== 200) bloqueadas++
    }

    expect(bloqueadas).toBe(0)

    await app.close()
  })

  it('mas o /health também tem teto — ele é maior, não infinito', async () => {
    const app = buildApp({ logger: false, docs: false })
    let ultimaResposta

    // Uma a mais que o limite PRÓPRIO da rota de vida. Rota isenta é rota sem
    // limite, e endereço público sem limite é alvo de amplificação.
    //
    // Este teste precisa exercitar o comportamento, e não comparar as duas
    // constantes: trocar `RATE_LIMIT_HEALTH` por `false` no arquivo de rotas
    // deixaria a rota sem teto sem alterar constante nenhuma, e uma comparação
    // entre números continuaria passando, feliz, sobre uma porta escancarada.
    for (let tentativa = 0; tentativa <= RATE_LIMIT_HEALTH.max; tentativa++) {
      ultimaResposta = await app.inject({
        method: 'GET',
        url: '/health',
        remoteAddress: '10.0.0.9',
      })
    }

    expect(RATE_LIMIT_HEALTH.max).toBeGreaterThan(RATE_LIMIT_GLOBAL.max)
    expect(ultimaResposta?.statusCode).toBe(429)

    await app.close()
  })
})

describe('Controle de origem (CORS)', () => {
  it('não autoriza origem que não está na lista', async () => {
    const app = buildApp({ logger: false, docs: false })

    const resposta = await app.inject({
      method: 'GET',
      url: '/health',
      remoteAddress: '10.0.0.7',
      headers: { origin: 'https://site-de-terceiro.example' },
    })

    // Repare que a API RESPONDE normalmente: o CORS não bloqueia a requisição,
    // ele deixa de dar a autorização que o navegador exige para entregar a
    // resposta ao script que a pediu. Quem bloqueia é o navegador — e é por isso
    // que CORS não protege contra um programa que fale HTTP direto.
    expect(resposta.statusCode).toBe(200)
    expect(resposta.headers['access-control-allow-origin']).toBeUndefined()

    await app.close()
  })
})

describe('Convivência com o resto da API', () => {
  it('não quebra a documentação da API', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({
      method: 'GET',
      url: '/documentation/json',
      remoteAddress: '10.0.0.8',
    })

    // A política de conteúdo do Helmet restringe de onde uma página pode
    // carregar script e estilo, e é o tipo de coisa que deixaria a documentação
    // em branco sem gerar erro nenhum no servidor.
    expect(resposta.statusCode).toBe(200)

    await app.close()
  })
})
```

E as regras novas do schema de ambiente, em `src/shared/env/env.schema.spec.ts`. Este é o
arquivo completo:

```typescript
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
```

Rode:

```bash
npm test
```

```
 Test Files  6 passed (6)
      Tests  52 passed (52)
```

Eram 40 na aula passada; agora são **52**.

### O detalhe do `remoteAddress`

Este é o tipo de coisa que só aparece quando se escreve o teste:

```typescript
remoteAddress: '10.0.0.4'
```

Sem ele, todos os testes deste arquivo viriam do mesmo endereço e **dividiriam a mesma cota de
100 requisições**. O teste que estoura o limite deixaria os seguintes sem crédito, e eles
falhariam — não por estarem errados, mas por causa da ordem em que rodaram.

> [!IMPORTANT]
> Teste que depende da ordem de execução é pior do que teste nenhum, porque falha de forma
> intermitente e treina o time a reexecutar até passar. Cada teste aqui usa um IP próprio, e
> passa a ser independente dos outros.

---

## 📄 Capítulo 10: O `package.json` e o README

Três pacotes novos. Este é o arquivo completo:

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
    "prettier": "^3.9.6",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0",
    "vitest": "^4.1.10"
  }
}
```

E no `README.md`, duas mudanças: a linha nova na tabela de configuração e a seção
`## Segurança`. Acrescente a linha do `CORS_ORIGINS` à tabela de variáveis:

```markdown
| Variável       | O que controla                                     | Padrão        |
| :------------- | :------------------------------------------------- | :------------ |
| `NODE_ENV`     | Ambiente: `development`, `test` ou `production`    | `development` |
| `PORT`         | Porta em que a API escuta                          | `3333`        |
| `HOST`         | Endereço de rede que a API aceita                  | `0.0.0.0`     |
| `CORS_ORIGINS` | Sites que podem chamar a API pelo navegador        | _(nenhum)_    |
```

E a seção nova, depois de `## Formato de erro`:

```markdown
## Segurança

| Proteção       | O que faz                                                    |
| :------------- | :----------------------------------------------------------- |
| **Helmet**     | Liga os cabeçalhos de segurança que o navegador respeita     |
| **CORS**       | Só as origens em `CORS_ORIGINS` podem chamar pelo navegador  |
| **Rate limit** | 100 requisições por minuto, por IP                           |

A rota `/health` tem limite próprio, de 240 por minuto, para não bloquear o monitoramento.

**Limite conhecido:** a contagem usa o IP da conexão. Atrás de um proxy, todos os clientes são
contados como um só. Será corrigido quando a API passar a rodar atrás de proxy.
```

E apague o arquivo de ataque, que era descartável:

```bash
# Windows (PowerShell)
Remove-Item ataque.js

# Linux / Mac
rm ataque.js
```

---

## 💾 Fechando o ciclo: mande para o GitHub

```bash
git add .
git commit -m "feat: adiciona helmet, cors e limite de requisições"
git push
```

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Sem nenhum erro, com os **52 testes** passando.

### 2. O limite existe de verdade

| O que fazer                              | Esperado                             |
| :--------------------------------------- | :----------------------------------- |
| Chamar `/health/ready` mais de 100 vezes | **429**, com a mensagem em português |
| Chamar `/health` 150 vezes               | Todas respondem **200**              |
| Esperar um minuto e tentar de novo       | Volta a responder 200                |

### 3. Os cabeçalhos chegam

Na aba **Network** do navegador (F12), a resposta de qualquer rota traz
`x-content-type-options`, `x-frame-options` e `strict-transport-security`.

### 4. A documentação continua de pé

`http://localhost:3333/documentation` abre normalmente, com as duas rotas.

### 5. O arquivo descartável foi apagado

```bash
git status
```

Não deve haver nenhum `ataque.js`.

---

## 🚨 Erros Comuns

**`Cannot find module '@fastify/helmet'`**
Faltou instalar. Rode `npm install @fastify/helmet @fastify/cors @fastify/rate-limit`.

**Os testes falham de forma diferente a cada execução**
Faltou o `remoteAddress` em algum teste. Sem ele, os testes dividem a mesma cota de
requisições e um derruba o outro, na ordem em que rodarem.

**`/health/ready` responde 429 no meio da aula, sem você ter atacado nada**
Você estourou o limite antes, e a janela de um minuto ainda não fechou. Espere um minuto — ou
reinicie o servidor, porque a contagem fica na memória do processo.

**O `npm run dev` não sobe: `CORS_ORIGINS ...`**
Falta a variável no seu `.env`. Acrescente `CORS_ORIGINS=` (vazio mesmo).

**A página `/documentation` abre em branco**
Confira se você não alterou a configuração do Helmet. A política padrão funciona com a
documentação; uma política mais restritiva escrita à mão pode não funcionar.

**Coloquei meu site em `CORS_ORIGINS` e continua bloqueado**
A origem precisa bater exatamente: protocolo, domínio e porta. `http://localhost:5173` e
`http://localhost:3000` são origens diferentes, e `http://` não é `https://`.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/09-gabarito.md`](./exercicios/09-gabarito.md).

**1. O CORS não protege o que você acha que protege**
Coloque `CORS_ORIGINS=http://localhost:5173` no `.env` e reinicie. Agora chame a API com
`curl` (ou com o `ataque.js`) passando o cabeçalho `Origin: https://site-qualquer.example`. A
requisição foi bloqueada? Por quê? O que isso diz sobre CORS como proteção do servidor?

**2. Descubra onde o limite é contado**
Estoure o limite, reinicie o servidor com `Ctrl+C` e `npm run dev`, e tente de novo
imediatamente. Você foi bloqueado? Onde a contagem estava guardada, e o que isso significa
quando a API rodar em mais de um processo ao mesmo tempo?

**3. Isente a rota e veja o que se perde**
Troque, no `health.routes.ts`, o `config: { rateLimit: RATE_LIMIT_HEALTH }` por
`config: { rateLimit: false }`. Rode `npm test`. Qual teste falha, e ele estava certo em
falhar? Depois desfaça.

**4. O que a resposta conta sobre você**
Liste **todos** os cabeçalhos de uma resposta da API, não só os do Helmet. Procure um que
revele qual servidor ou framework está por trás — muitos servidores mandam um. Você encontrou?

Depois olhe os que começam com `x-ratelimit`. O que eles contam a quem está chamando, e por
que isso pode ser bom e ruim ao mesmo tempo?

---

## 🎯 Resumo e Próximos Passos

Hoje a API deixou de aceitar qualquer coisa, de qualquer um, em qualquer volume.

O que ficou pronto:

- Limite de 100 requisições por minuto por IP, com resposta **429** em português e no formato
  único da API.
- `/health` com limite próprio, maior — para o monitoramento não silenciar.
- Cabeçalhos de segurança em **todas** as respostas, inclusive nas de erro.
- Lista fechada de origens autorizadas, configurável por ambiente, com o padrão em "ninguém".
- 52 testes.

E três ideias que valem além desta aula:

1. **Saiba quem obedece.** Uma proteção que depende do navegador colaborar protege o cidadão,
   não o servidor. As duas coisas importam — confundi-las é que é perigoso.
2. **Afrouxe em vez de desligar.** Quando a proteção atrapalhar um caso legítimo, o primeiro
   reflexo deve ser aumentar o limite, e não removê-lo.
3. **Escreva o buraco que você conhece.** O rate limit desta API tem um limite real atrás de
   proxy. Ele está escrito no código, no README e na próxima aula que vai resolvê-lo. Buraco
   anotado é dívida; buraco silencioso é armadilha.

**E agora?**

A sua API está pronta, testada, documentada e defendida. E ainda depende de alguém ter a
versão certa do Node instalada na mão para rodar em outro computador.

Na próxima aula você vai empacotar a API inteira — código, dependências e a versão exata do
Node — em uma coisa só, que roda igual em qualquer máquina. É o fim do "na minha máquina
funciona", e é também o que vai permitir, na aula seguinte, colocar um proxy na frente e
finalmente ver o problema do `request.ip` acontecer.
