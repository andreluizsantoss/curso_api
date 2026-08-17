# 🧪 Aula 05: Testes Automatizados e a Promessa da Aula 01

Bem-vindos à **Aula 05**! 🎉

Hoje uma promessa antiga vai ser cumprida.

Lá na Aula 01, quando criamos o `app.ts` separado do `server.ts`, ficou escrito no
comentário do arquivo:

> _"Separamos a montagem do app da inicialização do servidor para facilitar os testes
> automatizados: nos testes importamos apenas o app, sem precisar ocupar uma porta de rede."_

Vocês aceitaram aquilo por confiança, sem ter como conferir. **Hoje vocês vão ver funcionar** —
e vão entender por que aquela decisão, tomada quatro aulas atrás, valeu a pena.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar por que testar na mão não escala, com argumentos concretos.
- Escrever testes automatizados para uma classe e para uma rota HTTP.
- Simular uma requisição sem abrir porta de rede.
- Testar mensagens de erro — aquilo que quase todo projeto esquece de verificar.
- Entender por que testar o caminho que dá errado importa tanto quanto o que dá certo.

## 📋 Pré-requisitos

- Aulas 01 a 04 concluídas.
- `npm run check` passando no seu projeto.

---

## 😩 Capítulo 1: O problema que você já sentiu

Pense em como você verificou se a API funcionava nas últimas quatro aulas:

1. Abrir o terminal
2. Rodar `npm run dev`
3. Esperar subir
4. Abrir o navegador
5. Digitar `localhost:3333/health`
6. Olhar a resposta e julgar se está certa
7. Voltar ao terminal e parar o servidor

**Sete passos. Toda vez.**

E isso com **uma** rota. Some as rotas que ainda vão existir na API do Curso: cadastrar
cidadão, consultar protocolo, anexar documento, autenticar, listar atendimentos. Facilmente
trinta rotas.

Testar trinta rotas na mão, a cada alteração, levaria uma hora. Ninguém faz isso. Na prática
o que acontece é: você altera uma coisa, testa **só aquela**, e confia que o resto continua
funcionando.

### É exatamente aí que os bugs passam

O nome disso é **regressão**: algo que funcionava para de funcionar por causa de uma
alteração em outro lugar.

E aqui vai um caso real, deste projeto: entre a Aula 04 e esta, um dos comandos do projeto
quebrou por causa de uma configuração de import. O erro só apareceu quando alguém, por acaso,
rodou o comando na situação certa. **Um teste automatizado teria pego no ato.**

> [!IMPORTANT]
> **A ideia central de hoje:**
>
> Teste automatizado não é sobre provar que o código funciona **agora**. Você acabou de
> escrever, você sabe que funciona.
>
> É sobre descobrir, **daqui a três meses**, que uma alteração aparentemente inofensiva
> quebrou algo que ninguém lembrava que existia.
>
> Você não escreve testes para o você de hoje. Escreve para o você de daqui a três meses — e
> para o colega que vai mexer no seu código sem conhecer as armadilhas.

---

## 📖 Capítulo 2: O que é um teste automatizado?

Um teste é apenas um **programa que verifica outro programa**.

Ele faz três coisas, sempre nesta ordem:

1. **Prepara** uma situação
2. **Executa** o que quer testar
3. **Confere** se o resultado é o esperado

A analogia mais próxima é a **linha de inspeção de uma fábrica**. Ninguém confia que o
produto saiu certo só porque a máquina estava ligada. Existe um posto no fim da linha onde
cada peça é medida contra um padrão. Se não bate, a peça é separada **antes** de chegar ao
cliente.

Nossos testes são esse posto de inspeção. E a diferença para o teste manual é simples: a
inspeção acontece **em segundos**, quantas vezes quisermos, sem cansar e sem esquecer nenhum
item.

### Vocabulário

| Termo               | O que significa                                          |
| :------------------ | :------------------------------------------------------- |
| **Suíte**           | Um grupo de testes relacionados. Criada com `describe()` |
| **Caso de teste**   | Uma verificação específica. Criada com `it()`            |
| **Asserção**        | A conferência em si. Feita com `expect()`                |
| **Passou / falhou** | Se o resultado bateu com o esperado ou não               |

---

## 📦 Capítulo 3: Instalando o Vitest

O **Vitest** é a ferramenta que vai executar os nossos testes: ela encontra os arquivos de
teste, roda cada um e mostra um relatório.

```bash
npm install -D vitest
```

Repare no `-D`: testes rodam na nossa máquina, nunca no servidor de produção. Ferramenta de
desenvolvimento.

