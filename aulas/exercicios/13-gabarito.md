# 🗄️ Gabarito — Aula 13: Banco de dados

> Confira só depois de tentar. Os quatro exercícios foram executados antes de respondidos, e
> os números e mensagens abaixo saíram dessa execução.

---

## Exercício 1 — Acrescente um campo e leia o SQL antes de aplicar

> Adicione `dataNascimento DateTime?` ao modelo, gere a migration com `--create-only`, leia o
> SQL, e só então aplique. Explique em uma frase o que o `--create-only` permite fazer.

### Os comandos

No `prisma/schema.prisma`, dentro do `model Cidadao`:

```prisma
  dataNascimento DateTime?
```

```bash
npx prisma migrate dev --name adiciona-data-nascimento --create-only
```

O arquivo nasce, e **nada** foi aplicado ainda:

```sql
-- AlterTable
ALTER TABLE `cidadaos` ADD COLUMN `dataNascimento` DATETIME(3) NULL;
```

Confira no banco, antes de aplicar:

```bash
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api \
  -e "SHOW COLUMNS FROM cidadaos LIKE 'dataNascimento';"
```

A saída é **vazia**: a coluna não existe. Agora aplique:

```bash
npx prisma migrate dev
```

```
Your database is now in sync with your schema.
```

E a coluna aparece:

```
Field           Type          Null   Key   Default
dataNascimento  datetime(3)   YES          NULL
```

### A frase

O `--create-only` **separa gerar de aplicar**: ele escreve o SQL e para, permitindo ler,
revisar e até editar o arquivo antes que ele toque no banco.

### Por que isso importa mais do que parece

Neste exercício o SQL é de uma linha e não há o que revisar. O valor aparece quando a
alteração é destrutiva — remover coluna, renomear tabela, mudar tipo — ou quando ela precisa
ser feita em etapas, com dado sendo copiado no meio. Nesses casos, o SQL gerado é o rascunho,
e você é quem decide o que de fato vai rodar.

É também o único caminho honesto para o padrão expande/contrai, que é a Aula 14.

> [!TIP]
> Ao terminar, desfaça: apague a pasta da migration e a linha do schema. Assim o seu projeto
> volta a bater com o material.

---

## Exercício 2 — Prove que o índice único é do banco, não do código

> Sem passar pelo service, insira dois cidadãos com o mesmo CPF direto pelo cliente do Prisma.
> Descreva o erro e diga quem o gerou.

### O que foi feito

```ts
await prisma.cidadao.create({ data: { nome: 'Primeiro', cpf: '11111111111' } })
await prisma.cidadao.create({ data: { nome: 'Segundo', cpf: '11111111111' } })
```

### O resultado

```
recusado por: PrismaClientKnownRequestError
mensagem: Unique constraint failed on the constraint: `cidadaos_cpf_key`
```

### Quem gerou

O **banco**. O nome que aparece na mensagem — `cidadaos_cpf_key` — é o do índice criado pela
primeira migration, a partir do `@unique` do schema. Nenhuma linha de TypeScript foi
consultada: a segunda inserção chegou ao MySQL e foi recusada lá.

### O que isso ensina sobre a regra do service

A checagem no `CidadaoService` **não** é o que garante a unicidade. Ela existe para produzir
uma mensagem em português, com status 409, para quem consome a API — porque a mensagem acima,
que fala em `constraint` e cita o nome do índice, é escrita para desenvolvedor e revelaria
estrutura interna, o que a Aula 06 proibiu.

Tire a regra do service e o dado continua íntegro. Tire o `@unique` do banco e o dado passa a
depender de ninguém esquecer a regra — inclusive em caminhos de cadastro que ainda nem
existem.

---

## Exercício 3 — Descubra o que o `migrate deploy` faz de diferente

> Rode `prisma migrate status` e depois `prisma migrate deploy` com o banco já em dia. Explique
> por que ele não precisa do banco de sombra.

### O resultado

```
$ npx prisma migrate status
3 migrations found in prisma/migrations
Database schema is up to date!

$ npx prisma migrate deploy
No pending migrations to apply.
```

### Por que ele não precisa do banco de sombra

Porque os dois comandos respondem perguntas diferentes:

| Comando          | O que ele faz                                                      | Precisa de sombra? |
| :--------------- | :----------------------------------------------------------------- | :----------------: |
| `migrate dev`    | Compara o **schema** com o banco, **cria** migration nova e aplica |        sim         |
| `migrate deploy` | Aplica as migrations que já existem, na ordem, e para              |        não         |

O banco de sombra existe para uma conferência que só faz sentido ao **criar** migration:
aplicar todas em ordem, do zero, e verificar se o resultado bate com o schema declarado. É
assim que o Prisma percebe uma migration editada à mão depois de aplicada.

O `deploy` não cria nada e não compara com o schema — ele confia no que está versionado. Por
isso ele roda em servidor, onde criar bancos descartáveis seria, no mínimo, um susto.

> [!NOTE]
> É o mesmo comando que o `vitest.global-setup.ts` usa para preparar o banco de teste, e pelo
> mesmo motivo: lá também não se cria migration, só se aplica o que já existe.

---

## Exercício 4 — Meça o custo do cliente na imagem

> Construa a imagem sem o `--legacy-peer-deps`, meça, e depois com ele. Explique, com os dois
> números na mão, o que exatamente saiu de dentro da imagem.

### Os números medidos

| Instalação no estágio de produção      | Imagem     | `node_modules` | CLI `prisma` dentro? |
| :------------------------------------- | :--------- | :------------- | :------------------- |
| `npm ci --omit=dev`                    | **898 MB** | 400 MB         | sim                  |
| `npm ci --omit=dev --legacy-peer-deps` | **494 MB** | 122 MB         | não                  |

### O que saiu

```
19M   @prisma/dev
28M   @prisma/engines
43M   @prisma/studio-core
42M   prisma           (o CLI)
```

Ferramentas de **desenvolvimento**: o Prisma Studio, o CLI que cria migrations, os engines que
o CLI usa. Nada disso é executado pela API — que precisa apenas do `@prisma/client` e do
`@prisma/adapter-mariadb`, ambos dependências diretas e ambos permanecem na imagem.

### Por que eles estavam lá

O `@prisma/client` declara o CLI `prisma` como **peer dependency opcional**, e o npm resolve
peers sozinho, mesmo com `--omit=dev`. Dá para ver a corrente de dentro do container:

```bash
docker compose exec api npm ls prisma
```

```
@prisma/client@7.9.1
  `-- prisma@7.9.1
    `-- @prisma/studio-core@0.24.x
```

### A ressalva que vale mais que o número

O `--legacy-peer-deps` está **naquela linha do `Dockerfile`, e em nenhum outro lugar**. Na sua
máquina, `npm install` continua tendo de concluir sem ele: lá a opção esconderia
incompatibilidade real, que é o problema que a Aula 01 ensinou a não varrer para debaixo do
tapete.

A diferença é o objetivo. Na imagem, queremos deliberadamente **menos** do que o npm
instalaria. Na sua máquina, queremos exatamente o que o projeto declara.
