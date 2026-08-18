# 🗄️ Aula 13: Banco de dados — Prisma, MySQL e a camada Repository

A Aula 12 deixou um MySQL de pé: saudável, com nome de rede, volume que sobrevive ao `down` e
um banco de teste ao lado. E a API **não fala com ele**. Não existe uma linha de código que
abra conexão, nem tabela, nem endereço de banco em lugar nenhum.

Enquanto isso, tudo o que a API responde vem do próprio processo. Ela não guarda nada — e
"não guardar nada" é o que separa uma demonstração de um sistema.

Nesta aula entram três coisas que andam juntas:

1. **O Prisma**, que traduz TypeScript em consulta ao banco, com o editor sabendo os nomes das
   colunas antes de você digitá-los.
2. **As migrations**, que transformam cada alteração de tabela em um arquivo versionado no Git,
   ao lado do código que depende dela.
3. **A camada Repository**, que é o único lugar do projeto autorizado a conversar com o banco.

> [!IMPORTANT]
> O Prisma mudou bastante na versão 7, e a maior parte do material que você vai encontrar por
> aí ainda descreve a anterior. Esta aula aponta cada diferença na hora em que ela aparece —
> não para decorar, mas para você reconhecer material desatualizado quando esbarrar nele.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Declarar uma tabela em um schema e transformá-la em SQL versionado, com um comando.
- Explicar por que `prisma migrate dev` precisa de um **banco de sombra**, e por que dar
  permissão de criar banco ao usuário da aplicação é a saída errada.
- Ler o SQL que a migration gerou, e não confiar nele por fé.
- Escrever repository e service, sabendo dizer o que pertence a cada um.
- Rodar testes contra um **banco de verdade** sem destruir o banco em que você trabalha.
- Reconhecer o erro clássico de acrescentar coluna obrigatória a uma tabela com dados.
- Manter o cliente gerado fora do Git e ainda assim ter a imagem Docker funcionando.

---

## 📋 Pré-requisitos

Você precisa ter concluído a **Aula 12** e ter:

- o Docker Desktop em execução, com o ambiente da aula passada funcionando;
- o projeto passando no `npm run check`;
- o terminal aberto na pasta do projeto.

Confirme:

```bash
npm run compose:up
docker compose ps
```

Os dois serviços em `Up`, o `mysql` marcado `(healthy)`.

---

## 💣 Capítulo 1: A dor — o banco está lá, e ninguém fala com ele

Entre no banco e olhe o que existe:

```bash
docker compose exec mysql mysql -ucurso_api -ptroque-esta-senha-tambem -e "SHOW TABLES FROM curso_api;"
```

Silêncio. Nenhuma tabela.

Agora pense em como você criaria uma. A resposta imediata é "escrevo o `CREATE TABLE` na mão".
Funciona — e cria três problemas no mesmo instante:

| O que acontece                                          | Por que dói                                                            |
| :------------------------------------------------------ | :--------------------------------------------------------------------- |
| A estrutura passa a existir **só dentro daquele banco** | Quem clonar o projeto amanhã não tem como recriá-la                    |
| Ninguém revisa a alteração                              | Um `DROP COLUMN` digitado com pressa não passa por Pull Request nenhum |
| O código e o banco podem discordar                      | E você só descobre quando a consulta falha, com gente usando           |

É o mesmo problema da Aula 12, em outra camada: **conhecimento que mora fora do repositório**.
Lá era como subir o ambiente; aqui é qual a estrutura dos dados.

---

## 📖 Capítulo 2: O que o Prisma é (e o que mudou na versão 7)

**A analogia.** Sem ORM, você conversa com o banco escrevendo SQL em texto: se errar o nome de
uma coluna, descobre quando a consulta rodar. Com o Prisma, você declara a estrutura uma vez, e
ele gera **código TypeScript** a partir dela — o editor completa os nomes das colunas, e errar
um deles vira erro de compilação, não erro de produção.

O Prisma tem três partes, e vale saber o nome de cada uma:

| Peça               | O que faz                                                |
| :----------------- | :------------------------------------------------------- |
| `schema.prisma`    | Declara as tabelas e os campos. É a fonte da verdade     |
| Prisma **Migrate** | Compara o schema com o banco e gera o SQL da diferença   |
| Prisma **Client**  | O código gerado que você importa para fazer as consultas |

### O que mudou na versão 7

Quatro diferenças, e você vai esbarrar nas quatro:

| No Prisma 7                                               | No material antigo                                 |
| :-------------------------------------------------------- | :------------------------------------------------- |
| `PrismaClient` **exige** um driver adapter                | `new PrismaClient()` sem argumento nenhum          |
| A URL do banco fica fora do `schema.prisma`               | `url = env("DATABASE_URL")` dentro do `datasource` |
| O CLI **não** lê o arquivo `.env` sozinho                 | O `.env` era lido automaticamente                  |
| O gerador `prisma-client` escreve TypeScript na sua pasta | `prisma-client-js`, escondido em `node_modules`    |

Nenhuma delas é capricho: as três primeiras deixam explícito o que antes era mágica, e a quarta
faz o código gerado ser algo que você pode abrir e ler.

---

## 🛠️ Capítulo 3: Instalando e configurando

### Passo 1 — os pacotes

```bash
npm install @prisma/client @prisma/adapter-mariadb
npm install -D prisma
```

Três pacotes, e a divisão importa:

| Pacote                    | Onde                   | Para quê                                              |
| :------------------------ | :--------------------- | :---------------------------------------------------- |
| `@prisma/client`          | dependência            | O que a API usa em produção para consultar            |
| `@prisma/adapter-mariadb` | dependência            | O driver que fala o protocolo do MySQL                |
| `prisma`                  | dependência de **dev** | O CLI: gera o cliente, cria migrations, abre o Studio |

O CLI é ferramenta de quem escreve o código, não de quem o executa — mesma lógica do TypeScript
e do Vitest desde a Aula 05.

> [!NOTE]
> O driver do MySQL se chama `adapter-mariadb` porque os dois bancos falam o mesmo protocolo. O
> nome estranha, e está certo.

### Passo 2 — o `prisma.config.ts`

Crie o arquivo na **raiz** do projeto. Ele diz ao CLI onde está o schema, onde ficam as
migrations e como chegar ao banco.

O arquivo completo está no Capítulo 13. Por enquanto, as duas decisões que ele carrega:

**Como o CLI lê o `.env`.** O Node 24 tem `process.loadEnvFile()` embutido — nenhuma dependência
nova. Mas com um `if` na frente:

```ts
if (existsSync('.env')) {
  process.loadEnvFile()
}
```

O `if` não é zelo exagerado. Dentro do container **não existe `.env`**: o `.dockerignore` o
mantém fora da imagem desde a Aula 10, e lá a configuração chega por variável de ambiente. Sem
essa condição, o build da imagem morre com:

```
Failed to load config file: Error: ENOENT: no such file or directory, open '.env'
```

É o mesmo motivo do `--env-file-if-exists` que o `npm start` usa.

**Como o endereço do banco chega.** Lido de `process.env` direto, e não pelo ajudante `env()` do
Prisma. O motivo também foi medido: o `env()` **exige** a variável no instante em que o arquivo é
carregado, inclusive durante o `prisma generate` — que roda dentro do build da imagem, onde não
existe (e não deve existir) endereço de banco nenhum. Gerar o cliente é traduzir schema em
TypeScript; não conecta em lugar algum.

### Passo 3 — as variáveis

No `.env.example` e no seu `.env`:

```bash
DATABASE_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api"
DATABASE_TEST_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api_test"
SHADOW_DATABASE_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api_shadow"
```

> [!TIP]
> Se você trocou a `MYSQL_PORT` na Aula 12 porque já tinha um MySQL na sua máquina, use aqui a
> mesma porta. Estas três URLs falam com o banco **de fora** do Compose.

E a `DATABASE_URL` entra no `envSchema`, como toda variável que a API consome — sem valor
padrão, de propósito:

```ts
  DATABASE_URL: z
    .string({ error: 'é obrigatória e deve ser um texto' })
    .min(1, { error: 'é obrigatória — copie o valor do .env.example' })
    .regex(/^mysql:\/\/.+/, {
      error: 'deve começar com "mysql://" (ex.: mysql://usuario:senha@localhost:3306/banco)',
    }),
```

**Uma API que guarda dado e sobe apontando para um banco inventado é pior do que uma API que não
sobe:** ela funciona até a primeira consulta.

Repare também nas outras duas: `DATABASE_TEST_URL` e `SHADOW_DATABASE_URL` **não** entram no
schema. Quem as consome não é a API — é o Vitest e o CLI do Prisma. Cada variável é declarada
onde é lida, e é a mesma regra que manteve as `MYSQL_*` fora do schema na aula passada.

---

## 🧱 Capítulo 4: O schema e a primeira migration

### O modelo

Crie `prisma/schema.prisma` (arquivo completo no Capítulo 13). O coração dele é:

```prisma
model Cidadao {
  id           String   @id @default(uuid())
  nome         String
  cpf          String   @unique
  email        String?
  criadoEm     DateTime @default(now())
  atualizadoEm DateTime @updatedAt

  @@map("cidadaos")
}
```

| Marcação           | O que significa                                                             |
| :----------------- | :-------------------------------------------------------------------------- |
| `@id`              | Chave primária                                                              |
| `@default(uuid())` | O valor nasce sozinho, e não é sequencial — não dá para adivinhar o próximo |
| `@unique`          | O banco **recusa** um segundo registro com o mesmo CPF                      |
| `String?`          | Campo opcional. Sem a interrogação, é obrigatório                           |
| `@updatedAt`       | O Prisma atualiza sozinho a cada alteração                                  |
| `@@map`            | O modelo se chama `Cidadao` no código e `cidadaos` no banco                 |

### A primeira tentativa, que falha

```bash
npx prisma migrate dev --name cria-tabela-cidadaos
```

```
Error: P3014

Prisma Migrate could not create the shadow database. Please make sure the database
user has permission to create databases.

Original error: User was denied access on the database
`prisma_migrate_shadow_db_99250a5b-31d5-4b4f-8bae-89be697eca60`
```

**Leia com atenção, porque a saída fácil aqui é a errada.**

O `migrate dev` cria um banco descartável — o _shadow database_ — onde aplica todas as
migrations em ordem para conferir se elas produzem exatamente o schema declarado. É essa
conferência que detecta migration alterada à mão depois de aplicada.

Para criar esse banco, ele precisaria de permissão de **criar bancos**. E o usuário da aplicação
não tem essa permissão — o que é uma decisão de segurança da Aula 12, não um descuido: a
aplicação não é administradora do banco, exatamente como não é `root` dentro do container.

A saída errada é dar a permissão. A certa é **entregar o banco pronto**.

### A correção

No script de inicialização do MySQL, ao lado do banco de teste:

```sql
CREATE DATABASE IF NOT EXISTS curso_api_shadow;
GRANT ALL PRIVILEGES ON curso_api_shadow.* TO 'curso_api'@'%';
FLUSH PRIVILEGES;
```

E no `prisma.config.ts`, o apontamento:

```ts
    shadowDatabaseUrl: process.env.SHADOW_DATABASE_URL ?? '',
```

> [!WARNING]
> O script de inicialização roda **uma única vez**, quando o MySQL cria a pasta de dados. Para
> ele valer, o volume precisa nascer de novo:
>
> ```bash
> docker compose down -v
> npm run compose:up
> ```
>
> Este é o caso legítimo do `-v` que a Aula 12 mencionou. Ele apaga o banco inteiro, e aqui isso
> é o desejado — não há nada dentro ainda.

Confira que os três bancos nasceram:

```bash
docker compose exec mysql mysql -ucurso_api -ptroque-esta-senha-tambem -e "SHOW DATABASES;"
```

```
curso_api
curso_api_shadow
curso_api_test
```

### Agora sim

```bash
npx prisma migrate dev --name cria-tabela-cidadaos
```

```
Applying migration `20260818182751_cria_tabela_cidadaos`

The following migration(s) have been created and applied from new schema changes:

prisma/migrations/
  └─ 20260818182751_cria_tabela_cidadaos/
    └─ migration.sql

Your database is now in sync with your schema.
```

### Leia o SQL. Sempre

```bash
cat prisma/migrations/*/migration.sql
```

```sql
-- CreateTable
CREATE TABLE `cidadaos` (
    `id` VARCHAR(191) NOT NULL,
    `nome` VARCHAR(191) NOT NULL,
    `cpf` VARCHAR(191) NOT NULL,
    `email` VARCHAR(191) NULL,
    `criadoEm` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    `atualizadoEm` DATETIME(3) NOT NULL,

    UNIQUE INDEX `cidadaos_cpf_key`(`cpf`),
    PRIMARY KEY (`id`)
) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

É SQL comum, sem mágica nenhuma. **Este arquivo vai para o Git**, e é ele — não o
`schema.prisma` — que será executado no servidor. Ler o SQL antes de aplicar é o hábito que
separa quem confia da ferramenta de quem confere.

E confirme no banco:

```bash
docker compose exec mysql mysql -ucurso_api -ptroque-esta-senha-tambem curso_api -e "DESCRIBE cidadaos;"
```

```
Field           Type            Null    Key     Default
id              varchar(191)    NO      PRI     NULL
nome            varchar(191)    NO              NULL
cpf             varchar(191)    NO      UNI     NULL
email           varchar(191)    YES             NULL
criadoEm        datetime(3)     NO              CURRENT_TIMESTAMP(3)
atualizadoEm    datetime(3)     NO              NULL
```

---

## ⚙️ Capítulo 5: O cliente gerado

A tabela existe. O código ainda não sabe disso — o cliente precisa ser **gerado** a partir do
schema:

```bash
npx prisma generate
```

```
✔ Generated Prisma Client (7.9.1) to ./src/generated/prisma in 25ms
```

> [!IMPORTANT]
> O `migrate dev` **não** gera o cliente sozinho nesta versão. Se você alterar o schema, rodar a
> migration e o editor continuar reclamando de tipo, é isto: falta o `generate`.

### Onde o cliente mora, e por que fora do Git

O gerador escreve TypeScript legível em `src/generated/prisma`. Abra e olhe — é código comum.

E ele entra no `.gitignore`:

```
src/generated/
```

O motivo é o mesmo do `dist/`: **código gerado não se versiona**. Ele nasce do schema, é
reescrito inteiro a cada `generate`, e versioná-lo transformaria toda alteração de campo em um
Pull Request com milhares de linhas que ninguém escreveu.

Pela mesma razão, ele sai do ESLint e do Prettier: formatar arquivo que a ferramenta reescreve é
trabalho jogado fora.

> [!TIP]
> Consequência boa e não óbvia: por não ser versionado, o cliente gerado também não aparece para
> o verificador de coerência do curso. Arquivo que ninguém escreve não precisa ser conferido.

### E quem gera, em cada situação?

Se o cliente não está no Git, ele não vem no `git clone`. Alguém precisa gerá-lo — e essa
responsabilidade não pode ser "lembrar de rodar um comando":

| Situação               | Quem gera                                          |
| :--------------------- | :------------------------------------------------- |
| `npm run dev`          | O script `predev`, que o npm roda sozinho antes    |
| `npm test`             | O script `pretest`, pelo mesmo mecanismo           |
| `npm run build`        | A primeira parte do próprio `build`                |
| Build da imagem Docker | O `npm run build`, dentro do estágio de compilação |

Prove que funciona apagando tudo:

```bash
# Windows (PowerShell)
Remove-Item -Recurse -Force src/generated, dist

# Linux / Mac
rm -rf src/generated dist

npm run dev
```

A API sobe. Sem isso, o clone novo receberia:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module 'src/generated/prisma/client.ts'
```

---

## 🔌 Capítulo 6: A conexão, o repository e o service

### `src/shared/database/index.ts`

Uma instância do cliente para a aplicação inteira. Cada instância abre o próprio conjunto de
conexões, e um banco aguenta um número limitado delas — criar um cliente por requisição derruba
o banco antes de derrubar a API.

É aqui que aparece o driver adapter, obrigatório na versão 7:

```ts
const adapter = new PrismaMariaDb(env.DATABASE_URL)

export const prisma = new PrismaClient({ adapter })
```

E é aqui que mora o fechamento da conexão, que o `server.ts` chama no desligamento gracioso da
Aula 11:

```ts
export async function fecharBanco(): Promise<void> {
  await prisma.$disconnect()
}
```

No `server.ts`, a ordem não é detalhe:

```ts
    await app.close()
    await fecharBanco()
```

Fechar o banco **antes** de o servidor parar de atender cortaria justamente as requisições que
ainda estavam sendo respondidas — o contrário do que aquele encerramento existe para fazer.

### O repository

Único lugar do projeto que fala com o banco. Sem nenhuma regra de negócio dentro.

```ts
  async buscarPorCpf(cpf: string): Promise<CidadaoModel | null> {
    return prisma.cidadao.findUnique({ where: { cpf } })
  }
```

Devolver `null` é resposta legítima aqui. Decidir se isso é erro cabe a quem chamou.

> [!NOTE]
> O tipo do modelo gerado se chama `CidadaoModel`, com sufixo. Em material escrito para versões
> anteriores ele aparece só como `Cidadao`, e o editor vai dizer que não existe.

### O service

As regras. E a primeira delas mostra a divisão com clareza:

```ts
    const jaCadastrado = await this.repository.buscarPorCpf(dados.cpf)

    if (jaCadastrado !== null) {
      throw new AppError('Já existe um cidadão cadastrado com este CPF.', 409)
    }
```

"Mas o banco já recusaria sozinho, por causa do `@unique`." Sim — e as duas coisas têm papéis
diferentes:

| Onde                     | O que garante                                                          |
| :----------------------- | :--------------------------------------------------------------------- |
| A regra no **service**   | Uma mensagem em português, que quem consome a API entende              |
| O `@unique` no **banco** | A garantia final, que vale mesmo se outro caminho de cadastro esquecer |

Confiar só na primeira deixa o dado à mercê de um esquecimento. Confiar só na segunda entrega ao
cliente uma mensagem escrita para desenvolvedor — o que a Aula 06 proibiu.

O status é **409**, e não 400: a requisição está correta; o que impede o cadastro é o estado
atual do sistema.

---

## 🧪 Capítulo 7: Testes contra um banco de verdade

Até aqui os testes rodavam sem depender de nada externo. Os deste módulo são diferentes: eles
**gravam e leem de verdade**.

Poderíamos usar um dublê no lugar do repository. Seria mais rápido, e não provaria nada: um
dublê responde o que mandamos ele responder, e por isso nunca reprova por coluna com nome
errado, tipo incompatível ou índice ausente — que é exatamente onde o erro mora quando há banco.

### O banco separado, e por quê

Os testes limpam a tabela antes de cada caso. Se rodassem no banco de trabalho, `npm test`
apagaria o que você estava usando para experimentar — e você descobriria isso depois,
procurando o dado que sumiu.

Por isso existe o `curso_api_test`, criado pelo mesmo script de inicialização. A troca
acontece no `vitest.config.ts`:

```ts
    env: {
      NODE_ENV: 'test',
      DATABASE_URL: process.env.DATABASE_TEST_URL ?? '',
    },
```

E a estrutura dele é preparada por um arquivo que roda uma vez, antes de tudo — o
`vitest.global-setup.ts`, que aplica as migrations com `migrate deploy`.

> [!NOTE]
> `migrate deploy` só aplica o que já está versionado: não cria migration nova nem usa banco de
> sombra. É o comando que roda em produção, e é o assunto inteiro da Aula 14.

### A prova que importa

Grave uma linha no banco **de trabalho**, rode a suíte e confira que ela continua lá:

```bash
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api \
  -e "INSERT INTO cidadaos (id,nome,cpf,atualizadoEm) VALUES ('marcador','Nao Apague','99999999999',NOW(3));"

npm test

docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api -e "SELECT nome FROM cidadaos;"
```

```
=== antes ===        nome: Nao Apague
=== npm test ===     Test Files 7 passed | Tests 87 passed
=== depois ===       nome: Nao Apague
=== banco de teste === registros: 0
```

O banco de trabalho intacto, o de teste zerado. Isso é o isolamento funcionando — e é uma coisa
que se **prova**, não que se presume.

---

## ⚠️ Capítulo 8: A segunda migration, e o erro que todo mundo comete

Com a tabela já contendo dados, acrescente um campo obrigatório ao schema:

```prisma
  telefone     String
```

```bash
npx prisma migrate dev --name adiciona-telefone
```

```
⚠️ We found changes that cannot be executed:

  • Step 0 Added the required column `telefone` to the `cidadaos` table without a
    default value. There are 1 rows in this table, it is not possible to execute
    this step.
```

**Pare e pense no que ele está dizendo.** A coluna é obrigatória; existe uma linha na tabela; e
não há valor para essa linha. Não existe resposta certa que a ferramenta possa inventar — ela
teria de escolher um valor por você, e escolher errado em silêncio é o pior desfecho possível.

As saídas legítimas são três: tornar o campo opcional, dar um valor padrão, ou fazer a migração
em etapas (adicionar opcional → preencher → tornar obrigatório). A terceira é o padrão
**expande/contrai**, e é a Aula 14.

Aqui, a resposta é a primeira — telefone é mesmo opcional:

```prisma
  telefone     String?
```

```bash
npx prisma migrate dev --name adiciona-telefone
```

```sql
-- AlterTable
ALTER TABLE `cidadaos` ADD COLUMN `telefone` VARCHAR(191) NULL;
```

E a linha que já existia continua lá, com `NULL` no campo novo:

```
nome            telefone
Nao Apague      NULL
```

> [!CAUTION]
> Na sua máquina, se der errado, você apaga o banco e recomeça. Em um banco com dados reais essa
> opção não existe — e é por isso que migration em produção é uma aula inteira, e não um
> parágrafo desta.

---

## 🌱 Capítulo 9: Semeando o banco

Depois de um `down -v`, o banco nasce vazio. Um arquivo de **seed** resolve isso.

A palavra que importa é **idempotente**: rodar dez vezes tem o mesmo efeito de rodar uma. Isso
se consegue com `upsert` — atualiza se existe, cria se não existe:

```ts
    await prisma.cidadao.upsert({
      where: { cpf: cidadao.cpf },
      update: { nome: cidadao.nome, email: cidadao.email },
      create: cidadao,
    })
```

Um seed que só faz `create` funciona uma vez e quebra na segunda com erro de CPF duplicado — e
aí alguém "resolve" apagando o banco, que é o hábito errado de nascer.

Ligue o comando no `prisma.config.ts`:

```ts
    seed: 'tsx --env-file=.env prisma/seed.ts',
```

E rode duas vezes:

```bash
npx prisma db seed
npx prisma db seed
```

```
✅ Seed concluído. A tabela tem 3 cidadão(s).
✅ Seed concluído. A tabela tem 3 cidadão(s).
```

Três nas duas vezes. Os CPFs são inventados de propósito: material de estudo não usa documento
de pessoa real.

---

## 🐳 Capítulo 10: A imagem Docker, e um susto de 400 MB

O `Dockerfile` da Aula 10 precisa de duas mudanças. A primeira é óbvia — o schema tem de entrar
no estágio de construção, senão o `prisma generate` não tem o que ler:

```dockerfile
COPY prisma.config.ts ./
COPY prisma ./prisma
```

A segunda só apareceu **porque a imagem foi medida**.

### O que aconteceu

Construída do jeito natural, a imagem saiu assim:

| Antes desta aula (Aula 10) | Com o Prisma, sem cuidado |
| :------------------------- | :------------------------ |
| **380 MB**                 | **898 MB**                |

Mais que o dobro. Olhando por dentro:

```bash
docker compose exec api sh -c "du -sh node_modules/@prisma/* | sort -h | tail -4"
```

```
19M     node_modules/@prisma/dev
28M     node_modules/@prisma/engines
43M     node_modules/@prisma/studio-core
75M     node_modules/@prisma/client
```

O Prisma **Studio** e as ferramentas de desenvolvimento estavam dentro da imagem de produção. E
o CLI também:

```bash
docker compose exec api sh -c "ls node_modules | grep -c '^prisma$'"   # 1
```

### Por que, se o CLI é dependência de desenvolvimento?

Porque o `@prisma/client` declara o CLI como **peer dependency opcional**, e o npm resolve peers
sozinho — inclusive com `--omit=dev`. O `npm ls` de dentro do container mostra a corrente
inteira:

```
@prisma/client@7.9.1
  `-- prisma@7.9.1
    `-- @prisma/studio-core@0.24.x
```

A correção é dizer ao npm, **naquela linha específica**, para não resolver peers:

```dockerfile
RUN npm ci --omit=dev --legacy-peer-deps && npm cache clean --force
```

| Instalação da imagem            | Tamanho    | CLI dentro? |
| :------------------------------ | :--------- | :---------- |
| `--omit=dev`                    | 898 MB     | sim         |
| `--omit=dev --legacy-peer-deps` | **494 MB** | não         |

Tudo o que a API executa — `@prisma/client` e `@prisma/adapter-mariadb` — está declarado como
dependência direta, então nada essencial se perde.

> [!CAUTION]
> **Isto vale só para aquela linha do `Dockerfile`.** Na sua máquina, `npm install` continua
> tendo de concluir **sem** `--legacy-peer-deps`: lá a opção esconderia incompatibilidade de
> verdade. A diferença é o objetivo: na imagem, queremos deliberadamente **menos** do que o npm
> instalaria; na sua máquina, queremos exatamente o que o projeto declara.

### E prove que funciona

Depois de subir, consulte o banco a partir de dentro do container:

```bash
docker compose exec api node -e "
import('@prisma/adapter-mariadb').then(async (a) => {
  const m = await import('./dist/generated/prisma/client.js')
  const prisma = new m.PrismaClient({ adapter: new a.PrismaMariaDb(process.env.DATABASE_URL) })
  console.log('cidadaos:', await prisma.cidadao.count())
  await prisma.\$disconnect()
})"
```

```
cidadaos: 3
```

A API, dentro do container, falando com o banco pelo nome do serviço — que é a rede da Aula 12
funcionando por baixo.

---

## 🛑 Capítulo 11: o desligamento que voltou a falhar

A Aula 11 ensinou a API a morrer direito, e nós conferimos isso lá. Ao repetir a mesma
conferência **depois** de tudo desta aula, o resultado foi outro:

```
rodada 1 | 3577ms | exit=137
rodada 2 |  545ms | exit=0
rodada 3 | 3838ms | exit=137
rodada 4 | 3591ms | exit=137
rodada 5 | 3635ms | exit=137
```

Cinco de sete desligamentos terminaram em `SIGKILL` — e **sem uma linha de log** dizendo o que
aconteceu. O comando era o mais banal possível:

```bash
docker compose up -d api
docker compose stop api
```

### O que estava acontecendo

Lembre da Aula 11: o PID 1 do Linux **não tem ação padrão para sinal**. Sem ouvinte registrado,
o `SIGTERM` é ignorado — e o Docker manda o sinal **uma vez só**, espera o prazo dele e então
manda `SIGKILL`.

O ouvinte existia. Ele era registrado dentro do `start()`, logo antes do `listen`. O detalhe é
**quando** aquilo roda: antes de o `start()` começar, o processo precisa carregar o Fastify, os
plugins, o Zod, o logger e — agora — o cliente do Prisma. Isso leva cerca de meio segundo.

Meio segundo em que o processo já existe, já é PID 1, e ainda é **surdo**.

Um `stop` disparado logo depois do `up` cai justamente aí. E a chance disso é grande em dois
lugares que importam: no reinício automático de um deploy, e em qualquer script que suba e
derrube em sequência.

### A correção: registrar antes de carregar

Os ouvintes saíram do `start()` e foram para um módulo próprio, `src/shared/shutdown/index.ts`,
que é o **primeiro import** do `server.ts`:

```ts
import { registrarEncerrador } from './shared/shutdown/index.ts'

import type { FastifyInstance } from 'fastify'
import { buildApp } from './app.ts'
```

Em ESM os módulos são avaliados na ordem em que aparecem. Como esse arquivo não importa nada
pesado, os ouvintes passam a existir em poucos milissegundos — antes de o Fastify sequer estar
na memória.

E o ouvinte precisa saber lidar com as duas fases:

| Quando o sinal chega           | O que ele faz                                                |
| :----------------------------- | :----------------------------------------------------------- |
| A aplicação ainda está subindo | Sai com código **0** na hora: não há requisição para esperar |
| A aplicação já está atendendo  | Faz o encerramento gracioso da Aula 11                       |

### O resultado, medido do mesmo jeito

```
rodada 1 | 555ms | exit=0
rodada 2 | 550ms | exit=0
rodada 3 | 580ms | exit=0
rodada 4 | 630ms | exit=0
rodada 5 | 597ms | exit=0
rodada 6 | 618ms | exit=0
rodada 7 | 584ms | exit=0
```

Sete de sete, todas abaixo de 0,7 s.

> [!IMPORTANT]
> Repare no que denunciou o problema: **não foi um teste automatizado**. Foi repetir uma
> verificação antiga depois de mexer em outra coisa. É por isso que a seção "Como saber que deu
> certo" de cada aula não some quando a aula termina — ela vira a conferência das próximas.

---

## 🖥️ Capítulo 12: Vendo os dados sem instalar nada

```bash
npm run db:studio
```

O Prisma Studio abre no navegador com a tabela `cidadaos`: dá para ver, filtrar, editar e apagar
registro. É a ferramenta do próprio ORM, e não custa nenhum serviço a mais no Compose.

> [!WARNING]
> Ele escreve no banco de verdade, sem perguntar duas vezes. É excelente para inspecionar e
> perigoso para "dar uma arrumadinha" — mudança feita ali não tem migration, não tem revisão e
> não tem histórico.

---

## 📄 Capítulo 13: os arquivos, por inteiro

### `prisma/schema.prisma`

```prisma
// Modelo de dados do projeto.
//
// Este arquivo é a fonte da verdade da estrutura do banco: cada alteração aqui
// vira uma migration, e cada migration vira um arquivo SQL versionado no Git.

generator client {
  // O gerador do Prisma 7 escreve TypeScript direto na pasta abaixo, em vez de
  // esconder o cliente dentro de `node_modules`. A pasta fica fora do Git: é
  // código gerado, como o `dist/`.
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "mysql"
}

model Cidadao {
  id           String   @id @default(uuid())
  nome         String
  cpf          String   @unique
  email        String?
  telefone     String?
  criadoEm     DateTime @default(now())
  atualizadoEm DateTime @updatedAt

  @@map("cidadaos")
}
```

### `prisma.config.ts`

```ts
import { existsSync } from 'node:fs'

import { defineConfig } from 'prisma/config'

// O CLI do Prisma roda como um processo separado da API e, a partir da versão 7,
// não lê o `.env` sozinho. O Node 24 resolve isso sem dependência nenhuma: o
// `process.loadEnvFile()` carrega o arquivo para dentro do `process.env`, que é
// de onde o `env()` abaixo lê. É a mesma ideia do `--env-file` que o `npm run dev`
// já usa desde a Aula 04.
//
// O `if` não é zelo exagerado: **dentro do container não existe `.env`**, porque
// o `.dockerignore` o mantém fora da imagem desde a Aula 10. Lá a configuração
// chega por variável de ambiente, que é o contrato do Docker. Sem esta condição,
// o `prisma generate` do build morre com `ENOENT: no such file or directory,
// open '.env'` — e é o mesmo motivo do `--env-file-if-exists` do `npm start`.
if (existsSync('.env')) {
  process.loadEnvFile()
}

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
    // Comando que o `prisma db seed` executa. O `--env-file` é necessário
    // porque o seed roda em processo próprio, e o Node só lê o `.env` quando
    // mandam — o mesmo motivo do `npm run dev`.
    seed: 'tsx --env-file=.env prisma/seed.ts',
  },
  datasource: {
    // Lido direto do ambiente, e não pelo `env()` do Prisma, por um motivo
    // medido: o `env()` **exige** a variável no instante em que este arquivo é
    // carregado — inclusive no `prisma generate`, que roda dentro do build da
    // imagem e não tem, nem deve ter, endereço de banco nenhum. Gerar o cliente
    // é traduzir o schema em TypeScript; não se conecta a lugar algum.
    //
    // Quem realmente precisa da variável são os comandos que falam com o banco
    // (`migrate`, `db seed`, `studio`) — e esses reclamam sozinhos, na hora,
    // se ela estiver vazia.
    url: process.env.DATABASE_URL ?? '',

    // Banco descartável que o `migrate dev` usa para conferir se as migrations,
    // aplicadas em ordem, produzem exatamente o schema declarado. Vem pronto do
    // Compose porque o usuário da aplicação não pode criar banco — e isso é
    // decisão de segurança, não descuido.
    shadowDatabaseUrl: process.env.SHADOW_DATABASE_URL ?? '',
  },
})
```

### `docker/mysql/init/01-banco-de-teste.sql`

```sql
-- Cria o banco que os testes automatizados usam.
--
-- Ele existe para que `npm test` nunca apague o banco de trabalho: os testes
-- limpam as tabelas antes de cada execução, e fazer isso no banco em que você
-- estava experimentando é perder o trabalho sem aviso.
--
-- Este arquivo roda UMA vez, quando o MySQL inicializa a pasta de dados pela
-- primeira vez. Alterá-lo depois não tem efeito nenhum enquanto o volume
-- existir — é preciso `docker compose down -v` para o banco nascer de novo.
CREATE DATABASE IF NOT EXISTS curso_api_test;

-- O usuário da aplicação, criado pelo próprio MySQL a partir das variáveis do
-- Compose, recebe acesso também ao banco de teste. Sem esta linha ele enxergaria
-- apenas o banco de trabalho, e os testes falhariam com "access denied".
GRANT ALL PRIVILEGES ON curso_api_test.* TO 'curso_api'@'%';
FLUSH PRIVILEGES;