> [!NOTE]
> **Por que o Vitest e não outro?**
>
> Ele entende TypeScript e o nosso estilo de imports **sem nenhuma configuração**. Não vamos
> criar arquivo de configuração nenhum nesta aula — ele simplesmente funciona com o projeto
> do jeito que já está.
>
> Menos configuração é menos coisa para quebrar, e menos coisa para explicar.

### Onde os testes ficam

Neste projeto, o arquivo de teste mora **ao lado** do arquivo que ele testa:

```
src/modules/health/
├── health.service.ts        ← o código
└── health.spec.ts           ← o teste dele
```

O sufixo `.spec.ts` (de _specification_, especificação) é o que o Vitest procura.

**Por que lado a lado, e não numa pasta `tests/` separada:** quando você abre a pasta de uma
funcionalidade, vê tudo o que diz respeito a ela — inclusive o que garante que ela funciona.
Se o teste morasse longe, seria fácil alterar o código e esquecer que existia um teste
esperando por ele.

---

## ✍️ Capítulo 4: O primeiro teste

Vamos começar pela peça mais simples: o `HealthService`. Ele não depende de HTTP, não depende
de banco. É só uma classe com um método.

Crie o arquivo `src/modules/health/health.spec.ts`:

```typescript
import { describe, expect, it } from 'vitest'
import { HealthService } from './health.service.ts'

describe('HealthService', () => {
  it('responde com status "ok"', () => {
    const service = new HealthService()

    expect(service.getStatus().status).toBe('ok')
  })
})
```

Rode:

```bash
npx vitest run
```

Você deve ver:

```
 ✓ src/modules/health/health.spec.ts (1 test)

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

**Parabéns, primeiro teste automatizado da sua vida.** 🎉

### Lendo o que você escreveu

```typescript
describe('HealthService', () => { ... })
```

Agrupa testes relacionados. O texto é livre — serve para você se localizar no relatório
quando algo falhar.

```typescript
it('responde com status "ok"', () => { ... })
```

Um caso de teste. Leia em voz alta em inglês: _"it responds with status ok"_ — "ele responde
com status ok". A frase descreve o **comportamento esperado**, não o código.

> [!TIP]
> **Como escrever um bom nome de teste:** complete a frase _"ele deveria..."_.
>
> - ✅ `it('recusa uma porta que não é número')`
> - ❌ `it('testa a porta')`
>
> Quando um teste falha, o nome dele é a **primeira coisa** que você lê. Um nome bom já diz
> o que quebrou, antes mesmo de olhar o código.

```typescript
expect(service.getStatus().status).toBe('ok')
```

A asserção. Leia como uma frase: _"espero que isto seja igual a 'ok'"_. Se não for, o teste
falha e o Vitest mostra os dois valores lado a lado.

### Vendo um teste falhar de propósito

Um teste que você nunca viu falhar não é confiável — ele pode estar passando por engano.

Troque `'ok'` por `'okay'` e rode de novo:

```
AssertionError: expected 'ok' to be 'okay' // Object.is equality

Expected: "okay"
Received: "ok"
```

Repare como a mensagem é útil: ela diz o que você **esperava** e o que **recebeu**. Desfaça a
alteração antes de continuar.

### Testando o que muda a cada execução

O `getStatus()` devolve três campos, e só testamos um. Faltam o `uptime` e o `timestamp`.

Aqui aparece uma dificuldade que você vai encontrar a vida inteira: **esses dois valores mudam
a cada execução**. O `uptime` cresce a cada segundo. O `timestamp` nunca se repete.

Não dá para escrever `expect(uptime).toBe(4.7)` — o teste passaria uma vez e falharia para
sempre depois.

A saída não é desistir de testar. É **testar o que sempre vale**, mesmo quando o valor exato
muda. Acrescente os dois testes dentro do `describe('HealthService', ...)`, logo depois do
primeiro:

```typescript
  it('informa há quanto tempo a aplicação está no ar', () => {
    const service = new HealthService()

    // `uptime` é o tempo em segundos desde que o processo subiu. Não dá para
    // prever o valor exato, então verificamos a única coisa que sempre vale:
    // ele nunca pode ser negativo.
    expect(service.getStatus().uptime).toBeGreaterThanOrEqual(0)
  })

  it('devolve a data no formato universal ISO 8601', () => {
    const service = new HealthService()
    const { timestamp } = service.getStatus()

    // Se o texto puder ser convertido de volta para uma data válida, o formato
    // está correto. Data inválida vira `NaN` ao ser convertida.
    expect(Number.isNaN(new Date(timestamp).getTime())).toBe(false)
  })
