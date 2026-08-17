# Gabarito — Aula 04

---

## 1. Uma variável nova

São **três** arquivos, e o terceiro é o que quase todo mundo esquece.

**`src/shared/env/env.schema.ts`** — acrescente ao `envSchema`:

```typescript
  /**
   * Nome de exibição da API, usado em logs e na documentação.
   */
  NOME_DA_API: z.string().min(1, { error: 'não pode ficar vazio' }).default('API do Curso'),
```

**`src/server.ts`** — inclua no log de partida:

```typescript
    app.log.info(
      { api: env.NOME_DA_API, port: env.PORT, host: env.HOST, ambiente: env.NODE_ENV },
      'Servidor iniciado com sucesso',
    )
```

**`.env.example`** — acrescente:

```ini
# Nome de exibição da API
NOME_DA_API=API do Curso
```

### Por que o terceiro passo importa

O código funciona sem ele. É justamente por isso que ele é esquecido.

O `.env.example` é a **única documentação** de quais variáveis existem. Quem clona o projeto
copia esse arquivo e assume que ali está tudo.

Se você adicionar uma variável no schema e não no modelo, cria uma armadilha silenciosa:
enquanto a variável tiver `.default()`, ninguém percebe a falta. No dia em que alguém
adicionar uma variável **obrigatória** e esquecer do modelo, o colega novo clona o projeto,
copia o `.env.example`, e a API recusa subir por causa de uma variável que ele não tinha como
adivinhar que existia.

**Regra prática:** mexeu no `envSchema`, mexeu no `.env.example`. Sempre. Os dois andam juntos.

---

## 2. Torne uma variável obrigatória

Removendo o `.default('0.0.0.0')` do `HOST` e apagando a linha do `.env`:

```
❌ Configuração inválida. A API não foi iniciada.

   HOST: Invalid input: expected string, received undefined

   Confira o seu arquivo .env. O modelo está em .env.example.
```

**A API recusou subir — que era o esperado.** Mas repare em algo:

### A mensagem saiu em inglês. Por quê?

Este exercício expõe uma lacuna real do nosso schema, e é de propósito.

Olhe como escrevemos o `HOST`:

```typescript
HOST: z.string().min(1, { error: 'não pode ficar vazio' }).default('0.0.0.0'),
```

Nossa mensagem em português está presa ao **`.min(1)`**, que só é verificado quando o valor
existe e é um texto. Quando a variável simplesmente **não existe**, quem reprova é a checagem
de tipo do `z.string()` — e essa nunca recebeu mensagem customizada. O Zod então usa a dele,
que vem em inglês.

### Como consertar

A mensagem para o caso "ausente ou de tipo errado" vai no próprio `z.string()`:

```typescript
HOST: z
  .string({ error: 'é obrigatória e deve ser um texto' })
  .min(1, { error: 'não pode ficar vazio' }),
```

Agora as duas situações têm mensagem em português:

```
   HOST: é obrigatória e deve ser um texto
```

### A lição que fica

**Mensagem de erro também precisa ser testada.** É muito fácil escrever uma validação, ver o
caso que você imaginou funcionando, e nunca descobrir que o _outro_ caminho produz uma
mensagem inútil.

E foi exatamente isso que aconteceu no nosso schema: como as três variáveis têm `.default()`,
o caso "ausente" nunca acontece hoje, e o buraco ficou invisível. Ele só apareceria quando
alguém adicionasse a primeira variável obrigatória de verdade — provavelmente a
`DATABASE_URL`, na aula de banco de dados.

Vale anotar num lugar visível para ser resolvido antes disso — buraco conhecido e não
anotado é buraco que volta.

**Não esqueça de desfazer as duas alterações** antes de seguir.

---

## 3. Investigue o `coerce`

Trocando `z.coerce.number()` por `z.number()`, mesmo com um `.env` correto:

```
❌ Configuração inválida. A API não foi iniciada.

   PORT: deve ser um número inteiro (recebido: "3333")
```

**A API não sobe.** E repare no detalhe que torna isso confuso à primeira vista: o valor
recebido foi `"3333"`, que _parece_ perfeitamente válido.

### O que está acontecendo