-- O `prisma migrate dev` precisa de um banco descartável, o "shadow database",
-- para conferir se as migrations aplicadas em ordem produzem exatamente o schema
-- declarado. Por padrão ele tenta CRIAR esse banco sozinho — e o usuário da
-- aplicação não pode criar banco, o que é uma decisão de segurança, não um
-- descuido. A saída é entregar um banco pronto e apontá-lo no `prisma.config.ts`.
--
-- Ele existe só na máquina de quem desenvolve: em produção, `migrate deploy` não
-- usa shadow database nenhum.
CREATE DATABASE IF NOT EXISTS curso_api_shadow;
GRANT ALL PRIVILEGES ON curso_api_shadow.* TO 'curso_api'@'%';
FLUSH PRIVILEGES;
```

### `docker-compose.yml`

```yaml
# Ambiente completo do projeto: a API e o banco de dados de que ela vai precisar.
#
# Antes deste arquivo, subir o ambiente era conhecimento que morava fora do
# repositório — na memória de alguém, ou no histórico do terminal de alguém. Aqui
# ele vira código: lido, revisado e versionado como qualquer outro arquivo.
#
# Sobe tudo com `npm run compose:up`; derruba com `npm run compose:down`.
services:
  api:
    # Constrói a partir do `Dockerfile` que já existe, sem nenhuma linha nova
    # nele: é a mesma imagem de produção, subindo do mesmo jeito.
    build: .
    ports:
      - '${PORT:-3333}:3333'
    environment:
      NODE_ENV: production
      HOST: 0.0.0.0
      PORT: 3333
      # Dentro da rede do projeto o banco atende no nome do serviço e na porta
      # 3306 de sempre — a `MYSQL_PORT` do `.env` só muda a porta publicada na
      # sua máquina, que é assunto de quem está do lado de fora.
      DATABASE_URL: mysql://${MYSQL_USER}:${MYSQL_PASSWORD}@mysql:3306/${MYSQL_DATABASE}
    depends_on:
      mysql:
        # Esta linha é a lição desta aula. Sem ela, o `depends_on` espera apenas
        # o container do MySQL EXISTIR — e ele existe segundos antes de aceitar
        # conexão. Medido nesta máquina: o container ficou "running" em 1,0s, e o
        # banco só respondeu 19,9s depois. Uma API que subisse junto tentaria
        # conectar nesse vão e tomaria recusa, com o banco perfeitamente saudável
        # ao lado.
        condition: service_healthy

  mysql:
    # Linha LTS: recebe correção por anos e não muda de comportamento no meio do
    # caminho. A tag `8.4` entrega hoje o MySQL 8.4.11.
    image: mysql:8.4
    ports:
      # Publicar a porta é o que permite o `npm run dev`, que roda fora do
      # container, alcançar este banco. Se a sua máquina já tiver um MySQL na
      # 3306, troque `MYSQL_PORT` no `.env` — nada aqui dentro precisa saber.
      - '${MYSQL_PORT:-3306}:3306'
    environment:
      # Lidas do `.env` pelo próprio Compose. Senha nunca fica escrita no YAML,
      # que é versionado; o `.env` não é.
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - dados-mysql:/var/lib/mysql
      # Scripts rodados na primeira inicializacao do banco. E daqui que sai o
      # banco de teste, separado do de trabalho.
      - ./docker/mysql/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      # A pergunta é feita pela REDE (`-h 127.0.0.1`), e não pelo soquete local,
      # porque é pela rede que a API vai falar com o banco. Verificar por um
      # caminho que o cliente real não usa é verificar outra coisa.
      test: ['CMD', 'mysqladmin', 'ping', '-h', '127.0.0.1', '--silent']
      interval: 5s
      timeout: 3s
      # Enquanto o `start_period` corre, uma verificação que falha não conta como
      # falha: o MySQL inicializa a pasta de dados na primeira subida, e isso
      # levou cerca de 20s aqui. Sem essa folga, o container seria declarado
      # doente por estar fazendo exatamente o que devia.
      start_period: 30s
      retries: 10

volumes:
  # Volume nomeado: o banco sobrevive ao `docker compose down`. Quem apaga os
  # dados é o `down -v`, e é por isso que o `-v` não entra em nenhum atalho do
  # `package.json` — comando que destrói dado se digita por inteiro, de propósito.
  dados-mysql:
```

### `src/shared/database/index.ts`

```ts
/**
 * Conexão com o banco de dados
 *
 * Este arquivo cria **uma única** instância do cliente do Prisma para a
 * aplicação inteira. Cada instância abre o próprio conjunto de conexões com o
 * MySQL, e um banco aguenta um número limitado delas: criar um cliente por
 * requisição derruba o banco antes de derrubar a API.
 *
 * A partir do Prisma 7 o cliente **exige** um driver adapter — o pacote que sabe
 * conversar com aquele banco específico. Para MySQL, é o `@prisma/adapter-mariadb`
 * (o protocolo é o mesmo dos dois bancos). Chamar `new PrismaClient()` sem
 * adapter é erro, e é assim que se reconhece exemplo escrito para a versão antiga.
 */

import { PrismaMariaDb } from '@prisma/adapter-mariadb'

import { PrismaClient } from '../../generated/prisma/client.ts'
import { env } from '../env/index.ts'

/**
 * O adapter recebe o endereço do banco já validado pelo schema de ambiente.
 *
 * Repare que a URL **não** está no `schema.prisma`: no Prisma 7 o schema descreve
 * o formato dos dados, e o endereço de conexão vive na configuração. São duas
 * coisas diferentes — a estrutura é igual em todas as máquinas, o endereço não.
 */
const adapter = new PrismaMariaDb(env.DATABASE_URL)

/**
 * Cliente do Prisma usado por toda a aplicação.
 *
 * É por ele que os repositories fazem consulta. Ninguém mais deveria importá-lo:
 * quem precisa de dado pede ao repository, que é a única camada que conhece o
 * banco.
 */
export const prisma = new PrismaClient({ adapter })

/**
 * Fecha as conexões abertas com o banco.
 *
 * Chamado no desligamento gracioso da Aula 11. Sem isto, o processo pode sair com
 * conexões ainda abertas do lado do MySQL, que só as descarta quando o tempo
 * limite dele estoura — e, num deploy que reinicia a API várias vezes seguidas, é
 * assim que se esgota o número de conexões de um banco saudável.
 */
export async function fecharBanco(): Promise<void> {
  await prisma.$disconnect()
}
```

### `src/modules/cidadao/cidadao.repository.ts`

```ts
/**
 * CidadaoRepository
 *
 * Única camada do projeto que fala com o banco de dados. Ela traduz "quero o
 * cidadão de CPF tal" em consulta, e devolve o resultado — **sem nenhuma regra de
 * negócio**.
 *
 * A separação parece exagero com uma tabela só, e paga em duas situações que
 * chegam depois: quando o teste precisa rodar sem banco, e quando a consulta muda
 * (um índice novo, um `join`) sem que nenhuma regra tenha mudado junto.
 */

import { prisma } from '../../shared/database/index.ts'
// O tipo do modelo gerado pelo Prisma 7 leva o sufixo `Model`. Em material
// escrito para versoes anteriores ele aparece so como `Cidadao`.
import type { CidadaoModel } from '../../generated/prisma/models.ts'

/** Dados necessários para cadastrar um cidadão. */
export interface DadosDoNovoCidadao {
  nome: string
  cpf: string
  email?: string | undefined
}

export class CidadaoRepository {
  /**
   * Grava um cidadão novo.
   *
   * @param dados Nome, CPF e, opcionalmente, e-mail.
   * @returns     O registro gravado, já com `id` e datas preenchidos pelo banco.
   */
  async criar(dados: DadosDoNovoCidadao): Promise<CidadaoModel> {
    return prisma.cidadao.create({ data: dados })
  }

  /**
   * Busca um cidadão pelo identificador interno.
   *
   * @param id Identificador gerado no cadastro.
   * @returns  O cidadão, ou `null` quando não existe. Devolver `null` é resposta
   *           legítima do repositório: decidir se isso é erro cabe a quem chamou.
   */
  async buscarPorId(id: string): Promise<CidadaoModel | null> {
    return prisma.cidadao.findUnique({ where: { id } })
  }

  /**
   * Busca um cidadão pelo CPF.
   *
   * É esta consulta que sustenta a regra de não cadastrar o mesmo CPF duas vezes.
   * Ela é barata porque o CPF é `@unique` no schema — o que criou um índice no
   * banco, e não apenas uma promessa no código.
   *
   * @param cpf CPF, apenas dígitos.
   * @returns   O cidadão, ou `null`.
   */
  async buscarPorCpf(cpf: string): Promise<CidadaoModel | null> {
    return prisma.cidadao.findUnique({ where: { cpf } })
  }

  /**
   * Lista os cidadãos cadastrados, do mais recente para o mais antigo.
   *
   * @returns Todos os registros. Paginação entra quando existir volume que a
   *          justifique — hoje seria configuração para uma dor que não existe.
   */
  async listar(): Promise<CidadaoModel[]> {
    return prisma.cidadao.findMany({ orderBy: { criadoEm: 'desc' } })
  }
}
```

### `src/modules/cidadao/cidadao.service.ts`

```ts
/**
 * CidadaoService
 *
 * Concentra as regras de negócio de cidadão. Ele decide **o que pode acontecer**;
 * o repository sabe apenas **como falar com o banco**.
 *
 * A divisão fica visível na regra de CPF duplicado: o banco já recusaria o
 * segundo cadastro sozinho, porque o campo é `@unique`. Mesmo assim a checagem
 * existe aqui — e as duas coisas têm papéis diferentes:
 *
 *   • a regra no service devolve uma mensagem em português, que quem consome a
 *     API entende;
 *   • a restrição no banco é a garantia final, que continua valendo mesmo se um
 *     dia alguém escrever outro caminho de cadastro e esquecer a regra.
 *
 * Confiar só na primeira deixa o dado à mercê de um esquecimento. Confiar só na
 * segunda entrega ao cliente um erro escrito para desenvolvedor.
 */

import { AppError } from '../../shared/errors/app-error.ts'
import type { CidadaoModel } from '../../generated/prisma/models.ts'
import type { CidadaoRepository, DadosDoNovoCidadao } from './cidadao.repository.ts'

export class CidadaoService {
  /**
   * @param repository Camada de acesso ao banco. Recebida pelo construtor, e não
   *                   importada aqui dentro: é o que permite trocá-la por outra
   *                   implementação sem tocar nesta classe.
   */
  constructor(private readonly repository: CidadaoRepository) {}

  /**
   * Cadastra um cidadão, recusando CPF já existente.
   *
   * @param dados Nome, CPF e, opcionalmente, e-mail.
   * @throws {AppError} 409, quando o CPF já está cadastrado.
   * @returns O cidadão recém-criado.
   */
  async cadastrar(dados: DadosDoNovoCidadao): Promise<CidadaoModel> {
    const jaCadastrado = await this.repository.buscarPorCpf(dados.cpf)

    if (jaCadastrado !== null) {
      // 409 (conflito) e não 400: a requisição está correta em forma e conteúdo.
      // O que impede o cadastro é o estado atual do sistema, não um erro de quem
      // chamou — e essa diferença muda o que a outra ponta deve fazer a respeito.
      throw new AppError('Já existe um cidadão cadastrado com este CPF.', 409)
    }

    return this.repository.criar(dados)
  }

  /**
   * Busca um cidadão pelo identificador.
   *
   * @param id Identificador gerado no cadastro.
   * @throws {AppError} 404, quando não existe cidadão com aquele id.
   * @returns O cidadão encontrado.
   */
  async buscarPorId(id: string): Promise<CidadaoModel> {
    const cidadao = await this.repository.buscarPorId(id)

    if (cidadao === null) {
      throw new AppError('Cidadão não encontrado.', 404)
    }

    return cidadao
  }

  /**
   * Lista os cidadãos cadastrados.
   *
   * @returns Lista, do cadastro mais recente para o mais antigo.
   */
  async listar(): Promise<CidadaoModel[]> {
    return this.repository.listar()
  }
}
```

### `src/modules/cidadao/cidadao.spec.ts`

```ts
/**
 * Testes do módulo de cidadão
 *
 * Estes são os primeiros testes do projeto que tocam um **banco de verdade**, e
 * não uma imitação. A diferença importa: um dublê responde o que mandamos ele
 * responder, e por isso nunca reprova por causa de coluna com nome errado, tipo
 * incompatível ou índice ausente — que é exatamente onde o erro mora quando se
 * trabalha com banco.
 *
 * O banco usado é o `curso_api_test`, criado pelo Compose ao lado do de
 * trabalho. O `vitest.config.ts` troca a `DATABASE_URL` para apontar para ele.
 */