```

**Duas técnicas novas aparecem aqui:**

- `toBeGreaterThanOrEqual(0)` — em vez de exigir um valor exato, exige uma **faixa**. Um
  `uptime` negativo seria um defeito de verdade; 4,7 ou 812,3 segundos, não.
- Converter e conferir — `new Date(texto)` devolve uma data inválida quando o texto não é uma
  data reconhecível, e `.getTime()` dessa data inválida vira `NaN`. Então, se **não** deu
  `NaN`, o formato está certo. Testamos o formato sem precisar saber que dia é hoje.

> [!TIP]
> **Valor que muda sempre não é desculpa para não testar.** Quando o valor exato é
> imprevisível, pergunte-se: _"o que continua verdade em toda execução?"_ Quase sempre existe
> resposta — um intervalo, um formato, um tipo, um tamanho.

Rode de novo:

```bash
npx vitest run
```

```
 ✓ src/modules/health/health.spec.ts (3 tests)

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

---

## 🌐 Capítulo 5: Testando a rota — a promessa cumprida

Testar uma classe é fácil. E testar uma **rota HTTP**, que precisa de um servidor?

Aqui chegamos ao ponto alto da aula.

### O problema que a Aula 01 já tinha resolvido

Para testar uma rota, o caminho óbvio seria: subir o servidor numa porta, fazer uma
requisição de verdade, conferir a resposta, derrubar o servidor.

Isso funciona, mas é ruim: cada teste ficaria lento, dois testes rodando juntos brigariam
pela mesma porta, e um processo esquecido segurando a porta faria o teste falhar sem o
código ter absolutamente nada de errado.

**O Fastify tem uma solução melhor: o `app.inject()`.** Ele simula uma requisição HTTP
completa por dentro da aplicação, sem rede nenhuma. É como testar um carro num dinamômetro:
o motor funciona de verdade, as rodas giram de verdade, mas o carro não sai do lugar.

E para usar o `inject()`, você precisa de uma coisa: **conseguir montar o app sem iniciar o
servidor**.

É exatamente o que o `buildApp()` faz. Desde a Aula 01.

> [!IMPORTANT]
> Volte um instante ao `src/server.ts` e ao `src/app.ts`. Repare que o `server.ts` é o único
> que chama `app.listen()` — o que abre a porta de rede. O `app.ts` só monta e devolve.
>
> Essa separação parecia burocracia quatro aulas atrás. **É ela que torna esta aula possível.**
>
> Decisões de arquitetura funcionam assim: o custo aparece na hora, o benefício aparece
> meses depois. É por isso que vale confiar em algumas delas antes de entender por completo.

### Passo 1: Ajustando o `buildApp` para os testes

Tem um detalhe prático. Com o logger ligado, cada requisição simulada imprime várias linhas
de log, e o resultado dos testes fica ilegível.

Vamos permitir desligar o logger. Abra `src/app.ts` e deixe **completo** assim:

```typescript
/**
 * App — Montagem da instância do Fastify
 *
 * Este arquivo é responsável por:
 *   1. Criar a instância do Fastify com suas configurações.
 *   2. Registrar plugins globais.
 *   3. Registrar as rotas de cada módulo.
 *
 * Separamos a montagem do app (aqui) da inicialização do servidor (`server.ts`)
 * para facilitar os testes automatizados: nos testes importamos apenas o app,
 * sem precisar ocupar uma porta de rede.
 */

import Fastify from 'fastify'
import type { FastifyInstance } from 'fastify'
import { healthRoutes } from './modules/health/health.routes.ts'

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

  // No Fastify, cada conjunto de rotas é registrado como um plugin. Isso mantém
  // cada módulo isolado: um erro no registro de um não derruba os outros.
  app.register(healthRoutes)

  return app
}
```

**Duas coisas novas:**

- `options: BuildAppOptions = {}` — o `= {}` dá um valor padrão ao parâmetro. Assim
  `buildApp()` continua funcionando sem nenhum argumento, como no `server.ts`. **Nada quebra.**
- `options.logger ?? true` — o `??` já apareceu na Aula 01: "use o valor da esquerda; se ele
  não existir, use o da direita". Ou seja: ligado por padrão, desligado se você pedir.

### Passo 2: Testando a rota

Acrescente ao `src/modules/health/health.spec.ts`:

```typescript
import { describe, expect, it } from 'vitest'
import { buildApp } from '../../app.ts'
import { HealthService } from './health.service.ts'

// ... a suíte do HealthService que você já escreveu ...

describe('GET /health', () => {
  it('responde com código 200', async () => {
    const app = buildApp({ logger: false })

    const resposta = await app.inject({ method: 'GET', url: '/health' })

    expect(resposta.statusCode).toBe(200)

    await app.close()
  })
})
```

**O que está acontecendo:**

1. `buildApp({ logger: false })` monta a aplicação inteira em memória. Nenhuma porta é aberta.
2. `app.inject({ method: 'GET', url: '/health' })` simula a requisição. Ela percorre **todo**
   o caminho real: rota → controller → service → e volta.
3. `resposta.statusCode` é o que o cliente receberia.
4. `app.close()` libera os recursos ao final.

Repare no `async` e no `await`: requisições levam tempo, mesmo simuladas. Sem o `await`, o
teste conferiria o resultado antes de a resposta existir.

### Passo 3: Conferindo o conteúdo da resposta

```typescript
  it('devolve todos os campos combinados', async () => {
    const app = buildApp({ logger: false })

    const resposta = await app.inject({ method: 'GET', url: '/health' })
    const corpo = resposta.json()

    expect(corpo).toHaveProperty('status', 'ok')
    expect(corpo).toHaveProperty('uptime')
    expect(corpo).toHaveProperty('timestamp')
    expect(corpo).toHaveProperty('environment')

    await app.close()
  })
```

O `resposta.json()` converte o corpo da resposta em objeto. O `toHaveProperty` confere que o
campo existe — e, quando recebe um segundo argumento, que o valor também bate.

Repare que **não** verificamos o valor de `uptime` nem de `timestamp`. Eles mudam a cada
execução; exigir um valor exato faria o teste falhar sem que nada estivesse errado. Testamos
só o que é sempre verdade.

### Passo 4: Testando o caminho que dá errado

```typescript
  it('devolve 404 em um endereço que não existe', async () => {
    const app = buildApp({ logger: false })

    const resposta = await app.inject({ method: 'GET', url: '/endereco-que-nao-existe' })

    expect(resposta.statusCode).toBe(404)

    await app.close()
  })
```

> [!IMPORTANT]
> Este teste parece bobo, e é um dos mais importantes.
>
> Uma API que responde `200 OK` para **qualquer** endereço parece funcionar perfeitamente nos
> testes do caminho feliz. Só que ela está quebrada de um jeito grave: o aplicativo do cidadão
> pediria `/protocolos` com um erro de digitação, receberia "tudo certo" e mostraria uma tela
> vazia em vez de um erro.
>
> **Testar o que deve dar errado é o que garante que o certo é certo mesmo.**

### Passo 5: O arquivo completo

Construímos o `health.spec.ts` em pedaços, um passo de cada vez. Falta uma coisa: o arquivo
agora faz **duas** coisas bem diferentes — testa uma classe isolada e testa uma rota HTTP
inteira. Um leitor que abrir o arquivo daqui a seis meses merece saber disso na primeira linha.

Acrescente o cabeçalho no topo do arquivo e confira se o seu está **exatamente** assim:

```typescript
/**
 * Testes da funcionalidade de Health Check
 *
 * Repare no que este arquivo **não** faz: ele nunca abre uma porta de rede.
 *
 * Usamos `buildApp()` para montar a aplicação em memória e o `app.inject()` do
 * Fastify para simular uma requisição HTTP por dentro. É exatamente para isso
 * que `app.ts` é separado de `server.ts`: montar a aplicação e subir o servidor
 * são responsabilidades diferentes, e o teste só precisa da primeira.
 */

import { describe, expect, it } from 'vitest'
import { buildApp } from '../../app.ts'
import { HealthService } from './health.service.ts'

describe('HealthService', () => {
  it('responde com status "ok"', () => {
    const service = new HealthService()

    expect(service.getStatus().status).toBe('ok')
  })

  it('informa há quanto tempo a aplicação está no ar', () => {
    const service = new HealthService()

    // `uptime` é o tempo em segundos desde que o processo subiu. Não dá para
    // prever o valor exato, então verificamos a única coisa que sempre vale:
    // ele nunca pode ser negativo.
    expect(service.getStatus().uptime).toBeGreaterThanOrEqual(0)
  })

  it('devolve a data no formato universal ISO 8601', () => {
    const service = new HealthService()
    const { timestamp } = service.getStatus()

    // Se o texto puder ser convertido de volta para uma data válida, o formato
    // está correto. Data inválida vira `NaN` ao ser convertida.
    expect(Number.isNaN(new Date(timestamp).getTime())).toBe(false)
  })
})

describe('GET /health', () => {
  it('responde com código 200', async () => {
    const app = buildApp({ logger: false })

    const resposta = await app.inject({ method: 'GET', url: '/health' })

    expect(resposta.statusCode).toBe(200)

    await app.close()
  })

  it('devolve todos os campos combinados', async () => {
    const app = buildApp({ logger: false })

    const resposta = await app.inject({ method: 'GET', url: '/health' })
    const corpo = resposta.json()

    expect(corpo).toHaveProperty('status', 'ok')
    expect(corpo).toHaveProperty('uptime')
    expect(corpo).toHaveProperty('timestamp')
    expect(corpo).toHaveProperty('environment')

    await app.close()
  })

  it('devolve 404 em um endereço que não existe', async () => {
    const app = buildApp({ logger: false })

    const resposta = await app.inject({ method: 'GET', url: '/endereco-que-nao-existe' })

    // Testar o caminho que dá errado é tão importante quanto testar o que dá
    // certo. É o que garante que a API não responda "ok" para qualquer coisa.
    expect(resposta.statusCode).toBe(404)

    await app.close()
  })
})
```