**Toda variável de ambiente é texto.** Sempre. Não existe número em variável de ambiente —
o sistema operacional só sabe guardar texto.

Então o que chega ao schema é `"3333"` (com aspas), e não `3333`.

- `z.number()` verifica: "isto é um número?" → não, é texto → **reprova**.
- `z.coerce.number()` verifica: "consigo converter isto em número?" → `Number("3333")` dá
  `3333` → **aprova**.

O `coerce` (do inglês _coerce_, forçar) faz a conversão **antes** de validar.

### E por que isso resolve o bug da aula

É aqui que a peça se encaixa. Com o valor errado do começo da aula:

- `Number("8O80")` devolve `NaN`.
- `NaN` não é um número válido, então o Zod **reprova**.

Compare com o código antigo, do Capítulo 1:

```typescript
Number("8O80") || 3333   // NaN é "falso" → resultado: 3333, calado
```

O `||` tratava `NaN` como "não veio nada" e usava o valor reserva. O Zod trata `NaN` como
**"veio, e está errado"** — que é a interpretação correta.

**A diferença toda está em distinguir "ausente" de "inválido".** Ausente pode ter valor
padrão. Inválido nunca pode.

---

## 4. Descubra a ordem de precedência

Com `PORT=3333` no `.env` e rodando `PORT=5000 npm run dev`:

```json
{"level":30,"port":5000,"host":"0.0.0.0","ambiente":"development","msg":"Servidor iniciado com sucesso"}
```

**Subiu na 5000.** A variável do ambiente venceu a do arquivo.

### Por que essa ordem é a correta

Pense em quem define cada uma:

| Origem   | Quem define                              | Quando            |
| :------- | :--------------------------------------- | :---------------- |
| `.env`   | O desenvolvedor, na própria máquina      | Enquanto programa |
| Ambiente | O servidor ou o container que roda a API | Em produção       |

Se fosse ao contrário — se o arquivo vencesse — um `.env` esquecido dentro da imagem Docker
sobrescreveria a configuração real do servidor. A aplicação subiria em produção apontando
para o banco de teste de alguém.

Por isso a regra: **quem está mais perto de produção manda mais.**

É o mesmo motivo pelo qual, no Capítulo 8, a API subiu normalmente sem nenhum arquivo `.env`.
Em produção o arquivo não existe, e não faz falta: as variáveis vêm do ambiente, que é a
origem de maior autoridade.

---

## 5. `.env.example` no Git, `.env` fora

Resposta esperada, com suas palavras:

> O `.env.example` é o **modelo**: ele diz quais variáveis existem, sem revelar nenhum valor
> real. Precisa estar no Git para que qualquer pessoa que clone o projeto saiba o que
> configurar, sem precisar perguntar para alguém.
>
> O `.env` tem os **valores de verdade** daquela máquina — incluindo senhas de banco e chaves
> de acesso. Se fosse para o Git, esses segredos ficariam visíveis para todo mundo com acesso
> ao repositório, e permaneceriam no histórico para sempre.

### O que aconteceria de concreto se fosse ao contrário

**Se o `.env` fosse versionado:**

Se o repositório for público, existem programas rodando 24 horas por dia varrendo o GitHub
atrás exatamente disso. Uma senha de banco exposta é encontrada em **minutos**, não em dias.

E tem o agravante que quase ninguém entende na hora: apagar o arquivo no commit seguinte
**não resolve**. O Git guarda histórico — o valor continua acessível em qualquer commit
anterior. A única resposta correta é considerar a senha comprometida e **trocá-la** em todos
os serviços afetados, imediatamente.

Some-se a isso o conflito diário: cada pessoa da equipe teria valores diferentes no mesmo
arquivo, e todo `git pull` viraria uma briga.

**Se o `.env.example` não fosse versionado:**

O prejuízo é menor, mas real. Todo colega novo teria que descobrir na tentativa e erro quais
variáveis existem — subir a aplicação, ler a mensagem de erro, criar a variável, subir de
novo, ler o próximo erro. Meia hora de trabalho que um arquivo de exemplo resolve em trinta
segundos.

E, sem o modelo, ninguém percebe quando uma variável nova é adicionada ao projeto.