import { afterAll, beforeEach, describe, expect, it } from 'vitest'

import { prisma } from '../../shared/database/index.ts'
import { AppError } from '../../shared/errors/app-error.ts'
import { CidadaoRepository } from './cidadao.repository.ts'
import { CidadaoService } from './cidadao.service.ts'

const repository = new CidadaoRepository()
const service = new CidadaoService(repository)

// CPFs inventados, com dígito verificador que não fecha. Material didático não
// usa documento de pessoa real — nem mesmo um que pareça válido.
const CPF_A = '00000000191'
const CPF_B = '00000000272'

beforeEach(async () => {
  // Cada teste começa com a tabela vazia. Sem isto, um teste que conta registros
  // passaria sozinho e reprovaria junto com os outros — o tipo de intermitência
  // que faz o time desconfiar da suíte inteira.
  await prisma.cidadao.deleteMany()
})

afterAll(async () => {
  await prisma.$disconnect()
})

describe('CidadaoRepository — o que o banco de fato guarda', () => {
  it('grava e devolve o cidadão com id e datas preenchidos', async () => {
    const cidadao = await repository.criar({ nome: 'Maria Souza', cpf: CPF_A })

    expect(cidadao.id).toMatch(/^[0-9a-f-]{36}$/)
    expect(cidadao.nome).toBe('Maria Souza')
    expect(cidadao.criadoEm).toBeInstanceOf(Date)
    expect(cidadao.atualizadoEm).toBeInstanceOf(Date)
  })

  it('guarda o e-mail como nulo quando ele não é informado', async () => {
    const cidadao = await repository.criar({ nome: 'João Lima', cpf: CPF_A })

    // O campo é opcional no schema (`String?`), e o que chega do banco é `null`,
    // não `undefined`. São coisas diferentes em TypeScript, e confundi-las
    // produz comparação que nunca dá certo.
    expect(cidadao.email).toBeNull()
  })

  it('encontra pelo CPF o que acabou de gravar', async () => {
    await repository.criar({ nome: 'Ana Dias', cpf: CPF_A, email: 'ana@exemplo.gov.br' })

    const encontrado = await repository.buscarPorCpf(CPF_A)

    expect(encontrado?.nome).toBe('Ana Dias')
    expect(encontrado?.email).toBe('ana@exemplo.gov.br')
  })

  it('devolve nulo quando o CPF não está cadastrado', async () => {
    expect(await repository.buscarPorCpf(CPF_B)).toBeNull()
  })

  it('lista do cadastro mais recente para o mais antigo', async () => {
    await repository.criar({ nome: 'Primeira', cpf: CPF_A })
    await repository.criar({ nome: 'Segunda', cpf: CPF_B })

    const lista = await repository.listar()

    expect(lista.map((cidadao) => cidadao.nome)).toEqual(['Segunda', 'Primeira'])
  })

  it('recusa CPF repetido, porque o banco tem índice único', async () => {
    await repository.criar({ nome: 'Original', cpf: CPF_A })

    // Este é o teste que só um banco de verdade consegue provar. A garantia não
    // está no código: está na coluna, criada pela migration a partir do `@unique`.
    await expect(repository.criar({ nome: 'Cópia', cpf: CPF_A })).rejects.toThrow()
  })
})

describe('CidadaoService — as regras que o banco não conhece', () => {
  it('cadastra quando o CPF é novo', async () => {
    const cidadao = await service.cadastrar({ nome: 'Carlos Reis', cpf: CPF_A })

    expect(await repository.buscarPorId(cidadao.id)).not.toBeNull()
  })

  it('recusa CPF já cadastrado com mensagem em português e status 409', async () => {
    await service.cadastrar({ nome: 'Carlos Reis', cpf: CPF_A })

    // O banco também recusaria — mas com uma mensagem escrita para
    // desenvolvedor, que a Aula 06 proibiu de sair para quem chama a API.
    await expect(service.cadastrar({ nome: 'Outro', cpf: CPF_A })).rejects.toThrow(
      new AppError('Já existe um cidadão cadastrado com este CPF.', 409),
    )
  })

  it('não grava nada quando o cadastro é recusado', async () => {
    await service.cadastrar({ nome: 'Carlos Reis', cpf: CPF_A })

    await expect(service.cadastrar({ nome: 'Outro', cpf: CPF_A })).rejects.toThrow(AppError)

    expect(await repository.listar()).toHaveLength(1)
  })

  it('responde 404 ao buscar id inexistente', async () => {
    await expect(service.buscarPorId('nao-existe')).rejects.toThrow(
      new AppError('Cidadão não encontrado.', 404),
    )
  })
})
```

### `prisma/seed.ts`

```ts
/**
 * Semeadura do banco (seed)
 *
 * Preenche o banco com alguns registros de partida, para que ninguém precise
 * cadastrar tudo na mão depois de um `docker compose down -v`.
 *
 * A palavra que importa aqui é **idempotente**: rodar este arquivo dez vezes tem
 * o mesmo efeito de rodar uma. Isso é conseguido com `upsert`, que atualiza
 * quando o registro já existe e cria quando não existe. Um seed que só faz
 * `create` funciona uma vez e quebra na segunda, com erro de CPF duplicado — e
 * aí alguém "resolve" apagando o banco, que é o hábito errado de nascer.
 *
 * Os CPFs são inventados, e de propósito: material de estudo não usa documento de
 * pessoa real.
 */

import { prisma } from '../src/shared/database/index.ts'

const CIDADAOS = [
  { nome: 'Maria Aparecida Souza', cpf: '00000000191', email: 'maria@exemplo.gov.br' },
  { nome: 'João Batista Lima', cpf: '00000000272', email: null },
  { nome: 'Ana Clara Dias', cpf: '00000000353', email: 'ana@exemplo.gov.br' },
]

async function semear(): Promise<void> {
  for (const cidadao of CIDADAOS) {
    await prisma.cidadao.upsert({
      // O CPF é a chave natural do cadastro: é por ele que se sabe se aquela
      // pessoa já está no banco. Usar o `id` aqui não serviria, porque ele é
      // gerado no momento da criação e ninguém o conhece de antemão.
      where: { cpf: cidadao.cpf },
      update: { nome: cidadao.nome, email: cidadao.email },
      create: cidadao,
    })
  }

  const total = await prisma.cidadao.count()

  process.stdout.write(`\n✅ Seed concluído. A tabela tem ${String(total)} cidadão(s).\n\n`)
}

semear()
  .catch((erro: unknown) => {
    process.stderr.write(`\n❌ O seed falhou.\n${String(erro)}\n\n`)
    process.exit(1)
  })
  .finally(() => {
    // Sem fechar a conexão, o processo do seed fica vivo esperando um banco que
    // não vai mandar mais nada — e trava a esteira que o chamou.
    void prisma.$disconnect()
  })
```

### `src/shared/shutdown/index.ts`

```ts
/**
 * Tratamento de sinais de desligamento
 *
 * Este módulo existe por causa de uma janela medida, e não por precaução.
 *
 * Dentro de um container o Node roda como **PID 1**, e no Linux o PID 1 não tem
 * ação padrão para sinal: sem um ouvinte registrado, o `SIGTERM` é simplesmente
 * ignorado. O Docker manda o sinal **uma vez**, espera o prazo dele e então
 * manda `SIGKILL`, que ninguém trata.
 *
 * O problema é que registrar o ouvinte dentro do `start()` deixa uma janela
 * aberta: entre o processo nascer e o `start()` chegar a rodar, o programa
 * carrega o Fastify, os plugins e o cliente do Prisma. Um `docker compose stop`
 * disparado logo após o `up` cai bem nessa janela — e o container morre com
 * código **137**, sem uma linha de log dizendo por quê.
 *
 * Medido nesta máquina, com o ouvinte registrado só no `start()`:
 *
 *   7 desligamentos seguidos → 5 terminaram em SIGKILL (exit 137, ~3,6s)
 *
 * A correção é registrar o ouvinte **antes de carregar qualquer outra coisa**.
 * Por isso este arquivo é o primeiro `import` do `server.ts` e não importa nada
 * pesado: em ESM, os módulos são avaliados na ordem em que aparecem, então o que
 * está aqui roda antes de o Fastify sequer existir na memória.
 */

/** O que fazer quando o sinal chegar com a aplicação já montada. */
type Encerrador = (sinal: string) => Promise<void>

/**
 * Sinais que significam "termine o que está fazendo".
 *
 *   • SIGTERM — o que o `docker stop` e todo orquestrador enviam no deploy.
 *   • SIGINT  — o `Ctrl+C` de quem está desenvolvendo.
 *
 * O `SIGKILL` não entra na lista porque não pode entrar: ele não chega ao
 * processo. É o sistema operacional derrubando na marra.
 */
const SINAIS_DE_DESLIGAMENTO = ['SIGTERM', 'SIGINT'] as const

let encerrador: Encerrador | null = null

/**
 * Marca que um desligamento já começou.
 *
 * Sem isso, apertar `Ctrl+C` duas vezes com pressa dispararia dois encerramentos
 * ao mesmo tempo, e o segundo encontraria o servidor no meio do fechamento do
 * primeiro.
 */
let desligando = false

for (const sinal of SINAIS_DE_DESLIGAMENTO) {
  process.on(sinal, () => {
    if (desligando) return

    desligando = true

    // Sinal recebido antes de a aplicação terminar de subir. Não há requisição
    // em andamento para esperar — e sair agora, com código 0, é infinitamente
    // melhor do que ficar surdo até levar um SIGKILL.
    if (encerrador === null) {
      process.stdout.write(
        `${JSON.stringify({
          nivel: 'info',
          sinal,
          msg: 'Sinal recebido durante a partida. Saindo antes de começar a atender.',
        })}\n`,
      )

      process.exit(0)
    }

    // O ouvinte de sinal não pode ser `async`: o Node não espera pela promessa
    // dele. O `void` deixa explícito que a promessa segue por conta própria, e
    // que quem encerra trata os próprios erros.
    void encerrador(sinal)
  })
}

/**
 * Entrega o encerramento de verdade, assim que a aplicação estiver montada.
 *
 * @param fn Função que encerra o servidor com calma, recebendo o nome do sinal.
 */