Seis testes, dois grupos. O primeiro `describe` testa a classe sozinha; o segundo testa o
caminho completo, da rota até a resposta.

```bash
npx vitest run
```

```
 ✓ src/modules/health/health.spec.ts (6 tests)

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

---

## 🔍 Capítulo 6: Testando aquilo que ninguém costuma testar

Até aqui testamos o que o código **faz**. Agora vamos testar algo que quase todo projeto
esquece: **as mensagens de erro**.

### Por que mensagem de erro é comportamento, e não enfeite

Na Aula 04 escrevemos o `envSchema` com mensagens em português. A razão estava escrita no
próprio arquivo: quem lê essas mensagens é a nossa equipe, às vezes correndo, durante um
deploy que não subiu às onze da noite.

Ou seja: aquela mensagem **tem uma função**. Se ela sair errada, em inglês, ou genérica
demais, alguém perde minutos preciosos justamente na pior hora.

E tudo que tem função pode quebrar sem ninguém perceber. Logo, deve ser testado.

> [!IMPORTANT]
> É muito fácil escrever uma validação, conferir o caso que você imaginou, e nunca descobrir
> que o **outro** caminho produz uma mensagem inútil.
>
> Erro não testado é erro que só aparece para o usuário.

### Passo 6: Testando o contrato de ambiente

Crie `src/shared/env/env.schema.spec.ts`:

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

São 13 testes. Parece muito para um arquivo de configuração com três variáveis — e é
justamente esse o ponto.

> [!NOTE]
> Repare na função `mensagemDe`. Quando vários testes precisam da mesma preparação, vale
> extrair uma função auxiliar — o teste fica focado no que está sendo verificado, em vez de
> repetir a mesma mecânica cinco vezes.

### Por que tantos testes para três variáveis

Volte ao `env.schema.ts` da Aula 04 e conte as regras que você escreveu para o `PORT`: precisa
ser número, precisa ser **inteiro**, e precisa estar **entre 1 e 65535**. São três regras
diferentes, e cada uma pode quebrar sozinha.

Um único teste com `'8O80'` passaria mesmo se você tivesse esquecido a faixa de 1 a 65535 —
porque `'8O80'` já é recusado antes de chegar lá. Por isso cada regra ganha o seu teste:

| Teste                                 | A regra que ele protege                  |
| :------------------------------------ | :--------------------------------------- |
| `recusa uma porta que não é número`   | O valor precisa ser numérico             |
| `recusa uma porta com casas decimais` | Precisa ser inteiro — `80.5` não é porta |
| `recusa uma porta fora da faixa`      | Precisa estar entre 1 e 65535            |

**A regra prática:** um teste por regra, não um teste por campo. Quando um deles falhar, o nome
já diz qual regra você quebrou, sem precisar investigar.

E repare no primeiro teste de todos, o `aceita uma configuração completa e correta`. Ele parece
o mais inútil da lista — afinal, ele só confirma que o que deveria funcionar funciona. É
exatamente por isso que ele existe: sem ele, uma validação exagerada demais, que recusasse
**tudo**, faria os outros doze testes passarem numa boa.

> [!TIP]
> Uma suíte só com testes de "deve dar errado" é uma armadilha. Um `envSchema` que recusasse
> qualquer entrada passaria em todos eles — e derrubaria a API na primeira subida.
>
> **Sempre teste também o caminho que deve dar certo.**

### Os dois testes de `HOST` merecem atenção

Eles verificam o **mesmo campo**, o `HOST`, mas por motivos diferentes:

```typescript
expect(mensagemDe({ HOST: '' }, 'HOST')).toBe('não pode ficar vazio')
expect(mensagemDe({ HOST: 123 }, 'HOST')).toBe('é obrigatória e deve ser um texto')
```

Volte ao `env.schema.ts` e olhe a definição do `HOST`: ela tem `error` em **dois** lugares.

```typescript
HOST: z
  .string({ error: 'é obrigatória e deve ser um texto' })   // ← valor ausente ou de outro tipo
  .min(1, { error: 'não pode ficar vazio' })                 // ← veio, mas veio vazio
  .default('0.0.0.0'),
