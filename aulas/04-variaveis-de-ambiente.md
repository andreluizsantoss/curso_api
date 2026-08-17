# ⚙️ Aula 04: Variáveis de Ambiente e o Hábito de Falhar Cedo

Olá, equipe! Bem-vindos à **Aula 04**. 🎉

Nas três primeiras aulas construímos a API, colocamos ela no GitHub e ensinamos o computador
a cuidar da qualidade do código. Hoje vamos consertar um problema que está escondido no
projeto **desde a Aula 01**, e que ninguém notou — porque ele não dá erro.

Esse é o tipo mais perigoso de problema.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar o que é uma variável de ambiente e por que configuração não mora no código.
- Validar a configuração da aplicação com Zod, e fazer a API **recusar a partida** quando
  algo estiver errado.
- Organizar código compartilhado em `src/shared/` e importá-lo de qualquer módulo.
- Entender por que `.env` nunca vai para o Git, mas `.env.example` sempre vai.
- Documentar no `README.md` a configuração que o projeto passa a exigir.

## 📋 Pré-requisitos

- Aulas 01, 02 e 03 concluídas.
- `npm run check` passando no seu projeto.

---

## 💣 Capítulo 1: O bug que está no projeto agora

Antes de qualquer teoria, vamos ver o problema com os próprios olhos.

Abra o `src/server.ts` e olhe esta linha, que escrevemos na Aula 01:

```typescript
const PORT = Number(process.env.PORT) || 3333
```

Parece inofensiva, certo? Tenta ler a porta do ambiente; se não conseguir, usa 3333.

Agora **rode isto no terminal** (é para dar errado):

**Windows (PowerShell):**

```powershell
$env:PORT="8O80"; npm run dev
```

**Linux / Mac / Git Bash:**

```bash
PORT=8O80 npm run dev
```

> Repare com atenção: `8O80` tem a **letra O** no lugar do zero. É um erro de digitação que
> qualquer pessoa comete, e que ninguém enxerga relendo.

Olhe o log. A API subiu na porta **3333**.

Sem aviso. Sem erro. Sem nada.

### Por que isso aconteceu

Vamos destrinchar a linha, pedaço por pedaço:

1. `process.env.PORT` devolve o texto `"8O80"`.
2. `Number("8O80")` não consegue converter e devolve **`NaN`** (_Not a Number_, "não é um
   número").
3. Em JavaScript, `NaN` é considerado "falso" numa condição.
4. O operador `||` significa "se o que veio antes for falso, use o que vem depois".
5. Resultado: `PORT` vira **3333**, e o erro é engolido.

### Por que isso é grave de verdade

Imagine o dia do deploy da API do Curso. Quem configura o servidor digita `8O80` por
engano. O balanceador de carga é apontado para a porta 8080. A API sobe na 3333.

**O sistema fica fora do ar, e não há uma única mensagem de erro em lugar nenhum** para
indicar o motivo. O log diz "Servidor iniciado com sucesso". Alguém vai passar horas
procurando um problema que uma letra causou.

Pare o servidor (`Ctrl + C`) antes de continuar.

> [!IMPORTANT]
> **A ideia central desta aula, e leve ela para a carreira inteira:**
>
> É muito melhor um programa **recusar a partida** com uma mensagem clara do que subir com
> configuração errada e quebrar depois, na frente do usuário.
>
> Isso se chama **falhar rápido** (_fail fast_). O erro aparece no momento em que a pessoa
> responsável está olhando para a tela — e não três horas depois, para outra pessoa, sem
> contexto nenhum.

---

## 📖 Capítulo 2: O que é uma variável de ambiente?

Pense em um mesmo aplicativo instalado em três lugares diferentes:

| Onde        | Porta | Banco de dados                    |
| :---------- | :---- | :-------------------------------- |
| Sua máquina | 3333  | banco de teste local              |
| Homologação | 8080  | banco de homologação              |
| Produção    | 80    | banco real, com dados de cidadãos |

É **o mesmo código** nos três. O que muda é a **configuração**.

Variáveis de ambiente são valores que ficam guardados **fora do código**, no sistema
operacional onde o programa roda. O programa lê esses valores na partida e se adapta.

### A analogia do crachá

Pense no código como um funcionário e nas variáveis de ambiente como o **crachá** que ele
recebe ao entrar no prédio.

O funcionário é sempre o mesmo. Mas o crachá diz em qual andar ele vai trabalhar hoje e
quais portas ele pode abrir. Ele não precisa ser treinado de novo a cada prédio — ele só lê
o crachá.

### Por que configuração nunca fica dentro do código

1. **Segurança.** A senha do banco de produção não pode estar num arquivo versionado no Git,
   onde qualquer pessoa com acesso ao repositório a enxerga — e onde ela fica no histórico
   para sempre.
2. **Praticidade.** Mudar a porta não pode exigir alterar código, compilar de novo e fazer
   um novo deploy.
3. **O mesmo pacote em todo lugar.** O código que você testou em homologação precisa ser
   **exatamente** o mesmo que vai para produção. Se ele mudasse entre os ambientes, o teste
   não valeria de nada.

---

## 🔐 Capítulo 3: O arquivo `.env`

Digitar as variáveis na mão a cada vez que sobe o servidor seria insuportável. Por isso, em
desenvolvimento, usamos um arquivo chamado `.env`.

### Passo 1: Criando o modelo (`.env.example`)

Crie na raiz do projeto o arquivo `.env.example`:

```ini
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
```

> [!IMPORTANT]
> **Por que existem dois arquivos, e qual a diferença?**
>
> | Arquivo        | Vai para o Git? | O que contém                                                  |
> | :------------- | :-------------: | :------------------------------------------------------------ |
> | `.env.example` |   ✅ **sim**    | O **modelo**: quais variáveis existem, com valores de exemplo |
> | `.env`         |  ❌ **nunca**   | Os **valores reais** da sua máquina, incluindo segredos       |
>
> Sem o `.env.example`, um colega novo clona o projeto e não tem como adivinhar quais
> variáveis precisa criar. Ele descobriria na tentativa e erro, uma por vez.
>
> Com ele, é um comando só e o projeto sobe.

### Passo 2: Criando o seu `.env`

**Windows (PowerShell):**

```powershell
Copy-Item .env.example .env
```

**Linux / Mac / Git Bash:**

```bash
cp .env.example .env
```

### Passo 3: Conferindo que o guarda-costas está funcionando

Esta é a verificação mais importante da aula:

```bash
git status
```

Você deve ver o `.env.example` listado, e **o `.env` não deve aparecer em lugar nenhum**.

Isso acontece porque o `.gitignore` que criamos na Aula 02 já tinha a linha `.env`. Lembra
da seção "Variáveis de ambiente (senhas, tokens, etc.)"? Ela estava lá desde o primeiro dia,
esperando este momento.

> [!CAUTION]
> Se o `.env` **aparecer** no `git status`, pare tudo e confira o `.gitignore` antes de
> continuar. Segredo que entra no histórico do Git não sai mais — mesmo apagando o arquivo
> depois, o valor continua acessível em qualquer commit anterior.

---

## 📦 Capítulo 4: Quem lê o arquivo `.env`?

Aqui vem uma decisão do projeto que vale explicar, porque a maioria dos tutoriais na
internet faz diferente.

Durante muitos anos foi necessário instalar um pacote chamado `dotenv` só para ler o
arquivo `.env`. **Nós não vamos instalar nada disso.**

Desde a versão 20, o **Node.js lê o `.env` nativamente**, com uma flag na linha de comando:

```bash
node --env-file=.env arquivo.js
```

> [!TIP]
> **A lição que fica:** antes de instalar um pacote, confira se a plataforma já resolve o
> problema. Cada dependência a menos é um pacote a menos para atualizar, auditar e se
> preocupar quando surgir uma falha de segurança.
>
> É a regra da progressão estrita aplicada na prática: só instalamos o que é realmente
> necessário.

### Uma pegadinha importante das duas flags

Existem **duas** flags parecidas, e a diferença entre elas decide se a sua API sobe ou não
em produção:

| Flag                        | Se o arquivo não existir                  |
| :-------------------------- | :---------------------------------------- |
| `--env-file=.env`           | **Derruba o Node**, com código de saída 9 |
| `--env-file-if-exists=.env` | Avisa e continua normalmente              |

**Por que isso importa:** em produção **não existe** arquivo `.env`. Lá as variáveis vêm do
próprio ambiente do servidor ou do container. Se o comando de produção usasse
`--env-file=.env`, a API simplesmente não subiria.

Por isso cada comando usa uma flag diferente, e cada escolha tem um motivo:

- **`dev`** usa `--env-file`. Se você esqueceu de criar o `.env`, é melhor o erro aparecer
  agora, na sua máquina, do que a API subir com valores padrão e você não entender por quê.
- **`start`** usa `--env-file-if-exists`. Em produção o arquivo não existe, e isso é normal.

---

## 🛡️ Capítulo 5: Validando a configuração com Zod

Ler o arquivo é metade do trabalho. Falta a parte que resolve o bug do Capítulo 1:
**conferir se o conteúdo faz sentido**.

Para isso vamos usar o **Zod**, uma biblioteca de validação.

### Passo 4: Instalando o Zod

```bash
npm install zod
```

> Repare que desta vez **não** usamos o `-D`. O Zod entra em `dependencies`, e não em
> `devDependencies`, porque ele roda junto com a aplicação em produção — diferente do ESLint
> e do Prettier, que são ferramentas só de desenvolvimento.

### O que o Zod faz

O Zod deixa você descrever o **formato esperado** de um dado. Depois ele confere se o dado
real bate com a descrição, e ainda ensina ao TypeScript qual é o tipo resultante.

Pense nele como o formulário de papel de um cartório: os campos obrigatórios estão marcados,
cada um só aceita um tipo de informação, e o atendente devolve o formulário se algo estiver
errado — em vez de arquivar do jeito que veio.

### Passo 5: Criando o contrato

Crie a pasta `src/shared/env/` e, dentro dela, o arquivo `env.schema.ts`:

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

**Destrinchando as partes novas:**

- **`z.object({ ... })`** — descreve um objeto e o formato de cada campo dele.
- **`z.enum([...])`** — só aceita um dos valores da lista. Qualquer outro é recusado.
- **As duas mensagens do `HOST`** — repare que ele tem `error` em dois lugares: no `z.string()`
  e no `.min(1)`. Não é redundância. Cada um cobre uma situação diferente: o primeiro vale
  quando a variável **não veio** ou veio com o tipo errado; o segundo, quando ela veio mas
  está **vazia**. Sem o primeiro, esse caso cairia na mensagem padrão do Zod, que é em inglês.
  Na Aula 05 vamos escrever um teste que confere justamente isso.
- **`z.coerce.number()`** — _coerce_ significa "converter à força". Toda variável de ambiente
  chega como **texto**, então precisamos converter para número antes de validar. **É aqui que
  o bug do Capítulo 1 morre:** `"8O80"` vira `NaN` na conversão, e o Zod recusa `NaN`.
- **`.default(3333)`** — se a variável não vier, use este valor.
- **`z.infer<typeof envSchema>`** — a linha mais elegante do arquivo. Ela **deduz** o tipo
  TypeScript a partir do schema. Escrevemos a descrição uma vez só, e o tipo sai de graça.
  Não existe o risco clássico de mudar a validação e esquecer de atualizar o tipo.

### Passo 6: Validando na partida

Agora crie `src/shared/env/index.ts`:

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

import { envSchema } from './env.schema.ts'

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

**Três coisas para prestar atenção:**

**1. `safeParse` em vez de `parse`**

O Zod tem os dois. O `parse` lança uma exceção quando falha, e o programa morre com um monte
de texto técnico na tela. O `safeParse` devolve um objeto dizendo se deu certo, deixando
**nós** decidirmos como comunicar o problema.

Escolhemos o `safeParse` porque queremos uma mensagem que ajude de verdade quem está
tentando subir a aplicação às onze da noite.

**2. O laço `for` que mostra todos os problemas**

Repare que percorremos `resultado.error.issues` em vez de mostrar só o primeiro erro.

Se o `.env` tiver três variáveis erradas, você descobre as três **de uma vez**. Sem isso, a
pessoa corrige uma, roda de novo, descobre a segunda, corrige, roda de novo... Pequeno
detalhe, grande diferença no dia a dia.

**3. O comentário `eslint-disable-next-line`**

Na Aula 03 configuramos o ESLint para reclamar de `console.log`. E agora estamos usando
`console.error` de propósito.

Isso **não** é contradição. É o caso legítimo: neste ponto da partida, o logger do Fastify
ainda não existe — o app só é montado depois que a configuração é lida. Não há alternativa.

O que **não** se faz é desligar a regra no projeto inteiro por causa de três linhas. O certo
é desligar exatamente onde é necessário, **com o motivo escrito ao lado**. O texto depois do
`--` é obrigatório na nossa equipe: uma exceção sem justificativa vira mistério para quem
ler o código daqui a um ano.

---

## 🔗 Capítulo 6: Importando o novo módulo

Nosso módulo mora em `src/shared/env/`. Para usá-lo, basta o caminho relativo até ele:

```typescript
// de dentro de src/server.ts
import { env } from './shared/env/index.ts'

// de dentro de src/modules/health/health.service.ts
import { env } from '../../shared/env/index.ts'
```

Cada `../` significa "suba uma pasta". Do `health.service.ts`, subir duas vezes leva de
`src/modules/health/` até `src/`, e de lá descemos para `shared/env/`.

> [!IMPORTANT]
> **A regra de import do projeto, em uma frase:**
>
> **Todo import é relativo e leva `.ts`.**
>
> Só isso. O compilador troca a extensão para `.js` na hora do build — exatamente como você
> conferiu na Aula 01, comparando o seu `src/app.ts` com o `dist/app.js` gerado.

> [!NOTE]
> **Por que não usamos um "atalho" de import?**
>
> Você vai encontrar muitos projetos com atalhos do tipo `@/shared/env`, que evitam os
> `../../`. Este projeto **não usa** — e a decisão foi tomada depois de tentar e voltar atrás.
>
> Dois motivos:
>
> **1. O ganho aqui é pequeno.** A nossa arquitetura organiza por funcionalidade, o que mantém
> as pastas rasas. O caminho mais longo do projeto inteiro tem **dois** níveis (`../../`) — não
> é aquele `../../../../` que realmente atrapalha a leitura.
>
> **2. O custo é alto e escondido.** Atalhos precisam de configuração, e essa configuração
> costuma funcionar em um lugar e falhar em outro. Uma das formas que testamos chegava a fazer
> o modo de desenvolvimento carregar **código compilado antigo** em vez do arquivo que você
> acabou de salvar: você edita, salva, e nada muda — sem erro, sem aviso.
>
> É a mesma família de armadilha que você viu no Capítulo 1 desta aula. Um mecanismo que
> funciona sempre vale mais do que um mecanismo mais bonito que às vezes mente.
>
> Vale guardar os caminhos que **não** deram certo: eles ensinam tanto quanto os que deram.

---

## 🔧 Capítulo 7: Usando a configuração validada

Agora vamos trocar as leituras diretas de `process.env` pelo objeto `env`.

### Passo 7: Atualizando os scripts

No `package.json`, altere **dois** scripts:

```json
    "dev": "tsx watch --env-file=.env src/server.ts",
    "start": "node --env-file-if-exists=.env dist/server.js",
```

(As flags diferentes são as do Capítulo 4 — se não lembrar do motivo, vale reler.)

### Passo 8: Atualizando o `server.ts`

Abra `src/server.ts` e deixe **exatamente** assim:

```typescript
/**
 * Server — Ponto de entrada da aplicação
 *
 * Este arquivo é responsável APENAS por iniciar o servidor HTTP na porta
 * configurada. Toda a montagem do Fastify (plugins e rotas) está em `app.ts`.
 *
 * Essa separação permite que, nos testes automatizados, importemos apenas o
 * `app.ts` sem precisar abrir uma porta de rede real.
 */

import { buildApp } from './app.ts'
import { env } from './shared/env/index.ts'

/**
 * Sobe o servidor HTTP e o deixa ouvindo requisições.
 */
async function start(): Promise<void> {
  const app = buildApp()

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

**Compare com a versão anterior.** Aquelas duas linhas de conversão e valor reserva
sumiram por completo:

```typescript
const PORT = Number(process.env.PORT) || 3333   // ❌ não existe mais
const HOST = process.env.HOST ?? '0.0.0.0'      // ❌ não existe mais
```

O arquivo ficou **menor** e mais seguro ao mesmo tempo. Isso não é coincidência: quando a
validação acontece em um lugar só, o resto do código para de se defender sozinho.

### Passo 9: Atualizando o `health.service.ts`

Abra `src/modules/health/health.service.ts`. São duas alterações pequenas — o import no topo
e a linha do `environment`:

```typescript
/**
 * HealthService
 *
 * Concentra a lógica de negócio da funcionalidade de Health Check (checagem de
 * saúde). É ele quem sabe COMO montar a resposta; o controller apenas pede.
 *
 * Separar a lógica (aqui) da camada HTTP (controller) é o que nos permite testar
 * esta classe sem simular uma requisição da internet.
 */

import { env } from '../../shared/env/index.ts'

/**
 * Formato da resposta do health check.
 */
export interface HealthStatus {
  status: string
  uptime: number
  timestamp: string
  environment: string
}

export class HealthService {
  /**
   * Coleta o estado atual da aplicação.
   *
   * @returns Objeto com o status, há quanto tempo a API está no ar e em qual
   *          ambiente ela está rodando.
   */
  getStatus(): HealthStatus {
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

Com isso, **não existe mais nenhum `process.env` espalhado pelo projeto**. Toda a
configuração passa por um único portão validado.

### Passo 10: A extensão do `.env` no VS Code

Adicione ao `.vscode/extensions.json`:

```json
    // --- Aula 04: Variáveis de ambiente ---
    "mikestead.dotenv"
```

Ela colore os arquivos `.env`, separando visualmente o nome da variável do valor. Ajuda a
enxergar erro de digitação — como aquele `8O80` do começo da aula.

---

## 🧪 Capítulo 8: Vendo funcionar

Hora da recompensa. Vamos repetir o experimento do Capítulo 1.

### O caminho feliz

```bash
npm run dev
```

Repare no log: além da porta e do endereço, agora aparece também o ambiente.

```json
{"level":30,"port":3333,"host":"0.0.0.0","ambiente":"development","msg":"Servidor iniciado com sucesso"}
```

### O erro que antes passava despercebido

Pare o servidor e abra o seu `.env`. Troque a porta pelo erro de digitação do começo da aula:

```ini
PORT=8O80
```

Rode `npm run dev` de novo:

```
❌ Configuração inválida. A API não foi iniciada.

   PORT: deve ser um número inteiro (recebido: "8O80")

   Confira o seu arquivo .env. O modelo está em .env.example.
```

**Isso é a aula inteira em cinco linhas.** O mesmo erro que antes fazia a API subir calada na
porta errada agora impede a partida, diz exatamente qual variável está errada, o que se
esperava dela e o que foi recebido.

### Vários erros de uma vez

Deixe o `.env` assim, com dois problemas:

```ini
NODE_ENV=producao
PORT=99999
```

```
❌ Configuração inválida. A API não foi iniciada.

   NODE_ENV: deve ser "development", "test" ou "production" (recebido: "producao")
   PORT: deve estar entre 1 e 65535 (recebido: "99999")

   Confira o seu arquivo .env. O modelo está em .env.example.
```

Os dois problemas aparecem juntos. Repare também que `producao` (sem acento, em português)
foi recusado — exatamente o tipo de engano que faria a API se comportar como se estivesse em
desenvolvimento dentro do servidor de produção.

**Agora corrija o seu `.env`** de volta para os valores do `.env.example` antes de continuar.

### Como a produção enxerga isso

Em produção não existe arquivo `.env`. Vamos simular:

```bash
npm run build
```

**Windows (PowerShell):**

```powershell
$env:PORT="4100"; $env:NODE_ENV="production"; npm start
```

**Linux / Mac / Git Bash:**

```bash
PORT=4100 NODE_ENV=production npm start
```

Repare no log: o valor que veio do ambiente (`4100`) **venceu** o que estava no `.env` (`3333`).
Essa é a ordem correta — o servidor manda mais que o arquivo local.

**Agora o segundo experimento.** Pare o servidor, renomeie o `.env` temporariamente
(`Rename-Item .env .env.guardado` no PowerShell, `mv .env .env.guardado` no Linux/Mac) e rode
`npm start` de novo. Desta vez aparece:

```
.env not found. Continuing without it.
```

E a API **sobe do mesmo jeito** — é o `--env-file-if-exists` fazendo o trabalho dele, e é
exatamente assim que ela vai subir no servidor do órgão, onde esse arquivo não existe. Devolva o
nome do arquivo ao normal antes de continuar.

---

## 📄 Capítulo 9: Atualizando o README

Você acabou de criar uma exigência nova: quem clonar este projeto **precisa** criar um `.env`
antes de rodar `npm run dev`, senão a API nem sobe. Isso é o tipo de informação que não pode
morar só na sua cabeça.

Lembre da regra da Aula 03: quem muda o que o projeto exige, atualiza o README na mesma
tarefa. Desta vez a mudança não é um comando novo — é uma seção nova, `## Configuração`, logo
depois do "Começando":

````markdown
# API do Curso

API RESTful do sistema API do Curso, construída com **Fastify + TypeScript**.

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
````

> [!TIP]
> Repare no que essa seção evita: a pessoa nova clona o projeto, roda `npm run dev`, toma um
> erro na cara e perde meia hora até alguém dizer "ah, tem que copiar o `.env.example`". Três
> linhas de README compram essa meia hora de volta, para sempre e para todo mundo.

---

## 💾 Fechando o ciclo: mande para o GitHub

Hoje você criou o módulo de configuração, o `.env` e o `.env.example`, alterou o
`package.json`, o `server.ts` e o `health.service.ts`, e documentou a configuração no README.
Feche o ciclo da Aula 02:

```bash
git add .
git commit -m "feat: valida variaveis de ambiente com zod"
git push
```

Confira no navegador: o `.env.example` está lá, e o `.env` **não** está. É o `.gitignore` da
Aula 02 fazendo exatamente o trabalho para o qual foi criado — a senha do banco nunca sai da sua
máquina.

---

## ✅ Como saber que deu certo

| Verificação                                                                                | O que esperar                                            |
| :----------------------------------------------------------------------------------------- | :------------------------------------------------------- |
| `npm run check`                                                                            | Encerra com sucesso, sem erro em nenhuma das três etapas |
| `npm run dev`                                                                              | Sobe e o log mostra `"ambiente":"development"`           |
| `http://localhost:3333/health`                                                             | Devolve o JSON com `"environment": "development"`        |
| `$env:PORT="abc"; npm run dev` (PowerShell)<br>`PORT=abc npm run dev` (Linux/Mac/Git Bash) | **Recusa a partida** com a mensagem em português         |
| `git status`                                                                               | Mostra o `.env.example`, e **não** mostra o `.env`       |
| Buscar `process.env` no `src/`                                                             | Só é **lido** em `src/shared/env/index.ts`               |
| Abrir o `README.md`                                                                        | Tem a seção `## Configuração`, com as três variáveis     |

> [!NOTE]
> Depois de testar com `$env:PORT`, essa variável continua valendo para o resto daquela janela
> de terminal. Feche e abra o terminal, ou rode `Remove-Item Env:\PORT`, antes de seguir.

A última linha é a que fecha o item **A-01** do nosso checklist de produção. Confira você
mesmo com `Ctrl + Shift + F` no VS Code, buscando por `process.env`.

> [!NOTE]
> A busca vai encontrar `process.env` também dentro de **comentários** — no
> `health.service.ts` e no cabeçalho do próprio `index.ts`. Isso é esperado: são textos
> explicando a regra, não código lendo variável.
>
> O que importa é que a **leitura de verdade** aconteça em um lugar só. Saber distinguir
> "apareceu na busca" de "está sendo executado" é uma habilidade que vale para qualquer
> investigação de código daqui pra frente.

> [!CAUTION]
> Se o `.env` aparecer no `git status`, **não faça commit**. Corrija o `.gitignore` primeiro.

---

## 🚨 Erros Comuns

**`Cannot find module '../../shared/env/index.ts'`**
Confira a quantidade de `../`. De `src/modules/health/` até `src/shared/` são **dois**
níveis para subir. Contar errado é o engano mais comum com caminho relativo.

**`.env: not found` e o Node encerra**
Você rodou `npm run dev` sem ter criado o `.env`. Copie a partir do modelo:
`cp .env.example .env` (ou `Copy-Item .env.example .env` no PowerShell). Este erro é
proposital: é melhor avisar agora do que subir com valores padrão.

**A API sobe, mas ignora o que está no `.env`**
Confira se o script tem a flag `--env-file=.env`. Sem ela o arquivo é apenas um texto qualquer
na pasta, que ninguém lê.

**Alterei o `.env` e nada mudou**
O `.env` é lido **uma única vez**, na partida. Pare o servidor e suba de novo. O `tsx watch`
vigia arquivos `.ts`, não o `.env`.

**`env.PORT` aparece como `string` no editor**
Você provavelmente está importando de `process.env` em vez do módulo `shared/env`. Confira o
import no topo do arquivo.

**O ESLint reclama do `console.error` no `index.ts`**
Falta o comentário `// eslint-disable-next-line no-console -- motivo`. Ele precisa estar na
linha **imediatamente anterior** ao `console.error`, e o texto depois do `--` é obrigatório
na nossa equipe.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/04-gabarito.md`](./exercicios/04-gabarito.md).

**1. Uma variável nova**
Adicione uma variável `NOME_DA_API`, do tipo texto, com valor padrão
`'API do Curso'`. Depois exiba esse valor no log de partida do `server.ts`. Lembre-se
de atualizar também o `.env.example` — e explique por que esse último passo importa.

**2. Torne uma variável obrigatória**
Remova o `.default('0.0.0.0')` do `HOST` e apague a linha `HOST` do seu `.env`. O que
acontece? Qual mensagem aparece? Depois desfaça as duas alterações.

**3. Investigue o `coerce`**
No schema, troque `z.coerce.number()` por `z.number()` (sem o `coerce`) e rode com um `.env`
válido. A API sobe? Por quê? Explique com suas palavras o que o `coerce` está fazendo.
Depois desfaça.

**4. Descubra a ordem de precedência**
Com `PORT=3333` no seu `.env`, rode a versão do seu terminal:

**Windows (PowerShell):**

```powershell
$env:PORT="5000"; npm run dev
```

**Linux / Mac / Git Bash:**

```bash
PORT=5000 npm run dev
```

Em qual porta a API subiu? Qual valor venceu? Por que essa ordem é a correta para produção?
(No PowerShell, lembre de rodar `Remove-Item Env:\PORT` depois, ou a variável continua valendo
naquela janela.)

**5. Pergunta para responder por escrito**
Explique, com suas palavras, por que o `.env.example` vai para o Git e o `.env` não. O que
aconteceria de concreto se fosse ao contrário?

---

## 🎯 Resumo e Próximos Passos

Hoje vocês corrigiram um problema real de produção que estava escondido desde a Aula 01.

O que ficou pronto:

- Configuração fora do código, validada na partida.
- A API **recusa subir** com configuração errada, e explica exatamente o que está errado.
- Um único ponto do projeto lê `process.env`. O resto usa um objeto tipado.
- Um atalho de import que funciona igual em desenvolvimento e em produção.
- `.env.example` versionado, para qualquer pessoa subir o projeto em um comando.
- Zero dependência a mais para ler o arquivo — a plataforma já fazia isso.

E, mais importante que qualquer linha de código: o hábito de **falhar rápido e falhar claro**.

**E agora?**

Repare em uma coisa. Nesta aula, para saber se tudo continuava funcionando, você precisou
subir o servidor e conferir na mão. De novo. Igual nas três aulas anteriores.

Imagine isso com trinta rotas, e com um prazo em cima. Ninguém testa trinta rotas
manualmente a cada alteração — e é exatamente aí que os bugs passam.

Na **Aula 05** vamos ensinar o computador a testar a API sozinho. E é lá que aquela separação
entre `app.ts` e `server.ts`, que fizemos lá atrás na Aula 01 sem explicar direito o motivo,
finalmente vai mostrar para que serve.

Até a próxima! 🚀