export function registrarEncerrador(fn: Encerrador): void {
  encerrador = fn
}
```

### `src/server.ts`

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

// Este import vem primeiro de propósito, e a ordem é a correção de um defeito
// medido: ele registra os ouvintes de sinal antes de o Fastify, os plugins e o
// cliente do Prisma serem carregados. O porquê está no próprio arquivo.
import { registrarEncerrador } from './shared/shutdown/index.ts'

import type { FastifyInstance } from 'fastify'
import { buildApp } from './app.ts'
import { fecharBanco } from './shared/database/index.ts'
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

    // Só depois de o servidor parar de atender é que a conexão com o banco pode
    // ser fechada: fechá-la antes cortaria justamente as requisições que ainda
    // estavam sendo respondidas — o oposto do que este encerramento existe para
    // fazer. A ordem aqui é a regra, não detalhe de implementação.
    await fecharBanco()

    clearTimeout(prazoFinal)
    app.log.info('Servidor encerrado. Nenhuma requisição foi cortada, e o banco foi liberado.')

    process.exit(0)
  } catch (error) {
    app.log.error(error, 'Falha ao encerrar o servidor.')

    process.exit(1)
  }
}

/**
 * Sobe o servidor HTTP e o deixa ouvindo requisições.
 */
async function start(): Promise<void> {
  const app = buildApp()

  // A partir daqui o desligamento é o gracioso, com espera pelas requisições em
  // andamento. Antes desta linha, o ouvinte registrado na partida já respondia —
  // saindo na hora, porque não havia nada em andamento para esperar.
  registrarEncerrador((sinal) => encerrar(app, sinal))

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

### `src/shared/env/env.schema.ts`

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

  /**
   * Endereço completo do banco de dados: usuário, senha, servidor, porta e banco.
   *
   * Não tem valor padrão de propósito. Uma API que guarda dado e sobe apontando
   * para um banco inventado é pior do que uma API que não sobe: ela funciona até
   * a primeira consulta, e aí falha em produção, com gente usando.
   *
   * O formato é conferido aqui porque o erro do driver, quando a URL está torta,
   * é bem menos claro do que esta mensagem.
   */
  DATABASE_URL: z
    .string({ error: 'é obrigatória e deve ser um texto' })
    .min(1, { error: 'é obrigatória — copie o valor do .env.example' })
    .regex(/^mysql:\/\/.+/, {
      error: 'deve começar com "mysql://" (ex.: mysql://usuario:senha@localhost:3306/banco)',
    }),
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
 * Variáveis sem valor padrão, que todo objeto de teste precisa carregar.
 *
 * A `DATABASE_URL` entrou obrigatória de propósito: uma API que guarda dado e
 * sobe apontando para um banco inventado falha só na primeira consulta, com
 * gente usando. O custo dessa escolha aparece aqui — todo teste que antes
 * mandava `{}` agora precisa dizer para onde o banco aponta.
 */
const OBRIGATORIAS = { DATABASE_URL: 'mysql://usuario:senha@localhost:3306/banco' }

/**
 * Devolve a mensagem de erro da variável indicada, ou `undefined` se ela passou.
 */
function mensagemDe(entrada: Record<string, unknown>, variavel: string): string | undefined {
  const resultado = envSchema.safeParse({ ...OBRIGATORIAS, ...entrada })

  if (resultado.success) return undefined

  return resultado.error.issues.find((problema) => problema.path[0] === variavel)?.message
}

describe('envSchema — valores válidos', () => {
  it('aceita uma configuração completa e correta', () => {
    const resultado = envSchema.safeParse({
      ...OBRIGATORIAS,
      NODE_ENV: 'production',
      PORT: '8080',
      HOST: '127.0.0.1',
    })

    expect(resultado.success).toBe(true)
  })

  it('converte a porta de texto para número', () => {
    const resultado = envSchema.parse({ ...OBRIGATORIAS, PORT: '8080' })

    // Toda variável de ambiente chega como texto. Depois da validação, ela
    // precisa ser um número de verdade — não o texto "8080".
    expect(resultado.PORT).toBe(8080)
    expect(typeof resultado.PORT).toBe('number')
  })

  it('usa os valores padrão quando nada é informado', () => {
    const resultado = envSchema.parse({ ...OBRIGATORIAS, ...OBRIGATORIAS })

    expect(resultado.NODE_ENV).toBe('development')
    expect(resultado.PORT).toBe(3333)
    expect(resultado.HOST).toBe('0.0.0.0')
  })
})

describe('envSchema — valores inválidos', () => {
  it('recusa uma porta que não é número', () => {
    // O caso real que motivou esta validação: "8O80", com a letra O no lugar do
    // zero. Passa despercebido na leitura e derruba a API na partida.
    expect(envSchema.safeParse({ ...OBRIGATORIAS, PORT: '8O80' }).success).toBe(false)
  })

  it('recusa uma porta fora da faixa permitida', () => {
    expect(envSchema.safeParse({ ...OBRIGATORIAS, PORT: '0' }).success).toBe(false)
    expect(envSchema.safeParse({ ...OBRIGATORIAS, PORT: '99999' }).success).toBe(false)
  })

  it('recusa uma porta com casas decimais', () => {
    expect(envSchema.safeParse({ ...OBRIGATORIAS, PORT: '80.5' }).success).toBe(false)
  })

  it('recusa um ambiente fora da lista conhecida', () => {
    // "producao", em português, é o engano mais provável — e o mais perigoso,
    // porque faria a aplicação se comportar como se estivesse em desenvolvimento
    // dentro do servidor de produção.
    expect(envSchema.safeParse({ ...OBRIGATORIAS, NODE_ENV: 'producao' }).success).toBe(false)
  })

  it('recusa um endereço vazio', () => {
    expect(envSchema.safeParse({ ...OBRIGATORIAS, HOST: '' }).success).toBe(false)
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
    const resultado = envSchema.parse({ ...OBRIGATORIAS, ...OBRIGATORIAS })

    // O padrão precisa ser o fechado. Configuração de segurança que nasce aberta
    // "por enquanto" costuma ficar aberta para sempre.
    expect(resultado.CORS_ORIGINS).toEqual([])
  })

  it('transforma a lista separada por vírgula em array', () => {
    const resultado = envSchema.parse({
      ...OBRIGATORIAS,
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
      ...OBRIGATORIAS,
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
    expect(envSchema.parse({ ...OBRIGATORIAS, ...OBRIGATORIAS }).TRUST_PROXY).toBe(false)
  })

  it('aceita ligar e desligar por texto', () => {
    expect(envSchema.parse({ ...OBRIGATORIAS, TRUST_PROXY: 'true' }).TRUST_PROXY).toBe(true)
    expect(envSchema.parse({ ...OBRIGATORIAS, TRUST_PROXY: 'false' }).TRUST_PROXY).toBe(false)
  })

  it('converte a quantidade de proxies para número', () => {
    const resultado = envSchema.parse({ ...OBRIGATORIAS, TRUST_PROXY: '1' })

    // O Fastify trata número e texto de formas diferentes: "1" seria lido como
    // um endereço de rede em que confiar, e não como "um salto".
    expect(resultado.TRUST_PROXY).toBe(1)
    expect(typeof resultado.TRUST_PROXY).toBe('number')
  })

  it('recusa quantidade zero ou negativa', () => {
    // Confiar em "zero proxies" é o mesmo que `false`, escrito de um jeito que
    // ninguém entende ao reler o arquivo de configuração seis meses depois.
    expect(envSchema.safeParse({ ...OBRIGATORIAS, TRUST_PROXY: '0' }).success).toBe(false)
    expect(envSchema.safeParse({ ...OBRIGATORIAS, TRUST_PROXY: '-1' }).success).toBe(false)
  })

  it('recusa qualquer outro texto', () => {
    expect(envSchema.safeParse({ ...OBRIGATORIAS, TRUST_PROXY: 'sim' }).success).toBe(false)
    expect(envSchema.safeParse({ ...OBRIGATORIAS, TRUST_PROXY: '10.0.0.1' }).success).toBe(false)
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
    expect(envSchema.parse({ ...OBRIGATORIAS, ...OBRIGATORIAS }).LOG_LEVEL).toBeUndefined()
  })

  it('aceita os níveis conhecidos', () => {
    expect(envSchema.parse({ ...OBRIGATORIAS, LOG_LEVEL: 'debug' }).LOG_LEVEL).toBe('debug')
    expect(envSchema.parse({ ...OBRIGATORIAS, LOG_LEVEL: 'silent' }).LOG_LEVEL).toBe('silent')
  })

  it('recusa um nível inventado', () => {
    // "verbose" existe em outras ferramentas e não existe aqui. Sem esta trava,
    // a API subiria com o nível padrão do Pino e ninguém entenderia por que o
    // log não mudou.
    expect(envSchema.safeParse({ ...OBRIGATORIAS, LOG_LEVEL: 'verbose' }).success).toBe(false)
  })
})

describe('envSchema — endereço do banco (DATABASE_URL)', () => {
  it('recusa a ausência da variável', () => {
    // Sem valor padrão de propósito: subir apontando para um banco inventado é
    // pior do que não subir, porque a falha só aparece na primeira consulta.
    expect(envSchema.safeParse({}).success).toBe(false)
  })

  it('recusa endereço que não é de MySQL', () => {
    expect(
      envSchema.safeParse({ DATABASE_URL: 'postgresql://usuario:senha@localhost:5432/banco' })
        .success,
    ).toBe(false)
  })

  it('recusa texto vazio', () => {
    expect(envSchema.safeParse({ DATABASE_URL: '' }).success).toBe(false)
  })

  it('explica o formato esperado quando o endereço está torto', () => {
    expect(mensagemDe({ DATABASE_URL: 'localhost:3306' }, 'DATABASE_URL')).toBe(
      'deve começar com "mysql://" (ex.: mysql://usuario:senha@localhost:3306/banco)',
    )
  })

  it('aceita um endereço de MySQL completo', () => {
    const resultado = envSchema.parse({
      DATABASE_URL: 'mysql://agencia:senha@localhost:3306/agencia',
    })

    expect(resultado.DATABASE_URL).toBe('mysql://agencia:senha@localhost:3306/agencia')
  })
})
```

### `vitest.config.ts`

```ts
/**
 * Configuração do Vitest
 *
 * Até aqui os testes rodavam só com os padrões do Vitest, e isso bastava. Este
 * arquivo nasce por um motivo medido, não por precaução.
 *
 * O que aconteceu: em uma execução do `npm run check`, três dos 52 testes
 * falharam com `Test timed out in 5000ms` — o padrão do Vitest. Na mesma
 * execução, a linha de resumo acusou `import 13.08s`. Não foi o teste que ficou
 * lento: foi a **primeira importação** do Fastify e dos plugins, em máquina
 * fria, que estourou sozinha o orçamento de cinco segundos de cada teste.
 *
 * Nas quatro execuções seguintes, com o cache já quente, a mesma importação
 * levou entre 2,0s e 2,7s e tudo passou. Ou seja: a suíte não estava errada, ela
 * estava **intermitente** — e intermitente justamente onde mais dói, que é a
 * máquina recém-clonada, o container e a esteira, os três lugares em que nada
 * está aquecido.
 *
 * Teste que reprova por velocidade de máquina ensina o time a reexecutar até
 * passar, que é o hábito exato que destrói a confiança em qualquer suíte.
 */

import { existsSync } from 'node:fs'

import { defineConfig } from 'vitest/config'

// O Vitest não lê o `.env` sozinho, e a partir da Aula 13 ele precisa: é daqui
// que sai o endereço do banco de teste. O `loadEnvFile` é do próprio Node 24 —
// mesma solução usada no `prisma.config.ts`, sem dependência nova.
//
// O `if` cobre o ambiente que não tem arquivo `.env` nenhum, onde as variáveis
// chegam prontas pelo ambiente. Sem ele, a leitura falharia com `ENOENT`.
if (existsSync('.env')) {
  process.loadEnvFile()
}

export default defineConfig({
  test: {
    /**
     * Prepara o banco de teste antes de o primeiro teste rodar.
     */
    globalSetup: './vitest.global-setup.ts',

    /**
     * Variáveis entregues aos testes.
     *
     * A troca é a linha mais importante deste arquivo: dentro dos testes, a
     * `DATABASE_URL` aponta para o banco **de teste**. Sem isso, `npm test`
     * apagaria as tabelas do banco em que você estava trabalhando — e você só
     * descobriria isso depois, procurando o dado que sumiu.
     */
    env: {
      NODE_ENV: 'test',
      DATABASE_URL: process.env.DATABASE_TEST_URL ?? '',
    },

    /**
     * Tempo máximo de cada teste, em milissegundos.
     *
     * O padrão do Vitest é 5.000. O valor aqui cobre com folga o pior caso já
     * observado neste projeto (importação de 13,08s), e não custa nada quando
     * está tudo bem: um teste saudável daqui termina em menos de 700ms, então
     * este teto só é alcançado quando algo de fato travou.
     *
     * O número existe para absorver máquina fria, não para acomodar teste lento.
     * Se algum dia um teste chegar perto destes 15 segundos com a máquina
     * quente, o problema é o teste — e a resposta certa será consertá-lo, nunca
     * aumentar este valor.
     */
    testTimeout: 15_000,

    /**
     * Mesmo teto para `beforeAll`, `beforeEach` e companhia.
     *
     * Deixar o gancho com o padrão de 5s recriaria o problema em outro lugar:
     * é comum ser justamente o `beforeAll` quem paga a primeira importação.
     */
    hookTimeout: 15_000,
  },
})
```