```

Cada mensagem cobre um caminho diferente da validação. Um teste sozinho passaria mesmo se a
outra estivesse errada — por isso são dois.

> [!TIP]
> **Este é o tipo de detalhe que só aparece quando você testa.**
>
> Se faltasse a mensagem do `z.string()`, o erro sairia assim:
>
> ```
> HOST: Invalid input: expected string, received number
> ```
>
> Em inglês, e no meio de um deploy. Rode um teste, apague o `{ error: ... }` do
> `z.string()` de propósito, e veja acontecer. Depois desfaça.
>
> Nada disso quebraria a aplicação. Ela subiria e funcionaria. Só que, no dia em que alguém
> errasse a configuração, a ajuda viria pior. **Testar é o que impede uma qualidade dessas de
> se perder em silêncio.**

### Rodando tudo

```bash
npx vitest run
```

```
 Test Files  2 passed (2)
      Tests  19 passed (19)
```

---

## 🏃 Capítulo 7: Os atalhos no package.json

Digitar `npx vitest run` toda vez é chato. E tem uma mudança mais importante junto.

### Passo 7: Os scripts

```json
    "test": "vitest run",
    "test:watch": "vitest",
```

- **`npm test`** roda todos os testes uma vez e termina. É o que se usa antes de commitar.
- **`npm run test:watch`** fica **vigiando** os arquivos: você salva, ele roda de novo na hora,
  e só os testes afetados pela alteração. É o modo do dia a dia — deixe rodando num terminal
  ao lado enquanto programa.

### Passo 8: Testes entram no portão

Agora o mais importante. Altere o script `check`:

```json
    "check": "npm run lint && npm run format:check && npm run test && npm run build"
```

A partir de agora, **um teste quebrado reprova o `npm run check`**. Lembre do `&&`: se o teste
falhar, o build nem chega a rodar.

Isso muda o status dos testes no projeto: eles deixam de ser algo que "seria bom rodar" e
passam a ser uma condição para o código seguir adiante. Na Aula 06, o GitHub vai rodar esse
mesmo comando a cada envio.

### Passo 9: Mantendo os testes fora da produção

Tem um detalhe que passa despercebido. Rode o build e olhe o que foi gerado:

```bash
npm run build
```

Se você procurar na pasta `dist`, vai encontrar lá dentro os arquivos `.spec.js` — os seus
testes, **compilados e prontos para irem para o servidor**.

Isso está errado por dois motivos: código de teste não tem função nenhuma em produção, e ele
depende do Vitest, um pacote que nem estará instalado no servidor.

Crie na raiz o arquivo `tsconfig.build.json`:

```json
{
  // Configuração usada APENAS pelo `npm run build`.
  //
  // Ela herda tudo do `tsconfig.json` e só acrescenta uma coisa: manter os
  // arquivos de teste fora da pasta `dist/`.
  //
  // Por que dois arquivos em vez de um:
  //   - O `tsconfig.json` inclui os testes, para que o editor e o `npm run lint`
  //     também os verifiquem. Teste sem verificação de tipo é teste que mente.
  //   - Este arquivo os exclui, porque código de teste não deve ir para o
  //     servidor de produção: aumenta a imagem à toa e depende de pacotes que
  //     nem estão instalados lá.
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "dist", "src/**/*.spec.ts"]
}
```

E aponte o build para ele:

```json
    "build": "tsc --project tsconfig.build.json",
```

Rode `npm run build` de novo e confira a pasta `dist`: nenhum `.spec.js`.

> [!NOTE]
> Repare no raciocínio, que vale além deste caso: os testes precisam ser **verificados** como
> o resto do código, mas não precisam ser **entregues** com ele. Duas necessidades diferentes,
> dois arquivos de configuração.

### Passo 10: A extensão do VS Code

Acrescente a extensão do Vitest ao `.vscode/extensions.json`. O arquivo **completo** deve
ficar assim:

```json
{
  // Quando alguém clona este repositório e abre no VS Code, ele sugere instalar
  // estas extensões automaticamente. Ninguém precisa lembrar de avisar.
  //
  // Esta lista cresce junto com o projeto: cada ferramenta nova que entra
  // acrescenta aqui a extensão correspondente.
  "recommendations": [
    // --- TypeScript: ler erro e testar rota sem sair do editor ---
    "YoavBls.pretty-ts-errors",
    "usernamehw.errorlens",
    "humao.rest-client",
    "streetsidesoftware.code-spell-checker",
    "streetsidesoftware.code-spell-checker-portuguese-brazilian",

    // --- Git: ver quem alterou cada linha, e quando ---
    "eamodio.gitlens",

    // --- Padronização do código ---
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "EditorConfig.EditorConfig",

    // --- Variáveis de ambiente ---
    "mikestead.dotenv",

    // --- Testes ---
    "vitest.explorer"
  ]
}
```

A extensão do Vitest mostra um ▶ ao lado de cada teste no editor, para rodar um teste isolado
com um clique, e marca com ✓ ou ✗ direto no arquivo. Investigar um teste que falhou fica bem
mais rápido.

### Passo 11: Conferindo o `package.json` completo

Este arquivo cresceu bastante desde a Aula 01, um pedaço por aula. Vale parar e olhar o
resultado inteiro. O seu deve estar **exatamente** assim:

```json
{
  "name": "curso_api",
  "version": "1.0.0",
  "description": "API RESTful backend construída durante o curso",
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
    "fastify": "^5.12.0",
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

_(Os números das versões podem variar um pouco — o npm instala o mais recente dentro da faixa
permitida. O que precisa bater são os **nomes** dos pacotes e os **scripts**.)_

> [!TIP]
> Repare em `dependencies` × `devDependencies`. Só duas coisas rodam no servidor de produção:
> o **Fastify**, que atende as requisições, e o **Zod**, que valida a configuração na partida.
> Todo o resto — compilador, linter, formatador, executor de testes — é ferramenta nossa, de
> desenvolvimento.
>
> Essa separação não é burocracia: é o que mantém a imagem que vai para o servidor pequena e
> com menos superfície de ataque.

> [!NOTE]
> **Uma diferença entre o seu projeto e o nosso**
>
> Se você abrir o repositório original, vai ver um script a mais no `check`, chamado
> `check:aulas`. Ele existe porque este repositório é, ao mesmo tempo, um curso: o script confere
> se cada arquivo mostrado nas aulas continua igual ao que está no disco.
>
> **Ele não faz parte do projeto que você está construindo** — é ferramenta de quem mantém o
> material. Por isso o seu `package.json` não o tem.

---

## 📄 Capítulo 8: Atualizando o README

Dois comandos novos entraram no `package.json` hoje, e um comando antigo mudou de
comportamento: o `npm run check` agora roda os testes também. As duas coisas precisam
aparecer no README.

Repare que a segunda é a mais fácil de esquecer — o comando continua com o mesmo nome, então
nada chama atenção. Só que a descrição virou mentira. Documentação envelhece assim: não por
falta de linha nova, e sim por linha velha que ninguém releu.

````markdown
# API do Curso

API RESTful construída durante o curso, construída com **Fastify + TypeScript**.

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

| Variável   | O que controla                                | Padrão        |
| :--------- | :-------------------------------------------- | :------------ |
| `NODE_ENV` | Ambiente: `development`, `test` ou `production` | `development` |
| `PORT`     | Porta em que a API escuta                     | `3333`        |
| `HOST`     | Endereço de rede que a API aceita             | `0.0.0.0`     |

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

| Método | Rota      | O que devolve                              |
| :----- | :-------- | :----------------------------------------- |
| `GET`  | `/health` | O estado da API: status, uptime e ambiente |
````

---

## 💾 Fechando o ciclo: mande para o GitHub

Você escreveu os primeiros testes do projeto, mudou o `package.json` e o
`tsconfig.build.json`, e atualizou o README. Feche o ciclo da Aula 02:

```bash
git add .
git commit -m "test: adiciona testes automatizados com vitest"
git push
```

Confira no navegador que a pasta `src` agora tem os arquivos `.spec.ts` ao lado do código que
eles testam. A partir de agora, todo o resto do curso pode contar com eles: qualquer alteração
que quebre o que já funcionava aparece na hora, e não semanas depois.

---

## ✅ Como saber que deu certo

| Verificação                                   | O que esperar                                               |
| :-------------------------------------------- | :---------------------------------------------------------- |
| `npm test`                                    | `Test Files 2 passed`, `Tests 19 passed`                    |
| `npm run check`                               | Passa nas quatro etapas: lint, formatação, testes e build   |
| `npm run build` + procurar `.spec` em `dist/` | **Nenhum** arquivo de teste                                 |
| Quebrar um teste de propósito                 | `npm run check` **falha**, e o build nem roda               |
| `npm run test:watch`                          | Fica rodando; ao salvar um arquivo, executa de novo sozinho |
| Abrir o `README.md`                           | Tem `npm test` e `npm run test:watch` na tabela de comandos |

---

## 🚨 Erros Comuns

**`No test files found`**
O Vitest procura arquivos terminados em `.spec.ts` ou `.test.ts` dentro de `src`. Confira o
nome do arquivo — `health.test.ts` funciona, `health-spec.ts` não.

**`Cannot find module './health.service.ts'`**
Confira o caminho relativo e a extensão. A regra do projeto continua a mesma: todo import é
relativo e leva `.ts`.

**O teste passa mas eu sei que o código está errado**
Provavelmente falta o `await`. Sem ele, o teste termina antes de a resposta chegar e a
asserção nunca chega a ser conferida. Toda vez que houver `inject`, tem que haver `await`.

**Os testes imprimem um monte de log e não consigo ler o resultado**
Faltou o `{ logger: false }` no `buildApp()`.

**`EADDRINUSE` durante os testes**
Não deveria acontecer: o `inject()` não abre porta. Se aconteceu, algum teste está chamando
`app.listen()` — o que nenhum teste deve fazer.

**Um teste passa sozinho mas falha junto com os outros**
Sinal clássico de teste que deixou sujeira para trás. Confira se todo `buildApp()` tem o
`app.close()` correspondente.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/05-gabarito.md`](./exercicios/05-gabarito.md).

**1. Veja falhar antes de confiar**
Escolha um teste que está passando e altere o valor esperado para algo errado. Rode
`npm test`. Leia a mensagem inteira. Qual informação ela te dá que ajudaria a consertar?
Depois desfaça.

**2. Um teste novo, do zero**
Escreva um teste verificando que o `HealthService` devolve o campo `environment` com o valor
`'test'` quando os testes rodam. _(Dica: investigue qual `NODE_ENV` o Vitest define sozinho —
o próprio teste vai te dizer, se você errar o valor esperado de propósito.)_

**3. Teste o método HTTP errado**
Acrescente um teste que faça `POST` em `/health` e confira o código de status devolvido. Qual
você esperava? Qual veio? Por que o Fastify responde assim?

**4. Quebre o projeto e veja o portão funcionar**
No `health.controller.ts`, troque o `200` por `201`. Rode `npm run check`. Em qual etapa ele
parou? O build chegou a rodar? Explique o papel do `&&`. Depois desfaça.

**5. Pergunta para responder por escrito**
Por que o `app.inject()` é melhor que subir o servidor de verdade numa porta para testar uma
rota? Dê pelo menos dois motivos, com suas palavras.

---

## 🎯 Resumo e Próximos Passos

Hoje o projeto passou a se defender sozinho.

O que ficou pronto:

- 19 testes automatizados rodando em menos de um segundo.
- Rotas testadas sem abrir porta de rede, graças à separação feita na Aula 01.
- Mensagens de erro tratadas como comportamento, e não como enfeite: testadas como o resto.
- Testes fazendo parte do portão: código quebrado não passa do `npm run check`.
- Testes fora da pasta de produção.

E o hábito que importa mais que tudo isso: **escrever o teste, vê-lo falhar, corrigir, vê-lo
passar**. Um teste que você nunca viu falhar não é confiável.

**E agora?**

Repare numa fragilidade que ainda existe: os testes só rodam **se alguém lembrar de rodá-los**.

Se um de vocês enviar código sem rodar `npm run check`, o projeto quebrado vai para o GitHub e
ninguém percebe até alguém tropeçar. Foi exatamente assim que este projeto acumulou problemas
antes de vocês chegarem.

Na **Aula 06** vamos tirar o "lembrar" da equação: o GitHub vai rodar lint, formatação, testes
e build **sozinho**, a cada envio de código. Se algo falhar, aparece um ✗ vermelho para todo o
time ver — antes de virar problema de alguém.

Até a próxima! 🚀