### `vitest.global-setup.ts`

```ts
/**
 * Preparação do banco de teste
 *
 * Roda **uma vez**, antes de qualquer arquivo de teste, e garante que o banco de
 * teste tenha exatamente a estrutura declarada nas migrations.
 *
 * Por que aplicar as migrations em vez de criar as tabelas na mão: porque assim o
 * que os testes exercitam é a **mesma** estrutura que vai para o servidor. Um
 * banco de teste montado por outro caminho testa um sistema que não existe.
 *
 * O comando usado é o `migrate deploy`, e não o `migrate dev`: ele apenas aplica
 * o que já está versionado, sem tentar criar migration nova nem usar banco de
 * sombra. É o mesmo comando que roda em produção — assunto da próxima aula.
 */

import { execFileSync } from 'node:child_process'
import { existsSync } from 'node:fs'
import { join } from 'node:path'

export default function preparar(): void {
  const url = process.env.DATABASE_TEST_URL

  if (url === undefined || url === '') {
    throw new Error('DATABASE_TEST_URL não está definida. Copie-a do .env.example para o seu .env.')
  }

  // Chamamos o CLI pelo próprio Node, e não por `npx`. Dois motivos, ambos
  // medidos nesta máquina: no Windows o `npx` é um `.cmd`, que o Node recusa
  // executar sem `shell` (erro `EINVAL`); e ligar o `shell` faz o Node avisar
  // que os argumentos passam concatenados, em vez de um a um (aviso DEP0190).
  // Apontar para o arquivo do próprio pacote resolve os dois de uma vez, e
  // funciona igual nos três sistemas operacionais.
  const cliDoPrisma = join(process.cwd(), 'node_modules', 'prisma', 'build', 'index.js')

  if (!existsSync(cliDoPrisma)) {
    throw new Error('CLI do Prisma não encontrado. Rode `npm install` antes dos testes.')
  }

  // A variável é entregue só a este processo filho. O `.env` da máquina continua
  // apontando para o banco de trabalho, e nada aqui o altera.
  execFileSync(process.execPath, [cliDoPrisma, 'migrate', 'deploy'], {
    stdio: 'pipe',
    env: { ...process.env, DATABASE_URL: url },
  })
}
```

### `Dockerfile`

```dockerfile
# =============================================================================
# Imagem da API do Curso
# =============================================================================
#
# São dois estágios. O primeiro compila; o segundo roda. Só o resultado da
# compilação atravessa a fronteira entre eles, e é por isso que a imagem final
# não carrega TypeScript, testes nem ferramentas de qualidade.
#
# A versão do Node está fixada na linha do `FROM`, e é ela — não a versão
# instalada na máquina de quem faz o build — que roda em produção.


# ---------- Estágio 1: build ----------
# A `major` é fixada de propósito, e não a versão exata. Assim a imagem recebe
# correção de segurança do Node sem ninguém precisar editar arquivo, e a única
# mudança capaz de quebrar de verdade, que é a virada de major, continua travada.
FROM node:24-slim AS build

WORKDIR /app

# Os manifestos vêm antes do código de propósito. Cada instrução deste arquivo
# vira uma camada guardada em cache: enquanto `package.json` e `package-lock.json`
# não mudarem, o Docker reaproveita a instalação inteira e nem executa o `npm ci`
# de novo. Copiar o `src/` antes jogaria esse ganho fora a cada linha alterada.
COPY package.json package-lock.json ./

# `npm ci` em vez de `npm install`: ele instala exatamente o que está no
# `package-lock.json` e falha se os dois arquivos discordarem. É o que garante
# que o build de hoje e o de daqui a seis meses instalem as mesmas versões.
RUN npm ci

COPY tsconfig.json tsconfig.build.json ./

# O schema do Prisma precisa entrar antes do build: é dele que o `prisma generate`
# — embutido no `npm run build` — produz o cliente TypeScript em `src/generated`.
# Sem esta linha, a compilação falha por não encontrar os tipos, e falha aqui, no
# build, que é o melhor lugar possível para descobrir.
COPY prisma.config.ts ./
COPY prisma ./prisma

COPY src ./src

RUN npm run build


# ---------- Estágio 2: produção ----------
FROM node:24-slim AS producao

# Declarado aqui, e não só no `docker run`, para que o padrão da imagem já seja o
# comportamento de produção. Quem quiser outro valor passa `-e NODE_ENV=...`.
ENV NODE_ENV=production

WORKDIR /app

COPY package.json package-lock.json ./

# `--omit=dev` deixa de fora TypeScript, Vitest, ESLint e Prettier: são
# ferramentas de quem escreve o código, não de quem o executa. O `cache clean`
# apaga o que o npm guardou durante a instalação, que não serve para nada dentro
# da imagem e ocuparia espaço em todas as cópias dela.
#
# O `--legacy-peer-deps` entrou aqui por um número medido, e não por hábito. O
# `@prisma/client` declara o CLI `prisma` como peer **opcional**; com a resolução
# normal de peers, o npm traz o CLI inteiro para a imagem de produção — e junto
# vêm o Prisma Studio, os engines e as ferramentas de desenvolvimento:
#
#   sem a opção .... 898 MB, com `prisma` dentro
#   com a opção .... 494 MB, sem `prisma` dentro
#
# A opção diz ao npm para não resolver peers, e aqui isso é exatamente o
# desejado: tudo o que a API executa em produção — `@prisma/client` e
# `@prisma/adapter-mariadb` — está declarado como dependência direta.
#
# **Isto vale só para esta linha.** Na sua máquina, `npm install` continua tendo
# de concluir sem essa opção: lá ela esconderia incompatibilidade de verdade, que
# é o que a Aula 01 ensinou a não varrer para debaixo do tapete.
RUN npm ci --omit=dev --legacy-peer-deps && npm cache clean --force

# A única coisa que atravessa do estágio anterior. O código-fonte fica para trás.
COPY --from=build --chown=node:node /app/dist ./dist

# As imagens oficiais do Node já trazem um usuário sem privilégios chamado
# `node`. Rodar como `root` dentro do container é o padrão, mas não é motivo:
# se alguém conseguir execução aqui dentro, faz diferença o que ele pode fazer.
USER node

# `EXPOSE` não publica porta nenhuma — quem publica é o `-p` do `docker run`.
# Esta linha é documentação legível por ferramentas: diz em que porta esta imagem
# espera atender.
EXPOSE 3333

# A verificação bate na rota de vida, e não na de prontidão: `/health/ready`
# devolve uptime, ambiente e timestamp, que são informação interna. A porta é
# lida do ambiente porque cravar 3333 faria o container aparecer como `unhealthy`
# no dia em que alguém o subisse em outra porta, com a API perfeitamente no ar.
#
# Não usamos `curl`: esta imagem não o traz, e instalá-lo só para isso aumentaria
# o tamanho e a superfície de ataque. O Node já tem `fetch` embutido.
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||3333)+'/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

# Chamamos o `node` direto, e não o `npm start`. Dois motivos: o `npm start` deste
# projeto procura um arquivo `.env`, que não existe dentro do container — aqui a
# configuração chega por variável de ambiente, que é o contrato do Docker. E sem
# o `npm` no meio, o sinal de desligamento que o Docker envia chega à API sem
# intermediário.
CMD ["node", "dist/server.js"]
```

### `.vscode/extensions.json`

A extensão do Prisma entra agora, e não antes: ela dá realce e formatação ao `schema.prisma`,
que só passou a existir nesta aula.

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
    "vitest.explorer",

    // --- Empacotamento ---
    // Realce de sintaxe no Dockerfile e lista visual de imagens e containers.
    // É conforto, não caminho: o container se opera pelo terminal, que é o único
    // lugar que também existe no servidor.
    "ms-azuretools.vscode-docker",

    // --- Banco de dados ---
    // Realce e formatação do schema.prisma, e conferência dos nomes de campo
    // enquanto se escreve o modelo. O arquivo é a fonte da verdade da estrutura
    // do banco: vale a pena escrevê-lo com o editor ajudando.
    "Prisma.prisma"
  ]
}
```

### `package.json`

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
    "predev": "prisma generate",
    "dev": "tsx watch --env-file=.env src/server.ts",
    "prebuild": "node --eval \"require('node:fs').rmSync('dist', { recursive: true, force: true })\"",
    "build": "prisma generate && tsc --project tsconfig.build.json",
    "start": "node --env-file-if-exists=.env dist/server.js",
    "pretest": "prisma generate",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "docker:build": "docker build -t curso_api .",
    "docker:run": "docker run --rm -p 3333:3333 --name curso_api curso_api",
    "compose:up": "docker compose up -d",
    "compose:down": "docker compose down",
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio",
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
    "@prisma/adapter-mariadb": "^7.9.1",
    "@prisma/client": "^7.9.1",
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
    "prisma": "^7.9.1",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0",
    "vitest": "^4.1.10"
  }
}
```

### `.gitignore`

```bash
# =============================================
# Dependências
# =============================================
node_modules/

# =============================================
# Build (arquivos compilados pelo TypeScript)
# =============================================
dist/

# =============================================
# Cliente do Prisma (gerado a partir do schema)
# =============================================
# É código gerado, como o dist/: nasce do `prisma generate` e seria reescrito
# inteiro a cada alteração do schema. Versioná-lo obrigaria a revisar milhares
# de linhas que ninguém escreveu.
src/generated/

# =============================================
# Variáveis de ambiente (senhas, tokens, etc.)
# =============================================
.env
.env.local
.env.*.local

# =============================================
# Logs
# =============================================
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# =============================================
# Sistema operacional
# =============================================
.DS_Store
Thumbs.db

# =============================================
# IDE / Editor
# =============================================
.idea/
*.swp
*.swo
```

### `.prettierignore`

```bash
# =============================================
# Código gerado pelo compilador
# =============================================
dist/

# =============================================
# Código gerado pelo Prisma
# =============================================
src/generated/

# =============================================
# Dependências
# =============================================
node_modules/

# =============================================
# Arquivos gerados automaticamente pelo npm
# =============================================
package-lock.json
```

### `eslint.config.js`

```js
// @ts-check

/**
 * Configuração do ESLint (formato "flat config").
 *
 * O ESLint é o nosso analisador de qualidade: ele procura problemas de lógica e
 * más práticas. Quem cuida da aparência do código (espaços, aspas, quebras de
 * linha) é o Prettier, executado separadamente pelo comando `npm run format`.
 *
 * Manter as duas ferramentas separadas é uma decisão consciente: assim um espaço
 * sobrando nunca aparece com a mesma gravidade visual de um bug de verdade.
 */

import { defineConfig, globalIgnores } from 'eslint/config'
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import prettier from 'eslint-config-prettier'

export default defineConfig([
  // Pastas que o ESLint nunca deve analisar: código gerado e bibliotecas de terceiros.
  globalIgnores(['dist/**', 'node_modules/**', 'src/generated/**']),

  {
    files: ['**/*.ts'],
    extends: [
      js.configs.recommended,
      tseslint.configs.recommended,
      // Precisa ser o ÚLTIMO da lista: desliga as regras do ESLint que brigariam
      // com a formatação do Prettier.
      prettier,
    ],
    rules: {
      // O tipo `any` desliga a checagem do TypeScript naquele ponto. Às vezes é
      // inevitável, por isso é aviso e não erro — mas precisa ser uma decisão
      // consciente, nunca um descuido.
      '@typescript-eslint/no-explicit-any': 'warn',

      // Em produção usamos o logger do Fastify (`app.log`), que gera JSON
      // estruturado e pode ser filtrado por nível. `console.log` escreve texto
      // solto, que as ferramentas de monitoramento não conseguem indexar.
      //
      // É `error`, e não `warn`: aviso que não reprova nada é aviso que o time
      // aprende a ignorar. As exceções legítimas continuam possíveis, mas exigem
      // um `eslint-disable` com o motivo escrito ao lado.
      'no-console': 'error',

      // Variável declarada e não usada quase sempre indica código morto ou um
      // erro de digitação. O prefixo `_` marca o caso em que ignorar é proposital.
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_', varsIgnorePattern: '^_' },
      ],
    },
  },
])
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

# ---------------------------------------------
# Banco de dados (lido pelo Docker Compose)
# ---------------------------------------------
# Estas variáveis NÃO são lidas pela API — ela ainda não fala com o banco. Quem
# as consome é o serviço `mysql` do `docker-compose.yml`, na primeira vez que o
# container sobe: é com elas que o MySQL cria o banco e o usuário da aplicação.
#
# Por isso elas também não aparecem no `src/shared/env/env.schema.ts`: aquele
# arquivo é o contrato do que a API precisa, e a API não precisa disto ainda.

# Senha do usuário administrador do MySQL.
MYSQL_ROOT_PASSWORD=troque-esta-senha

# Banco criado automaticamente quando o container sobe pela primeira vez.
MYSQL_DATABASE=curso_api

# Usuário da aplicação, criado com acesso apenas ao banco acima. Rodar a
# aplicação como administrador do banco é o mesmo erro que rodá-la como `root`
# dentro do container: funciona, e amplia o estrago de qualquer falha.
MYSQL_USER=curso_api
MYSQL_PASSWORD=troque-esta-senha-tambem

# Porta em que o MySQL do Compose atende na SUA máquina. Só é necessária porque
# o `npm run dev` roda fora do container e precisa alcançar o banco. Se você já
# tiver um MySQL instalado ocupando a 3306, troque aqui — nada dentro do Compose
# precisa saber, porque lá dentro os serviços se falam pelo nome.
MYSQL_PORT=3306

# ---------------------------------------------
# Banco de dados da aplicação
# ---------------------------------------------
# Endereço completo do MySQL: usuário, senha, servidor, porta e nome do banco.
# A porta é a que o Compose publica NA SUA MÁQUINA (a `MYSQL_PORT` acima) —
# dentro da rede do Compose os serviços se falam pelo nome, na 3306 de sempre.
#
# Esta é a única das três que a API lê: ela está no `env.schema.ts` e não tem
# valor padrão, porque subir apontando para um banco inventado é pior do que
# não subir.
DATABASE_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api"

# Banco usado pelos testes automatizados. Quem o lê é o `vitest.config.ts`, que
# troca a `DATABASE_URL` durante o `npm test` — é o que impede a suíte de apagar
# o banco em que você estava trabalhando.
DATABASE_TEST_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api_test"

# Banco descartável que o `prisma migrate dev` usa para conferir as migrations.
# Quem o lê é o CLI do Prisma, pelo `prisma.config.ts`. Ele existe pronto porque
# o usuário da aplicação, de propósito, não tem permissão para criar bancos.
SHADOW_DATABASE_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api_shadow"
```

### `README.md`

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
| `DATABASE_URL` | Endereço do MySQL; obrigatória, sem valor padrão  | _(nenhum)_       |

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
| `npm run compose:up`   | Sobe o ambiente completo: API e banco de dados     |
| `npm run compose:down` | Derruba o ambiente, preservando os dados           |
| `npm run db:generate`  | Gera o cliente do Prisma a partir do schema        |
| `npm run db:migrate`   | Cria e aplica uma migration no banco local         |
| `npm run db:seed`      | Popula o banco com registros de exemplo            |
| `npm run db:studio`    | Abre o Prisma Studio para ver e editar dados       |

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

A contagem usa o IP de quem chamou — e, atrás de proxy, só acerta com o `TRUST_PROXY` ligado,
como explica a seção de container mais abaixo.

**Limite conhecido:** a contagem vive na memória do processo. Duas cópias da API contam
separado, e reiniciar zera o placar. Sair disso exige guardar a contagem fora do processo.

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

## Subindo o ambiente completo

O `docker-compose.yml` sobe a API e o MySQL juntos, na ordem certa, em qualquer máquina:

```bash
npm run compose:up     # sobe API e banco
npm run compose:down   # derruba, preservando os dados
```

**No dia a dia, o comando de trabalho continua sendo o `npm run dev`**, rodando na sua máquina
e apontando para o MySQL do Compose, cuja porta está publicada. O recarregamento automático
não se perde: ele apenas não vem do container.

Quatro coisas que esta configuração resolve:

- **O banco só recebe conexão quando responde de verdade.** O `depends_on` sozinho espera o
  container **existir**, não o serviço responder. Medido: o container do MySQL ficou `running`
  em 1,0 s e o banco só aceitou conexão 19,9 s depois, na primeira subida — cerca de 5 s nas
  seguintes, com o volume já inicializado. O `healthcheck` mais o `condition: service_healthy`
  fecham esse vão.
- **Os serviços se acham pelo nome.** Dentro da rede do projeto, `mysql` é endereço; ninguém
  configura IP.
- **Os dados sobrevivem ao `down`**, porque ficam em volume nomeado. Quem apaga é o `down -v`,
  que por isso não entra em nenhum atalho do `package.json`.
- **A senha não fica no YAML.** As variáveis `MYSQL_*` vivem no `.env`, que o Compose lê
  sozinho. Elas **não** passam pelo `envSchema`: quem as consome é o serviço `mysql`, não a
  API — que ainda não fala com o banco.

Se a sua máquina já tiver um MySQL ocupando a 3306, troque `MYSQL_PORT` no `.env`. Nada dentro
do Compose precisa saber.

## Banco de dados

O MySQL sobe junto com a API pelo Compose. A estrutura das tabelas é versionada em
`prisma/migrations/`: cada alteração vira um arquivo SQL que dá para ler e revisar, ao lado
do código que depende dela.

```bash
npm run db:migrate     # cria e aplica uma migration a partir do schema
npm run db:seed        # popula o banco com registros de exemplo (idempotente)
npm run db:studio      # abre a tela de ver e editar dados
```

Três decisões que valem lembrar:

- **O cliente do Prisma é gerado, e não versionado.** Ele nasce em `src/generated/prisma` a
  partir do schema, e os ganchos `predev` e `pretest` — mais o próprio `build` — o geram
  sozinhos. Clone novo funciona sem ninguém precisar lembrar de nada.
- **Os testes usam um banco separado** (`curso_api_test`). O `vitest.config.ts` troca a
  `DATABASE_URL` durante o `npm test`; sem isso, a suíte apagaria o banco em que você
  trabalha.
- **O usuário da aplicação não pode criar bancos.** É decisão de segurança, e é por isso que o
  banco de sombra exigido pelo `prisma migrate dev` já vem pronto do Compose, apontado pela
  `SHADOW_DATABASE_URL`.
````

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

### 2. A tabela existe, com o índice único

```bash
docker compose exec mysql mysql -ucurso_api -ptroque-esta-senha-tambem curso_api -e "SHOW INDEX FROM cidadaos;"
```

Precisa aparecer `cidadaos_cpf_key`.

### 3. O banco reproduz do zero

```bash
docker compose down -v
npm run compose:up
npx prisma migrate deploy
npx prisma db seed
```

Sem erro, e com três cidadãos ao final.

### 4. Os testes não tocam o banco de trabalho

A sequência do Capítulo 7.

### 5. O projeto sobe sem o cliente gerado no disco

```bash
# apague src/generated e dist, e então:
npm run dev
```

### 6. O container consulta o banco

O comando do Capítulo 10.

---

## 🚨 Erros Comuns

### "P3014: could not create the shadow database"

Falta o banco de sombra, ou falta o `SHADOW_DATABASE_URL` no seu `.env`. Se você já tinha o
volume criado antes desta aula, o script de inicialização não rodou: `docker compose down -v` e
suba de novo. **A saída errada é dar permissão de criar banco ao usuário da aplicação.**

### "Cannot find module 'src/generated/prisma/client.ts'"

O cliente não foi gerado. `npm run db:generate` resolve. Se acontecer sempre, confira se os
scripts `predev` e `pretest` estão no `package.json`.

### "ENOENT: no such file or directory, open '.env'"

O `prisma.config.ts` está tentando ler o `.env` onde ele não existe — tipicamente dentro do
container. Confira o `if (existsSync('.env'))`.

### "Error: Environment variable not found: DATABASE_URL"

O CLI não achou a variável. Confira se ela está no `.env` **e** se o `process.loadEnvFile()`
está sendo chamado no `prisma.config.ts`.

### "Access denied for user 'curso_api'@'%' to database 'curso_api_test'"

O `GRANT` do script de inicialização não rodou — de novo, o volume é anterior a ele.

### "Added the required column ... There are N rows in this table"

O Capítulo 8 inteiro. Torne o campo opcional, dê um valor padrão, ou faça em etapas.

### "O `npm run db:studio` trava e não abre nada"

Ele está esperando abrir o navegador. Acontece em ambiente sem navegador — servidor por SSH,
alguns terminais dentro de container. Rode apontando a porta e dispensando o navegador, e
depois abra o endereço à mão:

```bash
npx prisma studio --port 5555 --browser none
```

### O `npm test` apagou meus dados

Confira a linha `DATABASE_URL` dentro do `env` do `vitest.config.ts`. Se ela estiver ausente, os
testes rodaram no banco de trabalho.

---

## 🏋️ Exercícios

### 1. Acrescente um campo e leia o SQL antes de aplicar

Adicione `dataNascimento DateTime?` ao modelo, gere a migration com
`prisma migrate dev --create-only` (que **cria sem aplicar**), leia o SQL, e só então aplique
com `prisma migrate dev`. Explique em uma frase o que o `--create-only` permite fazer.

### 2. Prove que o índice único é do banco, não do código

Sem passar pelo service, insira dois cidadãos com o mesmo CPF direto pelo cliente do Prisma.
Descreva o erro que aparece e diga quem o gerou.

### 3. Descubra o que o `migrate deploy` faz de diferente

Rode `npx prisma migrate status` e depois `npx prisma migrate deploy` com o banco já em dia.
Explique por que ele não precisa do banco de sombra.

### 4. Meça o custo do cliente na imagem

Construa a imagem sem o `--legacy-peer-deps`, meça, e depois com ele. Explique, com os dois
números na mão, o que exatamente saiu de dentro da imagem.

Os gabaritos comentados estão em [`exercicios/13-gabarito.md`](./exercicios/13-gabarito.md).

---

## 📌 O que vem depois

A API guarda dado, a estrutura do banco está versionada, e os testes provam as duas coisas
contra um MySQL de verdade.

Falta a pergunta que muda tudo: **como rodar uma migration em um banco que já tem dados de
gente**. Na sua máquina, se der errado, você apaga e recomeça. Em produção não existe essa
opção — e é por isso que a **Aula 14** é inteira sobre isso: `migrate deploy`, o padrão
expande/contrai, em que momento do deploy a migration roda, o que fazer quando ela falha no meio
e o plano de retorno escrito **antes**.
