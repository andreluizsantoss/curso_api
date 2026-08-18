# 🧑‍💼 Aula 15: a primeira funcionalidade de negócio (e o endereço dela)

Catorze aulas construíram um servidor que valida o que entra, documenta a si mesmo, se defende
de quem abusa dele, morre sem cortar requisição no meio, sobe inteiro com um comando e sabe
alterar o próprio banco sem derrubar ninguém.

E a única coisa que ele responde, até agora, é se está vivo.

Hoje isso muda. Vamos construir o **cadastro de cidadão** — a primeira rota que existe para
alguém de fora, e não para a equipe que opera o serviço. Junto vêm duas perguntas que só fazem
sentido agora que existe rota de negócio:

1. **Que dados um cadastro de pessoa realmente precisa guardar** — e o que fazer com o que é
   dado pessoal?
2. **Em que endereço essa rota vive** — e o que acontece com quem a consome no dia em que ela
   mudar?

> [!IMPORTANT]
> A tabela `cidadaos` que você criou na Aula 13 era uma **demonstração**. Ela tinha os campos
> mínimos para ensinar banco de dados, e o próprio `schema.prisma` avisava isso em comentário.
> Hoje ela vira o cadastro de verdade. Fazer isso agora custa uma migration; fazer depois, com
> dado de gente dentro, custa o que a Aula 14 mostrou.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Desenhar uma tabela de dado pessoal decidindo **campo a campo**, com motivo escrito.
- Explicar por que **ninguém é apagado** e como funciona a exclusão lógica.
- Validar CPF pelos dígitos verificadores — e dizer o que isso **não** prova.
- Decidir o que **não** deve sair em cada resposta, e provar que não sai.
- Explicar o que obriga uma API a subir de versão, e o que não obriga.
- Dizer por que `/health` fica **fora** de `/api/v1`.
- Descrever o contrato de erro na documentação sem que ele encoste no serializador.

---

## 📋 Pré-requisitos

Você precisa ter concluído a **Aula 14** e ter o ambiente de pé:

```bash
npm run compose:up
npm run check
```

Nada novo para instalar nesta aula. Zod, Prisma e os schemas já estão no projeto desde as
Aulas 04, 07 e 13 — hoje é dia de usar o que já existe.

---

## 🧾 Capítulo 1: o cadastro que era demonstração

Abra o `prisma/schema.prisma` e leia o comentário que está lá desde a Aula 13. Ele lista, com
todas as letras, o que falta:

> nome social, data de nascimento, endereço, trilha de auditoria (quem cadastrou e quem
> alterou), exclusão lógica em vez de apagar a linha, e o tratamento de CPF exigido pela LGPD

Um comentário desses é uma dívida com data e dono. Hoje ela é paga.

### As três decisões que valem por todo o resto

**1. Quase tudo é opcional.** Só três campos são obrigatórios, e são os que identificam a
pessoa: CPF, nome completo e data de nascimento. O resto — RG, e-mail, telefone, endereço — é
opcional.

O motivo não é preguiça: é que um atendimento **não pode ser recusado** porque falta o
complemento do endereço. Campo obrigatório demais empurra quem atende a inventar valor para
preencher, e aí o cadastro fica pior do que se o campo estivesse vazio.

**2. Ninguém é apagado.** A exclusão preenche uma coluna `excluidoEm`, e todas as consultas
passam a filtrar quem ainda está ativo.

> **Analogia:** um cadastro num órgão público não é um post que se apaga. É mais parecido com
> uma pasta no arquivo morto: ela sai da mesa, para de aparecer nas buscas do dia a dia, e
> continua existindo — porque protocolo, atendimento e requisição antigos apontam para ela.
> Um `DELETE` transformaria todo esse histórico em referência quebrada, sem desfazer.

**3. Os tipos são declarados.** O padrão do Prisma para texto é `VARCHAR(191)`, que serve para
tudo e não cabe em nada. CPF tem 11 dígitos. Sigla de UF tem 2. Um `uuid` tem 36 caracteres,
sempre. Declarar o tamanho real deixa o índice menor — e índice menor é consulta mais rápida.

### O modelo

Substitua o `model Cidadao` inteiro pelo que está abaixo. O arquivo completo está no
Capítulo 12; o que importa aqui é ler os comentários, porque cada um responde a um "por quê".

```prisma
model Cidadao {
  // ── Identificação ─────────────────────────────────────────────────────────
  id String @id @default(uuid()) @db.Char(36)

  cpf String @unique @db.Char(11)

  nomeCompleto String @db.VarChar(150)

  nomeSocial String? @db.VarChar(150)

  dataNascimento DateTime @db.Date

  // ── Documento de identidade ───────────────────────────────────────────────
  rg           String? @db.VarChar(20)
  orgaoEmissor String? @db.VarChar(20)
  ufEmissor    String? @db.Char(2)

  // ── Contato ───────────────────────────────────────────────────────────────
  email String? @db.VarChar(254)

  telefone String? @db.VarChar(11)

  // ── Endereço ──────────────────────────────────────────────────────────────
  cep         String? @db.Char(8)
  logradouro  String? @db.VarChar(150)
  numero      String? @db.VarChar(10)
  complemento String? @db.VarChar(60)
  bairro      String? @db.VarChar(80)
  municipio   String? @db.VarChar(80)
  uf          String? @db.Char(2)

  // ── Trilha de auditoria ───────────────────────────────────────────────────
  criadoEm      DateTime  @default(now())
  criadoPor     String?   @db.VarChar(60)
  atualizadoEm  DateTime  @updatedAt
  atualizadoPor String?   @db.VarChar(60)

  // ── Exclusão lógica ───────────────────────────────────────────────────────
  excluidoEm  DateTime?
  excluidoPor String?   @db.VarChar(60)

  @@index([excluidoEm])
  @@map("cidadaos")
}
```

### Por que cada decisão, em uma tabela

| Decisão                                      | Motivo                                                                                                                                                             |
| :------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id` em `Char(36)`                           | O `uuid` tem 36 caracteres, sempre. É a chave primária, o índice mais consultado da tabela.                                                                        |
| `cpf` em `Char(11)`, só dígitos              | Formatar é trabalho de quem exibe. Guardar formatado deixa o mesmo CPF entrar duas vezes em grafias diferentes — e para o banco são textos diferentes.             |
| `nomeSocial` **ao lado** do nome de registro | Nome social é direito, não apelido: é por ele que a pessoa deve ser tratada. Substituir um pelo outro resolveria um requisito quebrando o outro.                   |
| `dataNascimento` em `@db.Date`               | Data de nascimento não tem hora. Guardar `DATETIME` convida ao bug de fuso: quem nasceu dia 1º aparece como dia 31 do mês anterior.                                |
| `dataNascimento` obrigatória                 | É o desempate quando dois cadastros têm nome parecido, e sustenta qualquer regra por faixa etária.                                                                 |
| Endereço em campos separados                 | Endereço inteiro em uma coluna só é impossível de consultar por bairro ou município — que é exatamente o que um órgão público pergunta.                            |
| `ufEmissor` junto do `rg`                    | RG é estadual: o mesmo número existe em unidades federativas diferentes. O número sozinho não identifica nada.                                                     |
| `telefone` **sem** `@unique`                 | Duas pessoas da mesma casa dividem telefone. A Aula 14 mostrou o estrago de descobrir isso com o índice já em produção.                                            |
| `criadoPor` / `atualizadoPor` nuláveis       | Ainda não existe login: não há de quem tirar o nome. Elas nascem agora porque acrescentar coluna a tabela com dado de gente é caro — e fazer isso hoje custa zero. |
| `@@index([excluidoEm])`                      | Toda listagem passa a filtrar por "não excluído". Filtro que roda em toda consulta merece índice.                                                                  |

> [!NOTE]
> **As duas colunas de auditoria nascem vazias, e isso é dívida registrada, não esquecimento.**
> Elas existem hoje pelo motivo da Aula 14: coluna nova em tabela com dado de gente dentro é
> uma migration cara e arriscada; em tabela quase vazia, é de graça. Quem vai preenchê-las é a
> aula de autenticação, quando existir um usuário logado para registrar.

---

## 🧱 Capítulo 2: a migration que o Prisma se recusa a fazer

Com o schema salvo, o passo natural seria gerar a migration. Tente:

```bash
npx prisma migrate dev --create-only --name modelo_real_cidadao
```

E leia o que acontece:

```
⚠️ We found changes that cannot be executed:

  • Step 0 Added the required column `dataNascimento` to the `cidadaos` table without a
    default value. There are 3 rows in this table, it is not possible to execute this step.
```

**Pare e reconheça a cena.** É exatamente o assunto da Aula 14, agora acontecendo com você: uma
coluna obrigatória sendo acrescentada a uma tabela que já tem linhas. O Prisma pergunta o óbvio
— _e as três linhas que já existem, que data de nascimento elas têm?_

> [!WARNING]
> Na Aula 14 você viu que o **banco** aceita isso calado, preenchendo com string vazia (item
> M-83 do checklist). Quem recusa aqui é o `migrate dev`, que confere os dados **antes** de
> gerar o SQL. O `migrate deploy` não confere nada: ele executa o arquivo. Por isso a recusa
> só aparece na sua máquina, e nunca no servidor.

A saída é o **expande/contrai**, o mesmo padrão da aula passada, em três passos.

### Passo 1 — a coluna nasce opcional

No `schema.prisma`, temporariamente:

```prisma
  dataNascimento DateTime? @db.Date
```

Gere e aplique. Repare no SQL: **todas** as colunas novas entram como `NULL`, o que torna a
migration retrocompatível por construção — o código velho continua funcionando sobre o schema
novo, que é a janela que a Aula 14 ensinou a respeitar.

```sql
ALTER TABLE `cidadaos` DROP PRIMARY KEY,
    ADD COLUMN `dataNascimento` DATE NULL,
    ADD COLUMN `nomeSocial` VARCHAR(150) NULL,
    ADD COLUMN `excluidoEm` DATETIME(3) NULL,
    -- ... as demais colunas novas, todas NULL ...
    MODIFY `id` CHAR(36) NOT NULL,
    MODIFY `cpf` CHAR(11) NOT NULL,
    ADD PRIMARY KEY (`id`);

CREATE INDEX `cidadaos_excluidoEm_idx` ON `cidadaos`(`excluidoEm`);
```

> [!CAUTION]
> Leia a primeira linha de novo: `DROP PRIMARY KEY`. Trocar o tipo da chave primária obriga o
> banco a derrubá-la e recriá-la, e o próprio Prisma avisa que, se isso falhar no meio, a tabela
> pode ficar **sem chave primária**. Em banco de trabalho é um susto; em produção é o tipo de
> migration que se faz com backup na mão e janela combinada.

### Passo 2 — preencha o que já existia

```sql
UPDATE cidadaos SET dataNascimento = '1990-01-01' WHERE dataNascimento IS NULL;
```

Em produção de verdade este valor não seria inventado: viria de outro sistema, de uma planilha
ou de um mutirão de atendimento. O que **não** dá é deixar nulo e exigir a coluna no passo
seguinte.

### Passo 3 — agora sim, obrigatória

Volte o schema para `DateTime` sem a interrogação, gere a segunda migration e aplique:

```sql
ALTER TABLE `cidadaos` MODIFY `dataNascimento` DATE NOT NULL;
```

Confira a tabela:

```bash
docker compose exec mysql mysql -ucurso_api -p curso_api -e "DESCRIBE cidadaos;"
```

```
Field           Type          Null  Key
id              char(36)      NO    PRI
cpf             char(11)      NO    UNI
nomeCompleto    varchar(150)  NO
dataNascimento  date          NO
excluidoEm      datetime(3)   YES   MUL
...
```

`PRI`, `UNI` e `MUL` são a chave primária, o índice único e o índice comum. As três garantias
que você declarou no schema, agora existindo no banco.

---

## 🔢 Capítulo 3: validando CPF de verdade

Sem conferência, o cadastro aceita `12345678900` e qualquer erro de digitação vira um cidadão
que ninguém encontra depois.

O CPF tem onze dígitos, e **os dois últimos são calculados a partir dos nove primeiros**. Dá
para refazer a conta e detectar quase todo erro de digitação sem consultar ninguém.

> [!IMPORTANT]
> **O que essa conta prova, e o que ela não prova.** Ela prova que o número é _bem formado_.
> Ela **não** prova que o CPF existe, nem que pertence a quem está cadastrando. Confundir as
> duas coisas é o erro conceitual clássico do assunto — da mesma família do Helmet, que
> **pede** ao navegador (Aula 09), e do `unhealthy`, que é **rótulo** e não ação (Aula 10).

Crie o arquivo `src/shared/validation/cpf.ts` (conteúdo completo no Capítulo 12). Dois detalhes
merecem atenção:

**O resto 10 vale 0.** É o único caso especial da regra oficial. Sem essa linha, a validação
recusa CPFs legítimos — e só os poucos que caem nesse caso, o que torna o defeito difícil de
notar em teste superficial.

**Sequência de um dígito só é recusada à parte.** `11111111111` e `00000000000` **passam** na
conta dos dígitos verificadores, por coincidência matemática. Nenhuma delas é um CPF de
verdade, e a recusa precisa vir escrita:

```ts
if (/^(\d)\1+$/.test(cpf)) {
  return false
}
```

---

## 📜 Capítulo 4: dois contratos de resposta, e o motivo

Aqui vem a decisão mais importante da aula, e ela não é técnica.

A rota de listagem devolve os cidadãos cadastrados. **Ela deve devolver o CPF de cada um?**

A resposta é não, e o motivo é o **tamanho do estrago**: uma listagem exposta por engano
entrega o documento de todo mundo de uma vez; uma consulta individual entrega um. E devolver
CPF numa lista de nomes não resolve nada que a consulta individual não resolva.

Então são dois schemas:

| Schema                 | Onde é usado        | Tem CPF? |
| :--------------------- | :------------------ | :------- |
| `cidadaoResumoSchema`  | `GET /cidadaos`     | **Não**  |
| `cidadaoDetalheSchema` | `GET /cidadaos/:id` | Sim      |

E aqui a Aula 07 é paga com juros: **isso não é uma recomendação a quem escreve o controller.**
O serializador do Fastify monta a resposta usando **apenas** o que está declarado no schema. O
controller pode devolver o registro inteiro do banco que o CPF continua de fora — porque não
existe no schema da listagem.

> **Analogia:** o schema não é uma revisão do que você escreveu. É uma peneira na saída. O que
> não está no desenho da peneira não passa, mesmo que alguém empurre.

As colunas de auditoria (`criadoPor`, `atualizadoPor`) não aparecem em **nenhum** dos dois: são
informação de operação, como o `uptime` da rota de prontidão.

### A ordem que importa no CPF

```ts
const cpfSchema = z
  .string()
  .transform(apenasDigitos)
  .refine(ehCpfValido, { message: 'CPF inválido. Confira os dígitos.' })
```

O `transform` roda **antes** do `refine`. É isso que faz quem manda `529.982.247-25` receber o
mesmo tratamento de quem manda `52998224725`. Invertendo a ordem, a validação receberia o texto
com pontuação e recusaria os dois.

### Por que `PATCH`, e não `PUT`

O cadastro tem dezoito campos. Obrigar quem quer alterar o telefone a reenviar os dezoito é
convite a **apagar dado por omissão** — o campo que a outra ponta esqueceu de mandar vira nulo.

E o `PATCH` recusa corpo vazio:

```ts
.refine((corpo) => Object.keys(corpo).length > 0, {
  message: 'Envie ao menos um campo para alterar.',
})
```

Sem essa linha, `PATCH` com `{}` responderia **200** sem ter feito nada: sucesso para alguém
que, do outro lado, achou que tinha alterado alguma coisa.

**O CPF não pode ser alterado.** Trocar o CPF de um cadastro não é corrigir um dado: é dizer que
aquela linha passou a ser outra pessoa. Quem errou o CPF exclui e cadastra de novo — e a trilha
de auditoria registra as duas coisas.

---

## 🗄️ Capítulo 5: exclusão lógica no repository e no service

No repository, a condição de "ainda ativo" vira uma constante:

```ts
const SOMENTE_ATIVOS = { excluidoEm: null }
```

Ela existe como constante, e não repetida em cada método, porque esquecê-la em **um** lugar faz
cadastro excluído reaparecer — e esse é o tipo de defeito que só é notado por quem pediu a
exclusão.

### A consulta que ignora a exclusão, de propósito

`buscarPorCpf` é a única que **não** filtra por ativos. O motivo é a coluna `@unique`: o CPF de
quem foi excluído continua ocupando a linha no banco.

Se essa busca filtrasse por ativos, a regra de duplicidade diria "pode cadastrar" e o banco
recusaria em seguida — com uma mensagem escrita para desenvolvedor, que é exatamente o que a
Aula 06 proibiu de sair para quem chama a API.

### Duas mensagens de 409, porque são duas situações

```ts
if (jaCadastrado.excluidoEm !== null) {
  throw new AppError(
    'Este CPF pertence a um cadastro excluído. Reative o cadastro existente em vez de criar outro.',
    409,
  )
}

throw new AppError('Já existe um cidadão cadastrado com este CPF.', 409)
```

A ação certa é diferente nos dois casos, então a mensagem é diferente. Dizer "CPF já cadastrado"
sobre um cadastro que a pessoa **não enxerga na listagem** parece defeito do sistema, e o
atendente vai gastar meia hora procurando o que não está lá.

### E a mesma mensagem para dois casos, também de propósito

```ts
if (cidadao === null) {
  throw new AppError('Cidadão não encontrado.', 404)
}
```

"Nunca existiu" e "foi excluído" respondem **igual**. Distinguir os dois contaria a quem
perguntou que aquele identificador já existiu um dia — informação que ele não tinha antes de
perguntar.

---

## 🎛️ Capítulo 6: o controller cuida da apresentação

O controller continua sem nenhuma regra de negócio. O que ele faz de próprio é **transformar o
registro do banco no formato que a versão 1 promete**:

```ts
function formatarData(data: Date): string {
  return data.toISOString().slice(0, 10)
}
```

Por que isso está no controller, e não no service? Porque `2026-08-18` é uma decisão de
**contrato de API**. Uma `v2` poderia escrever a mesma data de outro jeito sem que nenhuma regra
de negócio mudasse.

E repare no 201:

```ts
return reply
  .status(201)
  .header('Location', `/api/v1/cidadaos/${cidadao.id}`)
  .send(montarDetalhe(cidadao))
```

**201**, e não 200: alguma coisa passou a existir do outro lado. O cabeçalho `Location` responde
a pergunta seguinte de quem acabou de cadastrar — _onde foi parar?_ — antes de ela ser feita.

O `DELETE` responde **204**, sem corpo. Devolver o cadastro excluído seria contraditório:
acabamos de dizer que ele não deve mais aparecer nas consultas.

---

## 🔀 Capítulo 7: `/api/v1` — a mecânica é uma linha, a decisão não

No `app.ts`:

```ts
app.register(healthRoutes)
app.register(cidadaoRoutes, { prefix: PREFIXO_DA_API })
```

É isso. Se a aula parasse aqui, teria ensinado a digitar uma linha.

### Por que `/health` fica de fora

Repare que `healthRoutes` é registrada **sem** prefixo. Isso é decisão, não esquecimento.

Quem consulta `/health` é o monitoramento e o `HEALTHCHECK` do container que você escreveu na
Aula 10. Nenhum dos dois é um integrador. Se o alarme apontasse para `/api/v1/health`, o dia em
que a API subisse para a `v2` **derrubaria o monitoramento junto** — por uma mudança que não
tem nada a ver com ele.

`/health` é infraestrutura de operação. `/api/v1/cidadaos` é superfície de API. Coisas
diferentes, endereços diferentes.

### O que obriga a subir de versão

| Mudança                                           | Sobe versão? | Por quê                                            |
| :------------------------------------------------ | :----------: | :------------------------------------------------- |
| Acrescentar campo **opcional** na resposta        |    ❌ Não    | Quem já consome ignora o que não conhece           |
| Acrescentar rota nova                             |    ❌ Não    | Ninguém deixa de funcionar por existir algo a mais |
| Tornar **opcional** um campo que era obrigatório  |    ❌ Não    | Quem já mandava continua podendo mandar            |
| Corrigir defeito que fazia a API responder errado |    ❌ Não    | Voltar ao contrato prometido não é quebrá-lo       |
| **Remover** ou **renomear** campo da resposta     |    ✅ Sim    | Quebra quem lê aquele campo, hoje, em produção     |
| Tornar **obrigatório** um campo que era opcional  |    ✅ Sim    | Quebra quem não manda                              |
| **Apertar** validação, aceitando menos que antes  |    ✅ Sim    | Requisição que passava para de passar              |
| Mudar o **significado** de um status (201 → 202)  |    ✅ Sim    | Muda o que a outra ponta deve fazer a seguir       |

A regra que governa a tabela inteira, em uma frase:

> **É retrocompatível o que não obriga ninguém a mexer no código de quem consome.**

Compare com a Aula 14: lá a pergunta era _"o código velho continua funcionando sobre o schema
novo?"_. Aqui é _"o cliente velho continua funcionando contra a API nova?"_. É a **mesma
pergunta**, feita sobre HTTP em vez de sobre coluna de banco.

### Por quanto tempo a `v1` fica no ar

**Seis meses a partir do anúncio**, e o anúncio acontece no dia em que a `v2` sobe. Durante esse
prazo, a `v1` responde com dois cabeçalhos padronizados:

```
Deprecation: true
Sunset: Sat, 18 Feb 2027 00:00:00 GMT
```

Eles existem para que a informação chegue a quem integra **sem depender de alguém ler um
e-mail**. É a mesma ideia do `.env.example` da Aula 04: o aviso mora onde a pessoa já está
olhando.

Entre órgãos públicos, prazo curto não funciona: o sistema do outro lado tem o próprio
calendário, a própria fila e o próprio orçamento.

### Como duas versões convivem sem virar cópia e cola

A regra: **a versão vive na borda, nunca no miolo.**

```
src/modules/cidadao/
├── cidadao.service.ts        ← uma só, compartilhada pelas versões
├── cidadao.repository.ts     ← uma só
└── v1/
    ├── cidadao.routes.ts     ← o contrato HTTP da v1
    ├── cidadao.controller.ts
    └── cidadao.schema.ts
```

No dia da `v2`, nasce uma pasta `v2/` com os três arquivos dela, registrada com outro prefixo. O
service e o repository **não são duplicados**: a regra de negócio é a mesma para todas as
versões. O que muda de uma para outra é o formato do que entra e do que sai.

Duplicar o service é o erro clássico: a regra passa a existir em dois lugares, e a correção
feita em um não chega no outro.

> [!NOTE]
> **Não crie a pasta `v2/` hoje.** Ela não tem nada dentro, e criar estrutura "para o futuro" é
> exatamente o que a regra de progressão estrita deste curso proíbe. Ela nasce no dia em que
> houver uma mudança incompatível de verdade.

---

## 🚨 Capítulo 8: o contrato de erro na documentação

A API tem **um** formato de erro desde a Aula 06. Mas até agora ele não aparecia na
documentação: a página descrevia só as respostas de sucesso, e quem fosse integrar descobriria
o formato do erro errando.

O caminho óbvio seria declarar `response: { 404: ... }` em cada rota. **Ele não serve**, por dois
motivos independentes:

1. **Passaria o erro pelo serializador.** O Fastify monta a resposta usando o schema declarado —
   e se o corpo do erro divergisse dele em qualquer detalhe, a serialização falharia e o cliente
   receberia **500 no lugar do 400**. A resposta de erro é justamente a que não pode falhar.
2. **Não cobriria o 404 de endereço inexistente.** Ele acontece _sem rota_, no
   `notFoundHandler`, e portanto não tem schema onde ser declarado.

A saída é descrever o erro **só na documentação**, sem que ele encoste no caminho da resposta:
um componente reutilizável do OpenAPI, acrescentado depois que a especificação já está pronta.

### O que é derivado e o que é declarado

O arquivo `src/shared/docs/erro.ts` **deriva** o que consegue, lendo a própria operação:

- tem corpo ou parâmetro → pode falhar a validação → **400**;
- tem parâmetro no caminho → pode apontar para algo que não existe → **404**;
- toda rota → **429** (limite de requisições) e **500** (rede de segurança).

Derivar é melhor que manter uma lista à mão, porque lista envelhece em silêncio quando alguém
acrescenta um parâmetro à rota e esquece de atualizá-la.

O que **não** dá para derivar é regra de negócio. Só quem conhece o cadastro sabe que existe
conflito de CPF. Por isso a rota declara:

```ts
schema: {
  // ...
  errosPossiveis: ['409'],
}
```

E a declaração é guardada **por método**: `POST /api/v1/cidadaos` pode dar 409; o `GET` do mesmo
endereço, não. Guardar só pelo endereço faria a listagem prometer um erro que ela nunca devolve.

Com a API rodando, confira em `/documentation`:

```
POST   /api/v1/cidadaos      -> 201 400 409 429 500
GET    /api/v1/cidadaos      -> 200 429 500
GET    /api/v1/cidadaos/{id} -> 200 400 404 429 500
DELETE /api/v1/cidadaos/{id} -> 204 400 404 429 500
GET    /health               -> 200 429 500
```

> [!TIP]
> Repare que `errosPossiveis` **não é** uma chave do OpenAPI nem do Fastify: é invenção deste
> projeto. O Fastify só compila `body`, `querystring`, `params`, `headers` e `response` — toda
> outra chave do `schema` é levada adiante sem ser interpretada. É assim que `summary`,
> `description` e `tags` já funcionavam desde a Aula 08.

---

## 🧪 Capítulo 9: os testes, e um achado que só apareceu ao rodar

A suíte cresce de 87 para **127 testes**. Três deles merecem ser lidos com atenção.

### O teste de ausência

```ts
expect(lista[0]).not.toHaveProperty('cpf')
expect(lista[0]).not.toHaveProperty('criadoPor')
```

Ele não confere o que a resposta **tem**: confere o que ela **não pode ter**. É o mesmo tipo de
teste que a Aula 07 introduziu, e é o único que percebe se alguém acrescentar o CPF ao schema da
listagem sem pensar duas vezes.

### O teste que protege uma decisão

```ts
expect((await app.inject({ method: 'GET', url: '/health' })).statusCode).toBe(200)
expect((await app.inject({ method: 'GET', url: '/api/v1/health' })).statusCode).toBe(404)
```

Este teste existe para impedir que alguém, um dia, "organize" as rotas movendo o `/health` para
dentro do prefixo. Decisão escrita em comentário é intenção; decisão com teste é regra.

### O teste que consulta o banco por fora

```ts
expect(await prisma.cidadao.findUnique({ where: { id } })).not.toBeNull()
```

Para provar que a exclusão foi **lógica**, o teste consulta o banco direto, e não pelo
repository. Perguntar ao repositório se ele apagou seria confiar na palavra de quem está sendo
testado.

### O achado: um arquivo de teste por vez

Ao rodar a suíte pela primeira vez, **10 testes falharam** — e não nas mesmas linhas a cada
execução. CPF repetido sendo aceito. Registro sumindo entre gravar e consultar.

Nenhum deles era defeito do código sob teste.

A causa: o Vitest roda arquivos de teste **em paralelo**. Até a Aula 14 isso era só ganho de
tempo, porque um único arquivo tocava o banco. Agora são dois — o do service e o das rotas — e
os dois começam cada teste limpando a tabela. Contra o **mesmo** banco de teste, um apagava o
dado que o outro acabara de gravar.

A correção, no `vitest.config.ts`:

```ts
fileParallelism: false,
```

> [!IMPORTANT]
> **A lição não é a linha, é o diagnóstico.** Teste que falha em lugar diferente a cada execução
> quase nunca é defeito do código: é estado compartilhado. E a reação errada — rodar de novo até
> passar — é o hábito que destrói a confiança em qualquer suíte.

---

## 📄 Capítulo 10: atualizando o README

Quem acrescenta rota, atualiza o README. É a regra que a Aula 03 registrou, e ela vale aqui mais
do que nunca: o README passa a ter a tabela de rotas com o prefixo, a seção de versionamento com
a tabela do que obriga a subir de versão, e a seção do cadastro com as decisões sobre dado
pessoal.

O arquivo completo está no Capítulo 12.

---

## 💾 Capítulo 11: fechando o ciclo — mande para o GitHub

```bash
git add .
git commit -m "feat: adiciona o cadastro de cidadao sob /api/v1"
git push
```

Antes do commit, o de sempre:

```bash
npm run check
```

---

## 📚 Capítulo 12: os arquivos, por inteiro

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

/// Cadastro de cidadão.
///
/// Cada bloco de campos existe por um motivo diferente, e os comentários dizem
/// qual — inclusive nos casos em que a escolha foi recusar algo mais flexível.
///
/// Três decisões valem por todo o resto:
///
///   • **Quase tudo é opcional.** Atendimento não pode travar porque falta o
///     complemento do endereço. Só identificam a pessoa, e por isso são
///     obrigatórios: CPF, nome completo e data de nascimento.
///   • **Ninguém é apagado.** Cadastro de cidadão tem histórico ligado a ele, e
///     `DELETE` não tem desfazer. A exclusão preenche `excluidoEm`.
///   • **Os tipos são declarados.** O padrão do Prisma para texto é
///     `VARCHAR(191)`, que serve para tudo e não cabe em nada: CPF tem 11
///     dígitos, sigla de UF tem 2, e um índice sobre coluna larga é mais caro
///     sem ser mais útil.
model Cidadao {
  // ── Identificação ─────────────────────────────────────────────────────────
  // O uuid tem 36 caracteres, sempre. `Char(36)` deixa a chave primária — que
  // é o índice mais consultado da tabela — do tamanho exato do que guarda.
  id String @id @default(uuid()) @db.Char(36)

  // Guardado apenas com dígitos, sem ponto nem traço. Formatar é trabalho de
  // quem exibe; guardar formatado impede comparar e deixa o mesmo CPF entrar
  // duas vezes em grafias diferentes.
  cpf String @unique @db.Char(11)

  // O nome de registro, o que consta no documento.
  nomeCompleto String @db.VarChar(150)

  // Nome social é direito, não apelido: é por ele que a pessoa deve ser
  // tratada. Fica AO LADO do nome de registro, e não no lugar dele, porque
  // documento oficial exige o de registro — trocar um pelo outro resolveria um
  // requisito quebrando o outro.
  nomeSocial String? @db.VarChar(150)

  // Sem hora, de propósito: `@db.Date` em vez de `DateTime` completo. Data de
  // nascimento guardada com hora vira bug de fuso — quem nasceu no dia 1º
  // aparece como dia 31 do mês anterior em outro fuso horário.
  dataNascimento DateTime @db.Date

  // ── Documento de identidade ───────────────────────────────────────────────
  // Opcional porque nem todo atendimento exige RG, e porque quem tem CPF já
  // está identificado. O órgão emissor acompanha o número: RG é estadual, e o
  // mesmo número existe em unidades federativas diferentes.
  rg           String? @db.VarChar(20)
  orgaoEmissor String? @db.VarChar(20)
  ufEmissor    String? @db.Char(2)

  // ── Contato ───────────────────────────────────────────────────────────────
  // 254 é o tamanho máximo de um endereço de e-mail pela especificação.
  email String? @db.VarChar(254)

  // Só dígitos, com DDD: 11 caracteres cobrem celular, 10 cobrem fixo. Sem
  // `@unique`: duas pessoas da mesma casa dividem telefone, e a Aula 14 mostrou
  // o estrago de descobrir isso depois de o índice já estar em produção.
  telefone String? @db.VarChar(11)

  // ── Endereço ──────────────────────────────────────────────────────────────
  // Em campos separados, e não em um texto livre só. Endereço inteiro em uma
  // coluna é impossível de consultar por bairro ou por município — que é
  // exatamente o que um órgão público precisa perguntar.
  cep         String? @db.Char(8)
  logradouro  String? @db.VarChar(150)
  numero      String? @db.VarChar(10)
  complemento String? @db.VarChar(60)
  bairro      String? @db.VarChar(80)
  municipio   String? @db.VarChar(80)
  uf          String? @db.Char(2)

  // ── Trilha de auditoria ───────────────────────────────────────────────────
  // Quem criou e quem alterou por último. As duas colunas nascem NULAS porque o
  // projeto ainda não tem login: não há de quem tirar o nome. Elas existem
  // desde já mesmo assim, e o motivo é a Aula 14 — acrescentar coluna a uma
  // tabela que já tem dado de gente dentro é caro, e fazer isso agora custa
  // zero. Quem as preenche é a aula de autenticação.
  criadoEm      DateTime  @default(now())
  criadoPor     String?   @db.VarChar(60)
  atualizadoEm  DateTime  @updatedAt
  atualizadoPor String?   @db.VarChar(60)

  // ── Exclusão lógica ───────────────────────────────────────────────────────
  // Preenchido, significa "excluído": o cadastro some das consultas e continua
  // no banco. O CPF continua ocupado — a operação certa para quem volta é
  // reativar o cadastro que já existe, e não criar um segundo.
  excluidoEm  DateTime?
  excluidoPor String?   @db.VarChar(60)

  // Toda listagem filtra por "não excluído". Filtro que roda em toda consulta
  // merece índice.
  @@index([excluidoEm])
  @@map("cidadaos")
}
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
  {
    nomeCompleto: 'Maria Aparecida Souza',
    cpf: '00000000191',
    dataNascimento: new Date('1985-03-12'),
    email: 'maria@exemplo.gov.br',
    municipio: 'Itapetininga',
    uf: 'SP',
  },
  {
    nomeCompleto: 'João Batista Lima',
    // Nome social ao lado do de registro, e não no lugar dele: é assim que a
    // pessoa deve ser tratada, sem que o cadastro perca o que consta no documento.
    nomeSocial: 'Joana Batista Lima',
    cpf: '00000000272',
    dataNascimento: new Date('1992-11-30'),
    email: null,
    municipio: 'Sorocaba',
    uf: 'SP',
  },
  {
    nomeCompleto: 'Ana Clara Dias',
    cpf: '00000000353',
    dataNascimento: new Date('2001-07-04'),
    email: 'ana@exemplo.gov.br',
    municipio: 'Itapetininga',
    uf: 'SP',
  },
]

async function semear(): Promise<void> {
  for (const cidadao of CIDADAOS) {
    const { cpf, ...camposAtualizaveis } = cidadao

    await prisma.cidadao.upsert({
      // O CPF é a chave natural do cadastro: é por ele que se sabe se aquela
      // pessoa já está no banco. Usar o `id` aqui não serviria, porque ele é
      // gerado no momento da criação e ninguém o conhece de antemão.
      where: { cpf },
      update: camposAtualizaveis,
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

### `src/shared/validation/cpf.ts`

```ts
/**
 * Validação de CPF
 *
 * O CPF tem onze dígitos, e os **dois últimos são calculados a partir dos nove
 * primeiros**. Isso significa que um número digitado errado quase sempre pode ser
 * detectado sem consultar ninguém: basta refazer a conta.
 *
 * O que esta validação prova, e o que ela NÃO prova:
 *
 *   ✅ que o número é **bem formado** — foi digitado sem troca de algarismos;
 *   ❌ que o CPF **existe**;
 *   ❌ que ele pertence à pessoa que está cadastrando.
 *
 * Confundir as duas coisas é o erro conceitual clássico do assunto. Quem quiser a
 * segunda garantia precisa consultar a base da Receita Federal, que é integração
 * externa — outro assunto, com outro custo e outra falha possível.
 *
 * Mesmo assim vale muito: sem esta conferência, o cadastro aceita `12345678900`
 * e qualquer erro de digitação vira um cidadão que ninguém encontra depois.
 */

/** Quantidade de dígitos de um CPF, sem pontuação. */
const TAMANHO_DO_CPF = 11

/**
 * Calcula um dígito verificador do CPF.
 *
 * Os dois dígitos usam o mesmo algoritmo, mudando só quantos dígitos entram na
 * conta: o primeiro olha os 9 iniciais, o segundo olha os 10 (já com o primeiro
 * dígito calculado). Por isso a função recebe a quantidade como parâmetro, em vez
 * de existir duas vezes quase igual.
 *
 * @param digitos    Os dígitos do CPF, já separados em números.
 * @param quantidade Quantos dígitos entram no cálculo (9 ou 10).
 * @returns          O dígito verificador esperado, de 0 a 9.
 */
function calcularDigito(digitos: number[], quantidade: number): number {
  // Cada posição tem um peso, que começa em `quantidade + 1` e cai de um em um.
  // Para o primeiro dígito os pesos vão de 10 a 2; para o segundo, de 11 a 2.
  let soma = 0

  for (let posicao = 0; posicao < quantidade; posicao++) {
    soma += (digitos[posicao] ?? 0) * (quantidade + 1 - posicao)
  }

  const resto = (soma * 10) % 11

  // O resto 10 é o único caso especial da regra oficial: ele vale 0. Sem esta
  // linha, a validação recusaria CPFs legítimos — e só os poucos que caem neste
  // caso, o que torna o defeito difícil de perceber em teste superficial.
  return resto === 10 ? 0 : resto
}

/**
 * Verifica se um CPF é bem formado.
 *
 * @param cpf CPF com exatamente 11 dígitos, sem ponto e sem traço.
 * @returns   `true` quando os dois dígitos verificadores conferem.
 *
 * @example `ehCpfValido('00000000191')` devolve `true`
 */
export function ehCpfValido(cpf: string): boolean {
  if (cpf.length !== TAMANHO_DO_CPF || !/^\d+$/.test(cpf)) {
    return false
  }

  // Sequências de um dígito só (`00000000000`, `11111111111`) passam na conta dos
  // dígitos verificadores por coincidência matemática, e nenhuma delas é um CPF
  // de verdade. É a recusa que precisa vir escrita, porque o algoritmo sozinho
  // não a faz.
  if (/^(\d)\1+$/.test(cpf)) {
    return false
  }

  const digitos = [...cpf].map(Number)

  return calcularDigito(digitos, 9) === digitos[9] && calcularDigito(digitos, 10) === digitos[10]
}

/**
 * Remove tudo que não for dígito de um CPF.
 *
 * Quem consome a API pode mandar `529.982.247-25` ou `52998224725`. Guardar as
 * duas formas no banco deixaria o mesmo CPF entrar duas vezes, e a coluna
 * `@unique` não perceberia — para o banco são textos diferentes.
 *
 * @param cpf CPF como veio, com ou sem pontuação.
 * @returns   Somente os dígitos.
 */
export function apenasDigitos(cpf: string): string {
  return cpf.replaceAll(/\D/g, '')
}
```

### `src/shared/validation/cpf.spec.ts`

```ts
/**
 * Testes da validação de CPF
 *
 * Um teste por regra, e não um por campo — o mesmo princípio da suíte das
 * variáveis de ambiente. As regras aqui são quatro: tamanho, só dígitos,
 * sequência repetida e os dois dígitos verificadores.
 *
 * Os CPFs usados são inventados. Alguns fecham a conta e outros não, e é
 * exatamente por isso que servem: material de estudo não usa documento de pessoa
 * real, nem mesmo um que pareça válido.
 */

import { describe, expect, it } from 'vitest'

import { apenasDigitos, ehCpfValido } from './cpf.ts'

describe('ehCpfValido', () => {
  it('aceita um CPF com os dígitos verificadores corretos', () => {
    expect(ehCpfValido('00000000191')).toBe(true)
    expect(ehCpfValido('52998224725')).toBe(true)
  })

  it('recusa quando o último dígito não fecha a conta', () => {
    // Um único algarismo trocado no fim: é o erro de digitação mais comum, e é
    // exatamente o que o dígito verificador existe para pegar.
    expect(ehCpfValido('00000000192')).toBe(false)
  })

  it('recusa quando dois algarismos do meio são trocados de lugar', () => {
    // Troca de posição não muda a soma, mas muda os pesos — por isso o cálculo
    // usa peso decrescente, e não uma soma simples.
    expect(ehCpfValido('52998242725')).toBe(false)
  })

  it('recusa sequência de um dígito só', () => {
    // Estas passam na conta dos dígitos verificadores por coincidência
    // matemática, e nenhuma delas é um CPF de verdade. Sem a recusa explícita,
    // `11111111111` entraria no cadastro.
    expect(ehCpfValido('00000000000')).toBe(false)
    expect(ehCpfValido('11111111111')).toBe(false)
    expect(ehCpfValido('99999999999')).toBe(false)
  })

  it('recusa tamanho diferente de onze', () => {
    expect(ehCpfValido('5299822472')).toBe(false)
    expect(ehCpfValido('529982247250')).toBe(false)
    expect(ehCpfValido('')).toBe(false)
  })

  it('recusa o que não for só dígito', () => {
    // A pontuação é removida antes, pelo schema. Esta função recebe o número já
    // limpo, e recusar aqui é o que impede um caminho novo de esquecer a limpeza.
    expect(ehCpfValido('529.982.247-25')).toBe(false)
    expect(ehCpfValido('5299822472a')).toBe(false)
  })
})

describe('apenasDigitos', () => {
  it('remove ponto e traço', () => {
    expect(apenasDigitos('529.982.247-25')).toBe('52998224725')
  })

  it('não altera um número que já vem limpo', () => {
    expect(apenasDigitos('52998224725')).toBe('52998224725')
  })
})
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
 *
 * Uma decisão atravessa o arquivo inteiro: **ninguém é apagado**. A exclusão
 * preenche `excluidoEm`, e toda consulta filtra por quem ainda está ativo. O
 * cadastro de um cidadão tem histórico ligado a ele — protocolo, atendimento,
 * requisição — e `DELETE` transformaria esse histórico em referência quebrada,
 * sem desfazer possível.
 */

import { prisma } from '../../shared/database/index.ts'
// O tipo do modelo gerado pelo Prisma 7 leva o sufixo `Model`. Em material
// escrito para versoes anteriores ele aparece so como `Cidadao`.
import type { CidadaoModel } from '../../generated/prisma/models.ts'

/**
 * Dados aceitos no cadastro de um cidadão.
 *
 * Só três campos são obrigatórios, e são os que identificam a pessoa. O resto é
 * opcional porque um atendimento não pode ser recusado por falta de complemento
 * de endereço.
 */
export interface DadosDoNovoCidadao {
  cpf: string
  nomeCompleto: string
  dataNascimento: Date
  nomeSocial?: string | undefined
  rg?: string | undefined
  orgaoEmissor?: string | undefined
  ufEmissor?: string | undefined
  email?: string | undefined
  telefone?: string | undefined
  cep?: string | undefined
  logradouro?: string | undefined
  numero?: string | undefined
  complemento?: string | undefined
  bairro?: string | undefined
  municipio?: string | undefined
  uf?: string | undefined
}

/**
 * Campos que podem ser alterados depois do cadastro.
 *
 * O CPF fica de fora de propósito: trocá-lo não é alterar um dado, é dizer que
 * aquela linha passou a ser outra pessoa.
 */
export type DadosDeAtualizacao = Partial<Omit<DadosDoNovoCidadao, 'cpf'>>

/**
 * Condição de "ainda ativo", usada em toda consulta de leitura.
 *
 * Existe como constante, e não repetida em cada método, porque esquecê-la em um
 * único lugar faz cadastro excluído reaparecer — e esse é o tipo de defeito que
 * só é notado por quem pediu a exclusão.
 */
const SOMENTE_ATIVOS = { excluidoEm: null }

export class CidadaoRepository {
  /**
   * Grava um cidadão novo.
   *
   * @param dados Os dados do cadastro, já validados por quem chamou.
   * @returns     O registro gravado, já com `id` e datas preenchidos pelo banco.
   */
  async criar(dados: DadosDoNovoCidadao): Promise<CidadaoModel> {
    return prisma.cidadao.create({ data: dados })
  }

  /**
   * Busca um cidadão ativo pelo identificador interno.
   *
   * @param id Identificador gerado no cadastro.
   * @returns  O cidadão, ou `null` quando não existe ou foi excluído. Devolver
   *           `null` é resposta legítima do repositório: decidir se isso é erro
   *           cabe a quem chamou.
   */
  async buscarPorId(id: string): Promise<CidadaoModel | null> {
    return prisma.cidadao.findFirst({ where: { id, ...SOMENTE_ATIVOS } })
  }

  /**
   * Busca um cidadão pelo CPF, **incluindo os excluídos**.
   *
   * É a única consulta que ignora a exclusão lógica, e o motivo é a coluna
   * `@unique`: o CPF de quem foi excluído continua ocupando a linha no banco.
   * Se esta busca filtrasse por ativos, a regra de duplicidade diria "pode
   * cadastrar" e o banco recusaria em seguida — com uma mensagem escrita para
   * desenvolvedor, e não para quem está no balcão.
   *
   * @param cpf CPF, apenas dígitos.
   * @returns   O cidadão, ativo ou excluído, ou `null`.
   */
  async buscarPorCpf(cpf: string): Promise<CidadaoModel | null> {
    return prisma.cidadao.findUnique({ where: { cpf } })
  }

  /**
   * Lista os cidadãos ativos, do mais recente para o mais antigo.
   *
   * @returns Todos os registros não excluídos. Paginação entra quando existir
   *          volume que a justifique — hoje seria configuração para uma dor que
   *          não existe.
   */
  async listar(): Promise<CidadaoModel[]> {
    return prisma.cidadao.findMany({
      where: SOMENTE_ATIVOS,
      orderBy: { criadoEm: 'desc' },
    })
  }

  /**
   * Altera os campos enviados de um cadastro, e só eles.
   *
   * @param id    Identificador do cidadão.
   * @param dados Apenas os campos que devem mudar.
   * @returns     O registro já alterado.
   */
  async atualizar(id: string, dados: DadosDeAtualizacao): Promise<CidadaoModel> {
    return prisma.cidadao.update({ where: { id }, data: dados })
  }

  /**
   * Marca um cadastro como excluído, sem apagar a linha.
   *
   * @param id Identificador do cidadão.
   * @returns  O registro, já com a data de exclusão preenchida.
   */
  async excluir(id: string): Promise<CidadaoModel> {
    return prisma.cidadao.update({
      where: { id },
      data: { excluidoEm: new Date() },
    })
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
 *
 * Este service é **um só para todas as versões da API**. O que muda de uma versão
 * para outra é o formato do que entra e do que sai, que mora nos schemas de cada
 * `v*`. Duplicar a regra aqui criaria dois lugares para corrigir o mesmo defeito,
 * e a correção feita em um não chegaria no outro.
 */

import { AppError } from '../../shared/errors/app-error.ts'
import type { CidadaoModel } from '../../generated/prisma/models.ts'
import type {
  CidadaoRepository,
  DadosDeAtualizacao,
  DadosDoNovoCidadao,
} from './cidadao.repository.ts'

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
   * @param dados Os dados do cadastro, já validados na entrada.
   * @throws {AppError} 409, quando o CPF já está cadastrado — ativo ou excluído.
   * @returns O cidadão recém-criado.
   */
  async cadastrar(dados: DadosDoNovoCidadao): Promise<CidadaoModel> {
    const jaCadastrado = await this.repository.buscarPorCpf(dados.cpf)

    if (jaCadastrado !== null) {
      // 409 (conflito) e não 400: a requisição está correta em forma e conteúdo.
      // O que impede o cadastro é o estado atual do sistema, não um erro de quem
      // chamou — e essa diferença muda o que a outra ponta deve fazer a respeito.
      //
      // A mensagem muda conforme o cadastro esteja ativo ou excluído, porque a
      // ação certa é diferente nos dois casos. "CPF já cadastrado" para um
      // cadastro que a pessoa não enxerga na listagem parece defeito do sistema.
      if (jaCadastrado.excluidoEm !== null) {
        throw new AppError(
          'Este CPF pertence a um cadastro excluído. Reative o cadastro existente em vez de criar outro.',
          409,
        )
      }

      throw new AppError('Já existe um cidadão cadastrado com este CPF.', 409)
    }

    return this.repository.criar(dados)
  }

  /**
   * Busca um cidadão ativo pelo identificador.
   *
   * @param id Identificador gerado no cadastro.
   * @throws {AppError} 404, quando não existe cidadão ativo com aquele id.
   * @returns O cidadão encontrado.
   */
  async buscarPorId(id: string): Promise<CidadaoModel> {
    const cidadao = await this.repository.buscarPorId(id)

    if (cidadao === null) {
      // A mesma resposta para "nunca existiu" e para "foi excluído", de
      // propósito. Distinguir os dois casos contaria a quem perguntou que aquele
      // identificador já existiu um dia — informação que ele não tinha antes de
      // perguntar.
      throw new AppError('Cidadão não encontrado.', 404)
    }

    return cidadao
  }

  /**
   * Lista os cidadãos ativos.
   *
   * @returns Lista, do cadastro mais recente para o mais antigo.
   */
  async listar(): Promise<CidadaoModel[]> {
    return this.repository.listar()
  }

  /**
   * Altera os campos enviados de um cadastro ativo.
   *
   * @param id    Identificador do cidadão.
   * @param dados Apenas os campos que devem mudar.
   * @throws {AppError} 404, quando não existe cidadão ativo com aquele id.
   * @returns O cadastro já alterado.
   */
  async atualizar(id: string, dados: DadosDeAtualizacao): Promise<CidadaoModel> {
    // A busca antes da alteração existe para que a resposta seja 404, e não o
    // erro que o Prisma levanta quando o `update` não encontra a linha. Ela
    // também é o que faz a exclusão lógica valer para a alteração: quem foi
    // excluído não é encontrado aqui, e portanto não pode ser alterado.
    await this.buscarPorId(id)

    return this.repository.atualizar(id, dados)
  }

  /**
   * Exclui um cadastro — logicamente, preenchendo a data de exclusão.
   *
   * @param id Identificador do cidadão.
   * @throws {AppError} 404, quando não existe cidadão ativo com aquele id.
   */
  async excluir(id: string): Promise<void> {
    await this.buscarPorId(id)

    await this.repository.excluir(id)
  }
}
```

### `src/modules/cidadao/v1/cidadao.schema.ts`

```ts
/**
 * Contratos de entrada e de saída do cadastro de cidadão — versão 1
 *
 * Este arquivo mora dentro de `v1/` de propósito. O que muda entre duas versões
 * de uma API é justamente isto: **o formato do que entra e do que sai**. A regra
 * de negócio, que vive no service, é a mesma para todas as versões — duplicá-la
 * seria criar dois lugares para corrigir o mesmo defeito.
 *
 * Há dois schemas de resposta, e a diferença entre eles é uma decisão sobre dado
 * pessoal:
 *
 *   • `cidadaoResumoSchema`  — usado na listagem. **Não tem CPF.**
 *   • `cidadaoDetalheSchema` — usado na consulta por id. Tem o cadastro inteiro.
 *
 * O motivo é de tamanho do estrago: uma listagem exposta por engano entrega o CPF
 * de todo mundo de uma vez; uma consulta individual entrega um. Devolver CPF numa
 * lista de nomes não serve para nada que a consulta individual não resolva — e
 * essa é a pergunta certa a fazer sobre cada campo de resposta.
 *
 * As colunas de auditoria (`criadoPor`, `atualizadoPor`) não aparecem em nenhum
 * dos dois: são informação de dentro de casa, como o `uptime` da rota de
 * prontidão. Quem precisa delas consulta o banco, não a API.
 */

import { z } from 'zod'
import { apenasDigitos, ehCpfValido } from '../../../shared/validation/cpf.ts'

/** As 27 unidades federativas, usadas no endereço e no órgão emissor do RG. */
const UFS = [
  'AC',
  'AL',
  'AP',
  'AM',
  'BA',
  'CE',
  'DF',
  'ES',
  'GO',
  'MA',
  'MT',
  'MS',
  'MG',
  'PA',
  'PB',
  'PR',
  'PE',
  'PI',
  'RJ',
  'RN',
  'RS',
  'RO',
  'RR',
  'SC',
  'SP',
  'SE',
  'TO',
] as const

/** Data mais antiga aceita como nascimento. Ninguém vivo nasceu antes disso. */
const DATA_MINIMA_DE_NASCIMENTO = '1900-01-01'

/**
 * CPF: aceita com ou sem pontuação, guarda só os dígitos, recusa mal formado.
 *
 * A ordem importa. O `transform` roda **antes** do `refine`, então a conferência
 * dos dígitos verificadores sempre recebe o número limpo — e quem manda
 * `529.982.247-25` recebe o mesmo tratamento de quem manda `52998224725`.
 */
const cpfSchema = z
  .string()
  .transform(apenasDigitos)
  .refine(ehCpfValido, { message: 'CPF inválido. Confira os dígitos.' })
  .describe('CPF com 11 dígitos. Pontuação é aceita e descartada.')

/**
 * Data de nascimento: texto `AAAA-MM-DD`, nem no futuro nem impossível.
 *
 * Recusar data futura não é preciosismo: é o erro de digitação mais comum em
 * campo de data, e um cadastro com nascimento em 2087 atravessa qualquer regra
 * que dependa de idade.
 */
const dataDeNascimentoSchema = z.iso
  .date()
  .refine((data) => data <= new Date().toISOString().slice(0, 10), {
    message: 'A data de nascimento não pode estar no futuro.',
  })
  .refine((data) => data >= DATA_MINIMA_DE_NASCIMENTO, {
    message: 'A data de nascimento é antiga demais para ser real.',
  })
  .describe('Data de nascimento, no formato AAAA-MM-DD.')

/** Telefone com DDD, só dígitos: 10 para fixo, 11 para celular. */
const telefoneSchema = z
  .string()
  .transform(apenasDigitos)
  .refine((valor) => valor.length === 10 || valor.length === 11, {
    message: 'Telefone deve ter 10 dígitos (fixo) ou 11 (celular), com DDD.',
  })
  .describe('Telefone com DDD. Pontuação é aceita e descartada.')

/** CEP com 8 dígitos, aceito com ou sem traço. */
const cepSchema = z
  .string()
  .transform(apenasDigitos)
  .refine((valor) => valor.length === 8, { message: 'CEP deve ter 8 dígitos.' })
  .describe('CEP com 8 dígitos. O traço é aceito e descartado.')

/**
 * Campos do endereço, todos opcionais.
 *
 * Ficam em um objeto próprio porque `POST` e `PATCH` usam os mesmos — e porque
 * escrever duas vezes é escrever duas chances de divergir.
 */
const enderecoSchema = {
  cep: cepSchema.optional(),
  logradouro: z.string().max(150).optional().describe('Rua, avenida, praça.'),
  numero: z.string().max(10).optional().describe('Número. É texto: existe "s/n".'),
  complemento: z.string().max(60).optional().describe('Apartamento, bloco, fundos.'),
  bairro: z.string().max(80).optional(),
  municipio: z.string().max(80).optional(),
  uf: z.enum(UFS).optional().describe('Sigla da unidade federativa.'),
}

/** Campos do documento de identidade, todos opcionais. */
const documentoSchema = {
  rg: z.string().max(20).optional().describe('Número do RG, como consta no documento.'),
  orgaoEmissor: z.string().max(20).optional().describe('Órgão que emitiu o RG (ex.: SSP).'),
  ufEmissor: z.enum(UFS).optional().describe('Unidade federativa do órgão emissor.'),
}

/**
 * Corpo de `POST /api/v1/cidadaos`.
 *
 * Três campos são obrigatórios, e são exatamente os que identificam a pessoa:
 * CPF, nome completo e data de nascimento. Todo o resto é opcional, porque
 * atendimento não pode travar por falta de complemento de endereço.
 */
export const criarCidadaoBodySchema = z
  .object({
    cpf: cpfSchema,

    nomeCompleto: z
      .string()
      .trim()
      .min(3, 'O nome completo precisa ter ao menos 3 caracteres.')
      .max(150)
      .describe('Nome como consta no documento de identificação.'),

    nomeSocial: z
      .string()
      .trim()
      .min(3)
      .max(150)
      .optional()
      .describe('Nome pelo qual a pessoa deve ser tratada, quando diferente do de registro.'),

    dataNascimento: dataDeNascimentoSchema,

    ...documentoSchema,

    email: z.email('E-mail inválido.').max(254).optional(),
    telefone: telefoneSchema.optional(),

    ...enderecoSchema,
  })
  .describe('Dados para cadastrar um cidadão.')

/**
 * Corpo de `PATCH /api/v1/cidadaos/:id`.
 *
 * É o schema de criação com tudo opcional, mais **sem o CPF**. Trocar o CPF de um
 * cadastro não é alterar um dado: é dizer que aquela linha passou a ser outra
 * pessoa. Quem errou o CPF no cadastro exclui e cadastra de novo, e a trilha de
 * auditoria registra as duas coisas.
 *
 * O `.refine` no final recusa corpo vazio. Sem ele, `PATCH` com `{}` responderia
 * 200 sem ter feito nada — resposta de sucesso para uma requisição sem efeito.
 */
export const atualizarCidadaoBodySchema = criarCidadaoBodySchema
  .omit({ cpf: true })
  .partial()
  .refine((corpo) => Object.keys(corpo).length > 0, {
    message: 'Envie ao menos um campo para alterar.',
  })
  .describe('Campos a alterar. Só o que for enviado é modificado.')

/** Parâmetro de endereço das rotas que operam sobre um cadastro específico. */
export const cidadaoParamsSchema = z.object({
  id: z.uuid('O identificador precisa ser um uuid.').describe('Identificador do cidadão.'),
})

/**
 * Resposta da listagem — **sem CPF**, por decisão registrada no topo do arquivo.
 *
 * O serializador do Fastify monta a resposta usando apenas o que está declarado
 * aqui. Isso não é uma recomendação a quem escreve o controller: é o que
 * acontece. Um campo esquecido no controller não vaza, porque não existe neste
 * schema.
 */
export const cidadaoResumoSchema = z
  .object({
    id: z.uuid(),
    nomeCompleto: z.string(),
    nomeSocial: z.string().nullable(),
    dataNascimento: z.string().describe('Data de nascimento, no formato AAAA-MM-DD.'),
    municipio: z.string().nullable(),
    uf: z.string().nullable(),
    criadoEm: z.string().describe('Momento do cadastro, no padrão ISO 8601.'),
  })
  .describe('Resumo de um cidadão, como aparece na listagem.')

/** Resposta da consulta individual — o cadastro completo, com CPF. */
export const cidadaoDetalheSchema = cidadaoResumoSchema
  .extend({
    cpf: z.string().describe('CPF com 11 dígitos, sem pontuação.'),
    rg: z.string().nullable(),
    orgaoEmissor: z.string().nullable(),
    ufEmissor: z.string().nullable(),
    email: z.string().nullable(),
    telefone: z.string().nullable(),
    cep: z.string().nullable(),
    logradouro: z.string().nullable(),
    numero: z.string().nullable(),
    complemento: z.string().nullable(),
    bairro: z.string().nullable(),
    atualizadoEm: z.string().describe('Momento da última alteração, no padrão ISO 8601.'),
  })
  .describe('Cadastro completo de um cidadão.')

/** Lista de cidadãos ativos. */
export const cidadaoListaSchema = z
  .array(cidadaoResumoSchema)
  .describe('Cidadãos ativos, do cadastro mais recente para o mais antigo.')

/** Dados aceitos no cadastro, derivados do schema acima. */
export type CriarCidadaoBody = z.infer<typeof criarCidadaoBodySchema>

/** Dados aceitos na alteração, derivados do schema acima. */
export type AtualizarCidadaoBody = z.infer<typeof atualizarCidadaoBodySchema>
```

### `src/modules/cidadao/v1/cidadao.controller.ts`

```ts
/**
 * CidadaoController — versão 1
 *
 * Recebe a requisição, chama o service e devolve a resposta. **Nenhuma regra de
 * negócio mora aqui**: se este arquivo precisar decidir alguma coisa sobre
 * cidadão, a decisão está no lugar errado.
 *
 * O que ele faz de próprio é **apresentação**: transformar o registro do banco no
 * formato que a versão 1 da API promete. É por isso que a conversão de datas
 * acontece aqui, e não no service — `2026-08-18` é uma decisão de contrato de
 * API, e uma `v2` poderia escrever a mesma data de outro jeito sem que a regra de
 * negócio mudasse em nada.
 */

import type { FastifyReply, FastifyRequest } from 'fastify'
import type { CidadaoModel } from '../../../generated/prisma/models.ts'
import type { CidadaoService } from '../cidadao.service.ts'
import type { AtualizarCidadaoBody, CriarCidadaoBody } from './cidadao.schema.ts'

/** Resumo de um cidadão, como sai na listagem. */
interface CidadaoResumo {
  id: string
  nomeCompleto: string
  nomeSocial: string | null
  dataNascimento: string
  municipio: string | null
  uf: string | null
  criadoEm: string
}

/**
 * Converte uma data em `AAAA-MM-DD`.
 *
 * A coluna é do tipo `DATE`, sem hora: o banco devolve a meia-noite em UTC, e
 * cortar os dez primeiros caracteres do padrão ISO devolve exatamente o dia que
 * foi gravado. Formatar com o fuso da máquina é o que produziria o clássico
 * "nasceu dia 1º, aparece dia 31".
 */
function formatarData(data: Date): string {
  return data.toISOString().slice(0, 10)
}

/**
 * Monta o resumo de um cidadão — o formato da listagem, **sem CPF**.
 *
 * @param cidadao O registro como veio do banco.
 */
function montarResumo(cidadao: CidadaoModel): CidadaoResumo {
  return {
    id: cidadao.id,
    nomeCompleto: cidadao.nomeCompleto,
    nomeSocial: cidadao.nomeSocial,
    dataNascimento: formatarData(cidadao.dataNascimento),
    municipio: cidadao.municipio,
    uf: cidadao.uf,
    criadoEm: cidadao.criadoEm.toISOString(),
  }
}

/**
 * Monta o cadastro completo — o formato da consulta individual, com CPF.
 *
 * As colunas de auditoria (`criadoPor`, `atualizadoPor`) ficam de fora: são
 * informação de operação, como o `uptime` da rota de prontidão.
 *
 * @param cidadao O registro como veio do banco.
 */
function montarDetalhe(cidadao: CidadaoModel): Record<string, unknown> {
  return {
    ...montarResumo(cidadao),
    cpf: cidadao.cpf,
    rg: cidadao.rg,
    orgaoEmissor: cidadao.orgaoEmissor,
    ufEmissor: cidadao.ufEmissor,
    email: cidadao.email,
    telefone: cidadao.telefone,
    cep: cidadao.cep,
    logradouro: cidadao.logradouro,
    numero: cidadao.numero,
    complemento: cidadao.complemento,
    bairro: cidadao.bairro,
    atualizadoEm: cidadao.atualizadoEm.toISOString(),
  }
}

export class CidadaoController {
  /**
   * @param service Onde moram as regras de negócio de cidadão.
   */
  constructor(private readonly service: CidadaoService) {}

  /**
   * `POST /api/v1/cidadaos` — cadastra um cidadão.
   *
   * @returns 201 com o cadastro completo, ou 409 quando o CPF já existe.
   */
  async criar(
    request: FastifyRequest<{ Body: CriarCidadaoBody }>,
    reply: FastifyReply,
  ): Promise<FastifyReply> {
    const cidadao = await this.service.cadastrar({
      ...request.body,
      // O schema entrega a data como texto, que é o que trafega em JSON. O banco
      // guarda uma data. A conversão acontece na fronteira, e uma vez só.
      dataNascimento: new Date(request.body.dataNascimento),
    })

    // 201, e não 200: alguma coisa passou a existir do outro lado. O cabeçalho
    // `Location` diz onde ela está, que é a resposta à pergunta seguinte de quem
    // acabou de cadastrar.
    return reply
      .status(201)
      .header('Location', `/api/v1/cidadaos/${cidadao.id}`)
      .send(montarDetalhe(cidadao))
  }

  /**
   * `GET /api/v1/cidadaos` — lista os cadastros ativos.
   *
   * @returns 200 com a lista de resumos.
   */
  async listar(_request: FastifyRequest, reply: FastifyReply): Promise<FastifyReply> {
    const cidadaos = await this.service.listar()

    return reply.status(200).send(cidadaos.map(montarResumo))
  }

  /**
   * `GET /api/v1/cidadaos/:id` — consulta um cadastro.
   *
   * @returns 200 com o cadastro completo, ou 404.
   */
  async buscarPorId(
    request: FastifyRequest<{ Params: { id: string } }>,
    reply: FastifyReply,
  ): Promise<FastifyReply> {
    const cidadao = await this.service.buscarPorId(request.params.id)

    return reply.status(200).send(montarDetalhe(cidadao))
  }

  /**
   * `PATCH /api/v1/cidadaos/:id` — altera os campos enviados.
   *
   * @returns 200 com o cadastro já alterado, ou 404.
   */
  async atualizar(
    request: FastifyRequest<{ Params: { id: string }; Body: AtualizarCidadaoBody }>,
    reply: FastifyReply,
  ): Promise<FastifyReply> {
    const { dataNascimento, ...resto } = request.body

    const cidadao = await this.service.atualizar(request.params.id, {
      ...resto,
      // Só converte se o campo veio. Enviar `undefined` para o Prisma significa
      // "não mexa nesta coluna", que é exatamente o comportamento de um PATCH.
      ...(dataNascimento === undefined ? {} : { dataNascimento: new Date(dataNascimento) }),
    })

    return reply.status(200).send(montarDetalhe(cidadao))
  }

  /**
   * `DELETE /api/v1/cidadaos/:id` — exclusão lógica.
   *
   * @returns 204, sem corpo, ou 404.
   */
  async excluir(
    request: FastifyRequest<{ Params: { id: string } }>,
    reply: FastifyReply,
  ): Promise<FastifyReply> {
    await this.service.excluir(request.params.id)

    // 204 significa "deu certo, e não há nada para devolver". Devolver o cadastro
    // excluído seria contraditório: acabamos de dizer que ele não deve mais
    // aparecer nas consultas.
    return reply.status(204).send()
  }
}
```

### `src/modules/cidadao/v1/cidadao.routes.ts`

```ts
/**
 * Rotas do cadastro de cidadão — versão 1
 *
 * Este arquivo **não sabe em que endereço vive**. Ele declara `/cidadaos`, e quem
 * decide que isso fica sob `/api/v1` é o `app.ts`, no `register`. A diferença
 * importa no dia da `v2`: nasce uma pasta `v2/` com o seu próprio arquivo de
 * rotas, registrada com outro prefixo, e este continua exatamente como está.
 *
 * O que **não** se duplica nesse dia é o service: a regra de negócio é a mesma
 * para todas as versões. O que muda entre versões é o formato do que entra e do
 * que sai, e é por isso que só `routes`, `controller` e `schema` moram aqui.
 */

import type { FastifyInstance } from 'fastify'
import type { ZodTypeProvider } from 'fastify-type-provider-zod'
import { z } from 'zod'
import { CidadaoRepository } from '../cidadao.repository.ts'
import { CidadaoService } from '../cidadao.service.ts'
import { CidadaoController } from './cidadao.controller.ts'
import {
  atualizarCidadaoBodySchema,
  cidadaoDetalheSchema,
  cidadaoListaSchema,
  cidadaoParamsSchema,
  criarCidadaoBodySchema,
} from './cidadao.schema.ts'

/**
 * Plugin de rotas do cadastro de cidadão.
 *
 * @param app Instância do Fastify, entregue automaticamente pelo `app.register()`.
 */
export async function cidadaoRoutes(app: FastifyInstance): Promise<void> {
  // A corrente de dependências, montada à mão: o repository fala com o banco, o
  // service usa o repository, o controller usa o service. Com poucos módulos isso
  // cabe em três linhas e deixa tudo visível.
  const repository = new CidadaoRepository()
  const service = new CidadaoService(repository)
  const controller = new CidadaoController(service)

  const rotas = app.withTypeProvider<ZodTypeProvider>()

  rotas.post(
    '/cidadaos',
    {
      schema: {
        summary: 'Cadastra um cidadão',
        description:
          'Cria um cadastro novo. O CPF é conferido pelos dígitos verificadores e ' +
          'precisa ser único: cada pessoa é uma linha, para sempre.',
        tags: ['cidadaos'],
        body: criarCidadaoBodySchema,
        response: { 201: cidadaoDetalheSchema },

        // Erro de negócio, que a estrutura da rota não revela: só quem conhece a
        // regra sabe que existe conflito de CPF. Esta chave não é validada pelo
        // Fastify — ela existe para a documentação, e o `shared/docs/erro.ts`
        // explica por que ela não pode virar um schema de resposta.
        errosPossiveis: ['409'],
      },
    },
    async (request, reply) => {
      return controller.criar(request, reply)
    },
  )

  rotas.get(
    '/cidadaos',
    {
      schema: {
        summary: 'Lista os cidadãos cadastrados',
        description:
          'Devolve apenas os cadastros ativos, do mais recente para o mais antigo. ' +
          'O CPF **não** vem nesta resposta: uma lista entrega o dado de todo mundo ' +
          'de uma vez, e a consulta individual já resolve quem precisa dele.',
        tags: ['cidadaos'],
        response: { 200: cidadaoListaSchema },
      },
    },
    async (request, reply) => {
      return controller.listar(request, reply)
    },
  )

  rotas.get(
    '/cidadaos/:id',
    {
      schema: {
        summary: 'Consulta um cidadão',
        description: 'Devolve o cadastro completo, com CPF.',
        tags: ['cidadaos'],
        params: cidadaoParamsSchema,
        response: { 200: cidadaoDetalheSchema },
      },
    },
    async (request, reply) => {
      return controller.buscarPorId(request, reply)
    },
  )

  rotas.patch(
    '/cidadaos/:id',
    {
      schema: {
        summary: 'Altera um cadastro',
        description:
          'Modifica somente os campos enviados. O CPF não pode ser alterado: trocá-lo ' +
          'não é corrigir um dado, é dizer que aquela linha passou a ser outra pessoa.',
        tags: ['cidadaos'],
        params: cidadaoParamsSchema,
        body: atualizarCidadaoBodySchema,
        response: { 200: cidadaoDetalheSchema },
      },
    },
    async (request, reply) => {
      return controller.atualizar(request, reply)
    },
  )

  rotas.delete(
    '/cidadaos/:id',
    {
      schema: {
        summary: 'Exclui um cadastro',
        description:
          'Exclusão **lógica**: o cadastro sai das consultas e continua no banco, com a ' +
          'data da exclusão registrada. O CPF continua ocupado — quem volta tem o ' +
          'cadastro reativado, e não recriado.',
        tags: ['cidadaos'],
        params: cidadaoParamsSchema,
        // Escrito em Zod como todo o resto: o serializador registrado no `app.ts`
        // é o do Zod, e um schema em JSON Schema cru faz a rota nem chegar a
        // subir. O 204 não tem corpo, e `z.null()` é como isso se declara.
        response: { 204: z.null().describe('Cadastro excluído.') },
      },
    },
    async (request, reply) => {
      return controller.excluir(request, reply)
    },
  )
}
```

### `src/app.ts`

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
import { cidadaoRoutes } from './modules/cidadao/v1/cidadao.routes.ts'
import { healthRoutes } from './modules/health/health.routes.ts'
import { registerDocs } from './shared/docs/index.ts'
import { env } from './shared/env/index.ts'
import { errorHandler, notFoundHandler } from './shared/errors/error-handler.ts'
import { buildLoggerOptions } from './shared/logger/index.ts'
import { registerSecurity } from './shared/security/index.ts'
import { configurarMensagensEmPortugues } from './shared/validation/zod-locale.ts'

/**
 * Prefixo sob o qual vivem as rotas de negócio.
 *
 * A versão fica no endereço, e não em um cabeçalho, por um motivo prático: assim
 * ela aparece no log, no registro do proxy e na barra do navegador. Investigar
 * "por que este cliente recebeu resposta diferente" fica sendo leitura, e não
 * dedução.
 *
 * **O que obriga a subir para `/api/v2`:** remover ou renomear campo da resposta,
 * tornar obrigatório um campo que era opcional, apertar uma validação, ou mudar o
 * significado de um código de status. Acrescentar campo opcional, acrescentar
 * rota e corrigir defeito **não** obrigam — nada disso quebra quem já consome.
 */
export const PREFIXO_DA_API = '/api/v1'

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
  //
  // `/health` fica FORA do prefixo de versão, e isso é decisão, não esquecimento.
  // Ela é infraestrutura de operação: quem a consulta é o `HEALTHCHECK` do
  // container e quem monitora o serviço, não quem integra com a API. Se o alarme
  // apontasse para `/api/v1/health`, o dia em que a API subisse para a `v2`
  // derrubaria o monitoramento junto — por uma mudança que não tem nada a ver
  // com ele.
  app.register(healthRoutes)

  // As rotas de negócio, essas sim, nascem sob o prefixo de versão. É o que
  // permite mudar o contrato um dia sem obrigar todos os sistemas que consomem a
  // API a mudarem no mesmo dia.
  app.register(cidadaoRoutes, { prefix: PREFIXO_DA_API })

  return app
}
```

### `src/shared/docs/erro.ts`

```ts
/**
 * O contrato de erro na documentação da API
 *
 * A API tem **um** formato de resposta de erro, montado pelo `error-handler.ts`.
 * Até aqui ele não aparecia na documentação: a página descrevia só as respostas
 * de sucesso, e quem fosse integrar descobriria o formato do erro errando.
 *
 * O caminho óbvio seria declarar `response: { 404: ... }` em cada rota. Ele não
 * serve, por dois motivos independentes:
 *
 *   1. **Passaria o erro pelo serializador.** O Fastify monta a resposta usando o
 *      schema declarado — e se o corpo do erro divergisse dele em qualquer
 *      detalhe, a serialização falharia e o cliente receberia 500 no lugar do
 *      400. A resposta de erro é justamente a que não pode falhar.
 *   2. **Não cobriria o 404 de endereço inexistente.** Ele acontece *sem rota*,
 *      no `notFoundHandler`, e portanto não tem schema onde ser declarado.
 *
 * A saída é descrever o erro **só na documentação**, sem que ele encoste no
 * caminho da resposta: um componente reutilizável do OpenAPI, referenciado por
 * cada operação depois que a especificação já está pronta.
 */

/** Nome do componente no OpenAPI. É por ele que as operações fazem referência. */
const NOME_DO_COMPONENTE = 'RespostaDeErro'

/** Caminho de referência do componente, no formato que o OpenAPI espera. */
const REFERENCIA = `#/components/schemas/${NOME_DO_COMPONENTE}`

/**
 * O formato único de erro da API, descrito uma vez só.
 *
 * Espelha a interface `RespostaDeErro` do `error-handler.ts`. São duas escritas
 * do mesmo formato, e isso é uma divergência possível — mas a alternativa era
 * derivar a documentação do caminho da resposta, que é exatamente o que o motivo
 * 1 do cabeçalho proíbe.
 */
const SCHEMA_DE_ERRO = {
  type: 'object',
  description: 'Formato único de erro desta API. Toda falha responde assim.',
  properties: {
    statusCode: {
      type: 'integer',
      description: 'Código HTTP repetido no corpo, para quem lê o JSON sem olhar o cabeçalho.',
      example: 404,
    },
    error: {
      type: 'string',
      description: 'Nome oficial do código HTTP, em inglês.',
      example: 'Not Found',
    },
    message: {
      type: 'string',
      description: 'Mensagem em português, escrita para ser lida por uma pessoa.',
      example: 'Cidadão não encontrado.',
    },
  },
  required: ['statusCode', 'error', 'message'],
}

/** Texto que acompanha cada código de erro na página da documentação. */
const DESCRICAO_POR_CODIGO: Record<string, string> = {
  '400': 'Dados inválidos na requisição.',
  '404': 'O recurso pedido não existe.',
  '409': 'A requisição está correta, mas conflita com o estado atual do cadastro.',
  '429': 'Limite de requisições excedido. Tente de novo depois do tempo indicado.',
  '500': 'Erro interno. O detalhe fica no log da equipe, nunca na resposta.',
}

/**
 * Códigos que **toda** operação pode devolver, sem exceção.
 *
 * O 429 vem do limite de requisições, que é global; o 500 é a rede de segurança
 * do tratamento de erros. Nenhum dos dois depende do que a rota faz.
 */
const ERROS_DE_TODA_ROTA = ['429', '500']

/**
 * Descreve uma referência ao componente de erro, no formato do OpenAPI.
 *
 * @param codigo Código HTTP que está sendo descrito.
 */
function respostaDeErro(codigo: string): unknown {
  return {
    description: DESCRICAO_POR_CODIGO[codigo] ?? 'Erro.',
    content: { 'application/json': { schema: { $ref: REFERENCIA } } },
  }
}

/**
 * Descobre quais erros uma operação pode devolver, lendo a própria operação.
 *
 * Preferimos derivar a deduzir de uma lista escrita à mão: uma lista envelhece em
 * silêncio quando alguém acrescenta um parâmetro à rota e esquece de atualizá-la.
 *
 *   • tem corpo ou parâmetro → pode falhar a validação, então pode dar **400**;
 *   • tem parâmetro no caminho → pode apontar para algo que não existe, **404**.
 *
 * O que não dá para derivar é declarado pela rota, em `errosPossiveis` — hoje só
 * o 409 do CPF repetido, que é regra de negócio e não estrutura.
 *
 * @param operacao   A operação como o gerador de OpenAPI a montou.
 * @param declarados Os códigos que a própria rota declarou, se houver.
 */
function descobrirErros(operacao: Record<string, unknown>, declarados: string[]): string[] {
  const codigos = new Set([...ERROS_DE_TODA_ROTA, ...declarados])

  const parametros = Array.isArray(operacao['parameters']) ? operacao['parameters'] : []
  const temCorpo = operacao['requestBody'] !== undefined

  if (temCorpo || parametros.length > 0) {
    codigos.add('400')
  }

  if (parametros.some((parametro: unknown) => (parametro as { in?: string }).in === 'path')) {
    codigos.add('404')
  }

  return [...codigos].sort()
}

/**
 * Acrescenta o contrato de erro à especificação já montada.
 *
 * Roda **depois** que o gerador terminou, o que é o ponto: nada aqui participa do
 * caminho da resposta. Se esta função tivesse um defeito, a página ficaria errada
 * — e a API continuaria respondendo certo.
 *
 * @param especificacao A especificação OpenAPI completa.
 * @param declarados    Erros de negócio por `método endereço`, vindos das rotas.
 * @returns             A mesma especificação, com o componente e as referências.
 */
export function acrescentarContratoDeErro(
  especificacao: Record<string, unknown>,
  declarados: Map<string, string[]>,
): Record<string, unknown> {
  const componentes = (especificacao['components'] ?? {}) as Record<string, unknown>
  const schemas = (componentes['schemas'] ?? {}) as Record<string, unknown>

  schemas[NOME_DO_COMPONENTE] = SCHEMA_DE_ERRO
  componentes['schemas'] = schemas
  especificacao['components'] = componentes

  const caminhos = (especificacao['paths'] ?? {}) as Record<string, Record<string, unknown>>

  for (const [caminho, operacoes] of Object.entries(caminhos)) {
    for (const [metodo, operacao] of Object.entries(operacoes)) {
      const detalhe = operacao as Record<string, unknown>
      const respostas = (detalhe['responses'] ?? {}) as Record<string, unknown>

      // O endereço da rota usa `:id`; o do OpenAPI usa `{id}`. A chave do mapa
      // foi gravada na forma da rota, então é ela que precisa ser reconstruída.
      const chave = `${metodo} ${caminho.replaceAll(/\{(\w+)\}/g, ':$1')}`

      for (const codigo of descobrirErros(detalhe, declarados.get(chave) ?? [])) {
        respostas[codigo] = respostaDeErro(codigo)
      }

      detalhe['responses'] = respostas
    }
  }

  return especificacao
}
```

### `src/shared/docs/index.ts`

```ts
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
import type { SwaggerTransformObject } from '@fastify/swagger'
import fastifySwaggerUi from '@fastify/swagger-ui'
import type { FastifyInstance } from 'fastify'
import { jsonSchemaTransform } from 'fastify-type-provider-zod'
import { acrescentarContratoDeErro } from './erro.ts'

/** Endereço em que a documentação fica disponível. */
export const DOCS_ROUTE_PREFIX = '/documentation'

/**
 * Erros de negócio que cada rota declara, guardados entre as duas etapas.
 *
 * A documentação é montada em dois momentos: o `transform` roda uma vez por rota,
 * e o `transformObject` roda uma vez no fim, com a especificação inteira. Quem
 * conhece o `409` do CPF repetido é a rota; quem escreve a resposta é a etapa
 * final. Este mapa é a carona entre as duas.
 *
 * A chave é `método + endereço`, então duas montagens da mesma rota escrevem o
 * mesmo valor — o que torna inofensivo o fato de ele viver fora da instância.
 */
const ERROS_DECLARADOS = new Map<string, string[]>()

/**
 * Formato do schema de rota, com a chave extra que este projeto acrescenta.
 *
 * O Fastify só compila `body`, `querystring`, `params`, `headers` e `response`.
 * Qualquer outra chave é levada adiante sem ser interpretada — é assim que
 * `summary`, `description` e `tags` funcionam, e é onde `errosPossiveis` entra.
 */
interface SchemaDaRota {
  errosPossiveis?: string[]
  [chave: string]: unknown
}

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
        { name: 'cidadaos', description: 'Cadastro de cidadãos atendidos pelo órgão' },
      ],
    },

    // Esta linha é a ponte. Os nossos schemas são escritos em Zod, e o formato
    // OpenAPI é outro; o `jsonSchemaTransform` traduz um no outro. Sem ela, o
    // plugin não entenderia nenhum dos schemas do projeto.
    //
    // Envolvemos o tradutor para recolher, antes que ele rode, os erros de
    // negócio que a rota declarou. Eles não são schema de resposta — se fossem,
    // passariam pelo serializador, e uma divergência no corpo do erro viraria 500
    // em vez do código certo.
    transform: ({ schema, url, route, ...resto }) => {
      const { errosPossiveis, ...schemaLimpo } = (schema ?? {}) as SchemaDaRota

      if (errosPossiveis !== undefined) {
        // A chave leva o método junto: `POST /api/v1/cidadaos` pode dar 409 por
        // CPF repetido, e o `GET` do mesmo endereço não pode. Guardar só pelo
        // endereço faria a listagem prometer um erro que ela nunca devolve.
        const metodos = Array.isArray(route.method) ? route.method : [route.method]

        for (const metodo of metodos) {
          ERROS_DECLARADOS.set(`${metodo.toLowerCase()} ${url}`, errosPossiveis)
        }
      }

      return jsonSchemaTransform({ schema: schemaLimpo, url, route, ...resto })
    },

    // Roda uma vez, com a especificação pronta, imediatamente antes de ela ser
    // servida. É aqui que o contrato de erro é acrescentado — depois de tudo, e
    // fora do caminho da resposta.
    transformObject: (documento) => {
      // O tipo do plugin é uma união: `openapiObject` quando a especificação é
      // OpenAPI 3, `swaggerObject` quando é Swagger 2. Este projeto usa a
      // primeira, e o `in` é o que prova isso ao TypeScript.
      const especificacao = ('openapiObject' in documento
        ? documento.openapiObject
        : documento.swaggerObject) as unknown as Record<string, unknown>

      return acrescentarContratoDeErro(
        especificacao,
        ERROS_DECLARADOS,
      ) as ReturnType<SwaggerTransformObject>
    },
  })

  app.register(fastifySwaggerUi, {
    routePrefix: DOCS_ROUTE_PREFIX,
  })
}
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
     * Um arquivo de teste por vez.
     *
     * O Vitest roda arquivos em paralelo, e até a Aula 14 isso era só ganho de
     * tempo: um único arquivo tocava o banco. Com o cadastro de cidadão passaram
     * a ser dois — o do service e o das rotas —, e os dois começam cada teste
     * limpando a tabela.
     *
     * Em paralelo, contra o **mesmo** banco de teste, um apaga o dado que o outro
     * acabou de gravar. O resultado medido foi 10 falhas que não se repetiam nas
     * mesmas linhas: CPF repetido sendo aceito, registro sumindo entre gravar e
     * consultar. Nenhuma delas era defeito do código sob teste.
     *
     * A alternativa seria um banco por arquivo. Ela é mais rápida e bem mais
     * difícil de explicar — e o custo desta linha, hoje, é medido em segundos.
     */
    fileParallelism: false,

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

// CPFs inventados, com dígito verificador que fecha. Material didático não usa
// documento de pessoa real — nem mesmo um que pareça válido.
const CPF_A = '00000000191'
const CPF_B = '00000000272'

/** Data de nascimento qualquer, para os testes que não estão medindo a data. */
const NASCIMENTO = new Date('1990-05-20')

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
    const cidadao = await repository.criar({
      nomeCompleto: 'Maria Souza',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    expect(cidadao.id).toMatch(/^[0-9a-f-]{36}$/)
    expect(cidadao.nomeCompleto).toBe('Maria Souza')
    expect(cidadao.criadoEm).toBeInstanceOf(Date)
    expect(cidadao.atualizadoEm).toBeInstanceOf(Date)
  })

  it('guarda como nulo todo campo opcional que não é informado', async () => {
    const cidadao = await repository.criar({
      nomeCompleto: 'João Lima',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    // Os campos são opcionais no schema (`String?`), e o que chega do banco é
    // `null`, não `undefined`. São coisas diferentes em TypeScript, e confundi-las
    // produz comparação que nunca dá certo.
    expect(cidadao.email).toBeNull()
    expect(cidadao.nomeSocial).toBeNull()
    expect(cidadao.municipio).toBeNull()

    // As colunas de auditoria nascem nulas porque ainda não existe login. É
    // dívida registrada, e este teste é o que a mantém visível.
    expect(cidadao.criadoPor).toBeNull()

    // E, principalmente: quem nasce, nasce ativo.
    expect(cidadao.excluidoEm).toBeNull()
  })

  it('guarda a data de nascimento sem hora, no dia exato que recebeu', async () => {
    const cidadao = await repository.criar({
      nomeCompleto: 'Ana Dias',
      cpf: CPF_A,
      dataNascimento: new Date('2001-07-04'),
    })

    // A coluna é `DATE`, sem hora. Guardar com hora é o que produz o clássico
    // "nasceu dia 4, aparece dia 3" quando o fuso horário entra na conta.
    expect(cidadao.dataNascimento.toISOString()).toBe('2001-07-04T00:00:00.000Z')
  })

  it('encontra pelo CPF o que acabou de gravar', async () => {
    await repository.criar({
      nomeCompleto: 'Ana Dias',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
      email: 'ana@exemplo.gov.br',
    })

    const encontrado = await repository.buscarPorCpf(CPF_A)

    expect(encontrado?.nomeCompleto).toBe('Ana Dias')
    expect(encontrado?.email).toBe('ana@exemplo.gov.br')
  })

  it('devolve nulo quando o CPF não está cadastrado', async () => {
    expect(await repository.buscarPorCpf(CPF_B)).toBeNull()
  })

  it('lista do cadastro mais recente para o mais antigo', async () => {
    await repository.criar({ nomeCompleto: 'Primeira', cpf: CPF_A, dataNascimento: NASCIMENTO })
    await repository.criar({ nomeCompleto: 'Segunda', cpf: CPF_B, dataNascimento: NASCIMENTO })

    const lista = await repository.listar()

    expect(lista.map((cidadao) => cidadao.nomeCompleto)).toEqual(['Segunda', 'Primeira'])
  })

  it('recusa CPF repetido, porque o banco tem índice único', async () => {
    await repository.criar({ nomeCompleto: 'Original', cpf: CPF_A, dataNascimento: NASCIMENTO })

    // Este é o teste que só um banco de verdade consegue provar. A garantia não
    // está no código: está na coluna, criada pela migration a partir do `@unique`.
    await expect(
      repository.criar({ nomeCompleto: 'Cópia', cpf: CPF_A, dataNascimento: NASCIMENTO }),
    ).rejects.toThrow()
  })
})

describe('CidadaoRepository — exclusão lógica', () => {
  it('some da listagem, mas continua no banco', async () => {
    const cidadao = await repository.criar({
      nomeCompleto: 'Excluído',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    await repository.excluir(cidadao.id)

    expect(await repository.listar()).toHaveLength(0)

    // A prova que importa: a linha continua lá. Este teste consulta o banco por
    // fora do repositório de propósito — perguntar ao repositório se ele apagou
    // seria confiar na palavra de quem está sendo testado.
    const linha = await prisma.cidadao.findUnique({ where: { id: cidadao.id } })

    expect(linha).not.toBeNull()
    expect(linha?.excluidoEm).toBeInstanceOf(Date)
  })

  it('deixa de ser encontrado por id depois de excluído', async () => {
    const cidadao = await repository.criar({
      nomeCompleto: 'Excluído',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    await repository.excluir(cidadao.id)

    expect(await repository.buscarPorId(cidadao.id)).toBeNull()
  })

  it('continua sendo encontrado por CPF depois de excluído', async () => {
    const cidadao = await repository.criar({
      nomeCompleto: 'Excluído',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    await repository.excluir(cidadao.id)

    // É a única consulta que ignora a exclusão, e o motivo é a coluna `@unique`:
    // o CPF continua ocupado. Se esta busca filtrasse por ativos, a regra de
    // duplicidade diria "pode cadastrar" e o banco recusaria logo depois.
    expect(await repository.buscarPorCpf(CPF_A)).not.toBeNull()
  })
})

describe('CidadaoService — as regras que o banco não conhece', () => {
  it('cadastra quando o CPF é novo', async () => {
    const cidadao = await service.cadastrar({
      nomeCompleto: 'Carlos Reis',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    expect(await repository.buscarPorId(cidadao.id)).not.toBeNull()
  })

  it('recusa CPF já cadastrado com mensagem em português e status 409', async () => {
    await service.cadastrar({ nomeCompleto: 'Carlos Reis', cpf: CPF_A, dataNascimento: NASCIMENTO })

    // O banco também recusaria — mas com uma mensagem escrita para
    // desenvolvedor, que a Aula 06 proibiu de sair para quem chama a API.
    await expect(
      service.cadastrar({ nomeCompleto: 'Outro', cpf: CPF_A, dataNascimento: NASCIMENTO }),
    ).rejects.toThrow(new AppError('Já existe um cidadão cadastrado com este CPF.', 409))
  })

  it('avisa que o CPF pertence a um cadastro excluído, em vez de repetir a mensagem comum', async () => {
    const cidadao = await service.cadastrar({
      nomeCompleto: 'Carlos Reis',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    await service.excluir(cidadao.id)

    // Mensagem diferente porque a ação certa é diferente. "CPF já cadastrado"
    // para um cadastro que a pessoa não vê na listagem parece defeito do sistema.
    await expect(
      service.cadastrar({ nomeCompleto: 'Carlos Reis', cpf: CPF_A, dataNascimento: NASCIMENTO }),
    ).rejects.toThrow(/cadastro excluído/)
  })

  it('não grava nada quando o cadastro é recusado', async () => {
    await service.cadastrar({ nomeCompleto: 'Carlos Reis', cpf: CPF_A, dataNascimento: NASCIMENTO })

    await expect(
      service.cadastrar({ nomeCompleto: 'Outro', cpf: CPF_A, dataNascimento: NASCIMENTO }),
    ).rejects.toThrow(AppError)

    expect(await repository.listar()).toHaveLength(1)
  })

  it('responde 404 ao buscar id inexistente', async () => {
    await expect(service.buscarPorId('nao-existe')).rejects.toThrow(
      new AppError('Cidadão não encontrado.', 404),
    )
  })

  it('altera apenas os campos enviados', async () => {
    const cidadao = await service.cadastrar({
      nomeCompleto: 'Carlos Reis',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
      email: 'carlos@exemplo.gov.br',
    })

    const alterado = await service.atualizar(cidadao.id, { telefone: '15999990000' })

    expect(alterado.telefone).toBe('15999990000')
    // O que não foi enviado precisa continuar exatamente como estava. É a
    // diferença entre `PATCH` e `PUT`, e apagar dado por omissão é justamente o
    // acidente que o `PATCH` existe para evitar.
    expect(alterado.email).toBe('carlos@exemplo.gov.br')
    expect(alterado.nomeCompleto).toBe('Carlos Reis')
  })

  it('recusa alterar quem foi excluído', async () => {
    const cidadao = await service.cadastrar({
      nomeCompleto: 'Carlos Reis',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    await service.excluir(cidadao.id)

    await expect(service.atualizar(cidadao.id, { telefone: '15999990000' })).rejects.toThrow(
      new AppError('Cidadão não encontrado.', 404),
    )
  })

  it('recusa excluir duas vezes', async () => {
    const cidadao = await service.cadastrar({
      nomeCompleto: 'Carlos Reis',
      cpf: CPF_A,
      dataNascimento: NASCIMENTO,
    })

    await service.excluir(cidadao.id)

    await expect(service.excluir(cidadao.id)).rejects.toThrow(
      new AppError('Cidadão não encontrado.', 404),
    )
  })
})
```

### `src/modules/cidadao/v1/cidadao.routes.spec.ts`

```ts
/**
 * Testes das rotas do cadastro de cidadão — versão 1
 *
 * Aqui a API é exercitada por inteiro, do endereço à resposta, com `app.inject()`
 * — que percorre todo o caminho de uma requisição real (validação, rota,
 * controller, service, banco, serializador) **sem abrir porta de rede**.
 *
 * Três coisas só este nível consegue provar, e nenhuma delas aparece nos testes
 * de service:
 *
 *   • que `/health` ficou **fora** do prefixo de versão;
 *   • que o CPF **não sai** na listagem, porque o serializador só deixa passar o
 *     que o schema declara;
 *   • que cada erro chega ao cliente com o código e a frase certos.
 */

import { afterAll, beforeEach, describe, expect, it } from 'vitest'

import { buildApp } from '../../../app.ts'
import { prisma } from '../../../shared/database/index.ts'

const app = buildApp({ logger: false })

/** Cadastro mínimo válido: só os três campos obrigatórios. */
const CADASTRO_MINIMO = {
  cpf: '00000000191',
  nomeCompleto: 'Maria Aparecida Souza',
  dataNascimento: '1985-03-12',
}

beforeEach(async () => {
  await prisma.cidadao.deleteMany()
})

afterAll(async () => {
  await app.close()
  await prisma.$disconnect()
})

/** Cadastra alguém e devolve a resposta já convertida. */
async function cadastrar(dados: Record<string, unknown> = CADASTRO_MINIMO) {
  const resposta = await app.inject({ method: 'POST', url: '/api/v1/cidadaos', payload: dados })

  return { status: resposta.statusCode, corpo: resposta.json() as Record<string, unknown> }
}

describe('Versionamento da API', () => {
  it('serve as rotas de negócio sob /api/v1', async () => {
    const resposta = await app.inject({ method: 'GET', url: '/api/v1/cidadaos' })

    expect(resposta.statusCode).toBe(200)
  })

  it('não serve as rotas de negócio fora do prefixo', async () => {
    const resposta = await app.inject({ method: 'GET', url: '/cidadaos' })

    expect(resposta.statusCode).toBe(404)
  })

  it('mantém /health FORA do prefixo de versão', async () => {
    // Esta é a decisão que a aula defende, e é o teste que impede alguém de
    // "organizar" as rotas um dia movendo o `/health` para dentro do prefixo.
    // No dia da `v2`, isso derrubaria o monitoramento junto.
    expect((await app.inject({ method: 'GET', url: '/health' })).statusCode).toBe(200)
    expect((await app.inject({ method: 'GET', url: '/api/v1/health' })).statusCode).toBe(404)
  })
})

describe('POST /api/v1/cidadaos', () => {
  it('cadastra e responde 201 com o endereço do que foi criado', async () => {
    const resposta = await app.inject({
      method: 'POST',
      url: '/api/v1/cidadaos',
      payload: CADASTRO_MINIMO,
    })

    expect(resposta.statusCode).toBe(201)
    // 201 sem `Location` obriga quem cadastrou a adivinhar onde o registro foi
    // parar. O cabeçalho responde a pergunta seguinte antes de ela ser feita.
    expect(resposta.headers['location']).toBe(`/api/v1/cidadaos/${resposta.json().id as string}`)
  })

  it('aceita CPF com pontuação e guarda só os dígitos', async () => {
    const { corpo } = await cadastrar({ ...CADASTRO_MINIMO, cpf: '000.000.001-91' })

    // Guardar as duas grafias faria a coluna `@unique` deixar o mesmo CPF entrar
    // duas vezes: para o banco, são textos diferentes.
    expect(corpo['cpf']).toBe('00000000191')
  })

  it('recusa CPF com dígito verificador errado', async () => {
    const { status, corpo } = await cadastrar({ ...CADASTRO_MINIMO, cpf: '12345678900' })

    expect(status).toBe(400)
    expect(corpo['message']).toContain('CPF inválido')
  })

  it('recusa data de nascimento no futuro', async () => {
    const { status, corpo } = await cadastrar({ ...CADASTRO_MINIMO, dataNascimento: '2087-01-01' })

    expect(status).toBe(400)
    expect(corpo['message']).toContain('não pode estar no futuro')
  })

  it('recusa CPF repetido com 409, e não com 500', async () => {
    await cadastrar()

    const { status, corpo } = await cadastrar()

    // 500 seria o que aconteceria se a regra não existisse: o erro do índice
    // único do banco subiria cru até o tratamento de erros.
    expect(status).toBe(409)
    expect(corpo['message']).toBe('Já existe um cidadão cadastrado com este CPF.')
  })

  it('recusa cadastro sem os campos obrigatórios', async () => {
    const { status, corpo } = await cadastrar({ nomeCompleto: 'Só o nome' })

    expect(status).toBe(400)
    expect(corpo['message']).toContain('cpf')
  })
})

describe('GET /api/v1/cidadaos — a listagem', () => {
  it('não devolve o CPF de ninguém', async () => {
    await cadastrar()

    const resposta = await app.inject({ method: 'GET', url: '/api/v1/cidadaos' })
    const lista = resposta.json() as Record<string, unknown>[]

    // Teste de AUSÊNCIA. Ele não confere o que a resposta tem: confere o que ela
    // não pode ter. É o serializador do Fastify cumprindo o schema — o controller
    // poderia devolver o registro inteiro que o CPF continuaria de fora.
    expect(lista).toHaveLength(1)
    expect(lista[0]).not.toHaveProperty('cpf')
    expect(lista[0]).not.toHaveProperty('criadoPor')
  })

  it('devolve o CPF na consulta individual', async () => {
    const { corpo } = await cadastrar()

    const resposta = await app.inject({
      method: 'GET',
      url: `/api/v1/cidadaos/${corpo['id'] as string}`,
    })

    expect(resposta.json()['cpf']).toBe('00000000191')
  })
})

describe('GET /api/v1/cidadaos/:id', () => {
  it('responde 404 para id que não existe', async () => {
    const resposta = await app.inject({
      method: 'GET',
      url: '/api/v1/cidadaos/00000000-0000-4000-8000-000000000000',
    })

    expect(resposta.statusCode).toBe(404)
    expect(resposta.json()['message']).toBe('Cidadão não encontrado.')
  })

  it('responde 400 para id que nem uuid é', async () => {
    // 400 e não 404: o problema está na requisição, não no que ela procura. A
    // diferença muda o que a outra ponta deve fazer — corrigir a chamada, e não
    // procurar outro registro.
    const resposta = await app.inject({ method: 'GET', url: '/api/v1/cidadaos/nao-e-uuid' })

    expect(resposta.statusCode).toBe(400)
  })
})

describe('PATCH /api/v1/cidadaos/:id', () => {
  it('altera só o que foi enviado', async () => {
    const { corpo } = await cadastrar({ ...CADASTRO_MINIMO, email: 'maria@exemplo.gov.br' })

    const resposta = await app.inject({
      method: 'PATCH',
      url: `/api/v1/cidadaos/${corpo['id'] as string}`,
      payload: { telefone: '15999990000' },
    })

    expect(resposta.statusCode).toBe(200)
    expect(resposta.json()['telefone']).toBe('15999990000')
    expect(resposta.json()['email']).toBe('maria@exemplo.gov.br')
  })

  it('recusa corpo vazio', async () => {
    const { corpo } = await cadastrar()

    const resposta = await app.inject({
      method: 'PATCH',
      url: `/api/v1/cidadaos/${corpo['id'] as string}`,
      payload: {},
    })

    // Sem esta recusa, a API responderia 200 a uma requisição que não fez nada —
    // sucesso para quem, do outro lado, achou que tinha alterado alguma coisa.
    expect(resposta.statusCode).toBe(400)
  })

  it('recusa alteração de CPF', async () => {
    const { corpo } = await cadastrar()

    const resposta = await app.inject({
      method: 'PATCH',
      url: `/api/v1/cidadaos/${corpo['id'] as string}`,
      payload: { cpf: '00000000272' },
    })

    // Trocar o CPF não é corrigir um dado: é dizer que aquela linha passou a ser
    // outra pessoa.
    expect(resposta.statusCode).toBe(400)
  })
})

describe('DELETE /api/v1/cidadaos/:id — exclusão lógica', () => {
  it('responde 204 sem corpo', async () => {
    const { corpo } = await cadastrar()

    const resposta = await app.inject({
      method: 'DELETE',
      url: `/api/v1/cidadaos/${corpo['id'] as string}`,
    })

    expect(resposta.statusCode).toBe(204)
    expect(resposta.body).toBe('')
  })

  it('tira da listagem e devolve 404 na consulta, mas mantém a linha no banco', async () => {
    const { corpo } = await cadastrar()
    const id = corpo['id'] as string

    await app.inject({ method: 'DELETE', url: `/api/v1/cidadaos/${id}` })

    expect((await app.inject({ method: 'GET', url: '/api/v1/cidadaos' })).json()).toHaveLength(0)
    expect((await app.inject({ method: 'GET', url: `/api/v1/cidadaos/${id}` })).statusCode).toBe(
      404,
    )

    // A prova de que foi exclusão lógica, e não `DELETE`: a linha continua lá.
    expect(await prisma.cidadao.findUnique({ where: { id } })).not.toBeNull()
  })

  it('mantém o CPF ocupado, e explica isso a quem tentar cadastrar de novo', async () => {
    const { corpo } = await cadastrar()

    await app.inject({ method: 'DELETE', url: `/api/v1/cidadaos/${corpo['id'] as string}` })

    const { status, corpo: recusa } = await cadastrar()

    expect(status).toBe(409)
    expect(recusa['message']).toContain('cadastro excluído')
  })
})

describe('O contrato de erro é o mesmo em toda a API', () => {
  it('responde erro sempre com statusCode, error e message', async () => {
    const semRota = await app.inject({ method: 'GET', url: '/api/v1/nao-existe' })
    const validacao = await app.inject({ method: 'GET', url: '/api/v1/cidadaos/nao-e-uuid' })

    for (const resposta of [semRota, validacao]) {
      expect(Object.keys(resposta.json()).sort()).toEqual(['error', 'message', 'statusCode'])
    }

    // O 404 sem rota é o caso que nenhum schema de rota alcança: ele acontece
    // antes de existir rota. É por isso que o contrato de erro é descrito só na
    // documentação, e não como schema de resposta.
    expect(semRota.json()['statusCode']).toBe(404)
  })
})
```

### `src/shared/docs/docs.spec.ts`

```ts
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
  paths: Record<
    string,
    Record<string, { summary?: string; tags?: string[]; responses?: Record<string, unknown> }>
  >
  components?: { schemas?: Record<string, { properties?: Record<string, unknown> }> }
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
    expect(Object.keys(especificacao.paths)).toEqual([
      '/health',
      '/health/ready',
      '/api/v1/cidadaos',
      '/api/v1/cidadaos/{id}',
    ])

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

describe('O contrato de erro na documentação', () => {
  it('descreve o formato de erro uma vez só, como componente reutilizável', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })
    const especificacao = resposta.json<EspecificacaoOpenApi>()

    // Um componente, referenciado por todas as operações. Descrever o erro rota
    // a rota daria o mesmo resultado na tela e um lugar novo para divergir a
    // cada rota criada.
    const componente = especificacao.components?.schemas?.['RespostaDeErro']

    expect(Object.keys(componente?.properties ?? {})).toEqual(['statusCode', 'error', 'message'])

    await app.close()
  })

  it('promete 409 no cadastro, e não na listagem do mesmo endereço', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })
    const rota = resposta.json<EspecificacaoOpenApi>().paths['/api/v1/cidadaos']

    // O conflito de CPF é regra de negócio: só quem a conhece sabe que existe.
    // Por isso ele é declarado pela rota — e declarado por MÉTODO, senão a
    // listagem prometeria um erro que nunca devolve.
    expect(Object.keys(rota?.['post']?.responses ?? {})).toContain('409')
    expect(Object.keys(rota?.['get']?.responses ?? {})).not.toContain('409')

    await app.close()
  })

  it('promete 404 onde existe parâmetro no endereço, e não onde não existe', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })
    const caminhos = resposta.json<EspecificacaoOpenApi>().paths

    // Derivado da própria operação, e não de uma lista escrita à mão: lista
    // envelhece em silêncio quando alguém acrescenta um parâmetro e esquece dela.
    expect(Object.keys(caminhos['/api/v1/cidadaos/{id}']?.['get']?.responses ?? {})).toContain(
      '404',
    )
    expect(Object.keys(caminhos['/health']?.['get']?.responses ?? {})).not.toContain('404')

    await app.close()
  })

  it('não deixa vazar a marca interna que carrega os erros declarados', async () => {
    const app = buildApp({ logger: false, docs: true })

    const resposta = await app.inject({ method: 'GET', url: '/documentation/json' })

    // `errosPossiveis` é invenção deste projeto, não faz parte do OpenAPI, e
    // serve só para levar a informação da rota até a montagem final.
    expect(resposta.body).not.toContain('errosPossiveis')

    await app.close()
  })
})
```

### `requisicoes/cidadaos.http`

```http
# Requisições do cadastro de cidadão
#
# Este arquivo é lido pela extensão REST Client (humao.rest-client).
# Com o servidor rodando (`npm run dev`), clique em "Send Request" logo acima
# de cada requisição para executá-la sem sair do editor.
#
# Repare que todas elas começam com `/api/v1`. A rota `/health` não — ela é
# infraestrutura de monitoramento, e por isso fica fora do prefixo de versão.

@host = http://localhost:3333
@api = {{host}}/api/v1

### Cadastrar um cidadão com o mínimo obrigatório
# São só três campos: CPF, nome completo e data de nascimento. Todo o resto é
# opcional, porque atendimento não pode travar por falta de complemento.
POST {{api}}/cidadaos
Content-Type: application/json

{
  "cpf": "529.982.247-25",
  "nomeCompleto": "Carlos Eduardo Nogueira",
  "dataNascimento": "1978-04-22"
}

### Cadastrar um cidadão com o cadastro completo
# Repare no CPF com pontuação e no telefone com parênteses: a API aceita as duas
# formas e guarda só os dígitos. Guardar formatado deixaria o mesmo CPF entrar
# duas vezes, em grafias diferentes, sem a coluna `@unique` perceber.
POST {{api}}/cidadaos
Content-Type: application/json

{
  "cpf": "111.444.777-35",
  "nomeCompleto": "Joana Batista Lima",
  "nomeSocial": "Joana Lima",
  "dataNascimento": "1992-11-30",
  "rg": "12.345.678-9",
  "orgaoEmissor": "SSP",
  "ufEmissor": "SP",
  "email": "joana@exemplo.gov.br",
  "telefone": "(15) 99876-5432",
  "cep": "18200-000",
  "logradouro": "Rua das Palmeiras",
  "numero": "120",
  "complemento": "Apto 31",
  "bairro": "Centro",
  "municipio": "Itapetininga",
  "uf": "SP"
}

### Listar os cidadãos ativos
# O CPF NÃO vem nesta resposta, de propósito: uma listagem exposta por engano
# entregaria o documento de todo mundo de uma vez.
GET {{api}}/cidadaos

### Consultar um cidadão pelo id (troque pelo id que o cadastro devolveu)
# Aqui o CPF vem, porque é uma consulta a um cadastro específico.
GET {{api}}/cidadaos/COLE-AQUI-O-ID

### Alterar só o telefone (troque pelo id)
# PATCH, e não PUT: o que não for enviado permanece exatamente como estava.
PATCH {{api}}/cidadaos/COLE-AQUI-O-ID
Content-Type: application/json

{
  "telefone": "1533334444"
}

### Excluir um cadastro (troque pelo id)
# Exclusão LÓGICA: a linha continua no banco, com a data de exclusão preenchida.
# A resposta é 204, sem corpo.
DELETE {{api}}/cidadaos/COLE-AQUI-O-ID

### Tentar cadastrar CPF já usado — deve responder 409
POST {{api}}/cidadaos
Content-Type: application/json

{
  "cpf": "529.982.247-25",
  "nomeCompleto": "Outra Pessoa Qualquer",
  "dataNascimento": "1990-01-01"
}

### Tentar cadastrar CPF mal formado — deve responder 400
# Os dois últimos dígitos do CPF são calculados a partir dos nove primeiros.
# Este número não fecha a conta, e a API recusa antes de tocar no banco.
POST {{api}}/cidadaos
Content-Type: application/json

{
  "cpf": "12345678900",
  "nomeCompleto": "Nome Qualquer",
  "dataNascimento": "1990-01-01"
}

### Tentar nascer no futuro — deve responder 400
POST {{api}}/cidadaos
Content-Type: application/json

{
  "cpf": "529.982.247-25",
  "nomeCompleto": "Nome Qualquer",
  "dataNascimento": "2087-01-01"
}

### Conferir que /health continua FORA do prefixo de versão
GET {{host}}/health

### E que dentro do prefixo ela não existe — deve responder 404
GET {{api}}/health
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
| `npm run db:deploy`    | Aplica as migrations já versionadas (servidor)     |
| `npm run db:seed`      | Popula o banco com registros de exemplo            |
| `npm run db:studio`    | Abre o Prisma Studio para ver e editar dados       |

## Rotas

As rotas de negócio vivem sob o prefixo de versão `/api/v1`. As de monitoramento **não**:

| Método   | Rota                   | O que faz                                         |
| :------- | :--------------------- | :------------------------------------------------ |
| `GET`    | `/health`              | **Vida:** apenas `{ "status": "ok" }`             |
| `GET`    | `/health/ready`        | **Prontidão:** status, uptime, momento e ambiente |
| `POST`   | `/api/v1/cidadaos`     | Cadastra um cidadão (201, ou 409 se o CPF existe) |
| `GET`    | `/api/v1/cidadaos`     | Lista os cadastros ativos, **sem CPF**            |
| `GET`    | `/api/v1/cidadaos/:id` | Consulta um cadastro completo, com CPF            |
| `PATCH`  | `/api/v1/cidadaos/:id` | Altera somente os campos enviados                 |
| `DELETE` | `/api/v1/cidadaos/:id` | Exclusão **lógica**: some das consultas (204)     |

Toda rota declara o contrato da resposta com Zod. Campo que não está no contrato **não sai**,
mesmo que o código o devolva por engano. É por isso que o CPF não aparece na listagem: ele não
está declarado no schema dela.

## Versionamento da API

`/health` fica **fora** de `/api/v1`, e isso é decisão, não esquecimento: quem a consulta é o
monitoramento e o `HEALTHCHECK` do container. Se o alarme apontasse para `/api/v1/health`, o dia
da `v2` derrubaria o monitoramento junto — por uma mudança que não tem nada a ver com ele.

**O que obriga a subir de versão:**

| Mudança                                           | Sobe versão? |
| :------------------------------------------------ | :----------: |
| Acrescentar campo opcional na resposta            |    ❌ Não    |
| Acrescentar rota nova                             |    ❌ Não    |
| Corrigir defeito que fazia a API responder errado |    ❌ Não    |
| Remover ou renomear campo da resposta             |    ✅ Sim    |
| Tornar obrigatório um campo que era opcional      |    ✅ Sim    |
| Apertar uma validação, aceitando menos que antes  |    ✅ Sim    |
| Mudar o significado de um código de status        |    ✅ Sim    |

A regra que governa a tabela inteira: **é retrocompatível o que não obriga ninguém a mexer no
código de quem consome.** É a mesma pergunta do padrão expande/contrai, feita sobre HTTP em vez
de sobre coluna de banco.

Quando a `v2` existir, a `v1` continua no ar por **6 meses**, respondendo com os cabeçalhos
`Deprecation: true` e `Sunset: <data>`. O que muda entre as versões são as rotas, os schemas e os
controllers, que vivem em `src/modules/cidadao/v1/`. O service e o repository são **um só**:
duplicar a regra de negócio criaria dois lugares para corrigir o mesmo defeito.

## Cadastro de cidadão

Três campos são obrigatórios, e são os que identificam a pessoa: **CPF**, **nome completo** e
**data de nascimento**. Todo o resto — documento, contato e endereço — é opcional, porque um
atendimento não pode ser recusado por falta de complemento de endereço.

| Decisão                              | Por quê                                                                                                                               |
| :----------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------ |
| **CPF conferido de verdade**         | Os dois últimos dígitos são calculados a partir dos nove primeiros. Isso prova que o número é **bem formado**, não que ele existe.    |
| **CPF guardado só com dígito**       | Guardar formatado deixaria o mesmo CPF entrar duas vezes em grafias diferentes, e a coluna `@unique` não perceberia.                  |
| **Nome social ao lado**              | Nome social é direito, e é por ele que a pessoa deve ser tratada. Fica ao lado do de registro, que o documento oficial exige.         |
| **Exclusão lógica**                  | Ninguém é apagado. `DELETE` transformaria histórico ligado ao cadastro em referência quebrada, sem desfazer.                          |
| **CPF de excluído continua ocupado** | Uma pessoa é uma linha, para sempre. Quem volta tem o cadastro **reativado**, e a API diz isso em vez de repetir "CPF já cadastrado". |
| **CPF fora da listagem**             | Uma lista exposta por engano entrega o documento de todo mundo de uma vez; a consulta individual entrega um.                          |
| **Auditoria não sai na API**         | `criadoPor` e `atualizadoPor` são informação de operação. Hoje nascem nulas — quem as preenche é a aula de autenticação.              |

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

A página descreve também o **formato de erro**, como um componente reutilizável chamado
`RespostaDeErro`. Ele não é declarado como schema de resposta em cada rota, e o motivo é
prático: isso passaria o corpo do erro pelo serializador, e qualquer divergência viraria 500 no
lugar do 400 — justamente na resposta que não pode falhar.

**Em produção os três respondem 404**, por decisão registrada. Agora que existem rotas de
negócio, metade do motivo original caiu; a outra metade continua de pé: publicar o mapa completo
de um cadastro com dado pessoal, sem exigir login, entrega mais a quem mapeia o serviço do que a
quem integra com ele. Quando houver autenticação, a documentação volta a subir — protegida por
login, e não desligada.

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
- **Migration em servidor roda com `db:deploy`**, nunca com `db:migrate`: o primeiro só
  aplica o que já está versionado; o segundo cria migration e faz perguntas. E ela roda
  **antes** de a versão nova subir, com o código velho ainda no ar — por isso toda alteração
  precisa ser retrocompatível (padrão expande/contrai).
- **Antes de qualquer migration em banco com dado: backup, e backup já restaurado alguma vez.**
  O passo a passo está em `docs/referencia/modelo-plano-de-migration.md`.
- **O usuário da aplicação não pode criar bancos.** É decisão de segurança, e é por isso que o
  banco de sombra exigido pelo `prisma migrate dev` já vem pronto do Compose, apontado pela
  `SHADOW_DATABASE_URL`.
````

---

## ✅ Como saber que deu certo

| O que conferir                    | Comando                              | Resultado esperado                       |
| :-------------------------------- | :----------------------------------- | :--------------------------------------- |
| O projeto inteiro                 | `npm run check`                      | Termina com **código 0**                 |
| Os testes                         | `npm test`                           | **127 passed**                           |
| A tabela tem o modelo real        | `DESCRIBE cidadaos;`                 | 23 colunas, `dataNascimento` `NO` (nulo) |
| A rota de negócio está versionada | `GET /api/v1/cidadaos`               | **200**, com a lista                     |
| A de monitoramento não está       | `GET /api/v1/health`                 | **404** no formato de erro da API        |
| E continua fora do prefixo        | `GET /health`                        | **200** `{"status":"ok"}`                |
| O CPF não vaza na listagem        | `GET /api/v1/cidadaos`               | Nenhum objeto tem o campo `cpf`          |
| E aparece na consulta individual  | `GET /api/v1/cidadaos/:id`           | O campo `cpf` está lá                    |
| O CPF é conferido                 | `POST` com `"cpf": "12345678900"`    | **400**, "CPF inválido"                  |
| A pontuação é aceita              | `POST` com `"cpf": "529.982.247-25"` | **201**, e grava `52998224725`           |
| O CPF repetido é recusado         | `POST` duas vezes com o mesmo CPF    | **409**, não 500                         |
| A exclusão é lógica               | `DELETE`, depois `SELECT` no banco   | 204; a linha continua, com `excluidoEm`  |
| O contrato de erro documentado    | `GET /documentation/json`            | Tem `components.schemas.RespostaDeErro`  |

**A saída real desta bateria, executada:**

```
GET /health           -> 200 {"status":"ok"}
GET /api/v1/health    -> 404 | Endereço não encontrado: GET /api/v1/health
POST                  -> 201 | Location: /api/v1/cidadaos/f7c70ac3-fe44-479a-a576-7296dad95ea8
  cpf gravado         : 52998224725   (enviamos "529.982.247-25")
  telefone gravado    : 15998765432   (enviamos "(15) 99876-5432")
POST CPF repetido     -> 409 | Já existe um cidadão cadastrado com este CPF.
POST CPF mal formado  -> 400 | Dados inválidos no corpo da requisição: cpf — CPF inválido.
POST data no futuro   -> 400 | ... dataNascimento — A data de nascimento não pode estar no futuro.
PATCH corpo vazio     -> 400 | ... Envie ao menos um campo para alterar.
DELETE                -> 204 | corpo: ""
GET do excluído       -> 404 | Cidadão não encontrado.
POST mesmo CPF        -> 409 | Este CPF pertence a um cadastro excluído. Reative...

no banco: linhas 4 | ativos 3 | excluidos 1
listagem: id, nomeCompleto, nomeSocial, dataNascimento, municipio, uf, criadoEm
```

---

## 🐛 Erros comuns

### `Invalid schema passed: {"type":"null"}` ao subir a API

Você escreveu o schema do 204 em JSON Schema cru:

```ts
response: { 204: { type: 'null' } }   // ❌
```

O serializador registrado no `app.ts` é o do **Zod**, desde a Aula 07. Todo schema de rota é
escrito em Zod:

```ts
response: { 204: z.null().describe('Cadastro excluído.') }   // ✅
```

A rota nem chega a subir — o erro aparece na partida, e não na primeira requisição.

### `There are N rows in this table, it is not possible to execute this step`

Coluna obrigatória em tabela que já tem linhas. É o Capítulo 2 inteiro: a coluna nasce opcional,
o dado é preenchido, e só então ela vira obrigatória.

### Os testes falham em lugares diferentes a cada execução

Estado compartilhado, quase nunca defeito do código. Se dois arquivos de teste usam o mesmo
banco e ambos limpam a tabela, eles se atropelam. `fileParallelism: false` no `vitest.config.ts`.

### `GET /api/v1/cidadaos` responde 404

Duas causas possíveis:

1. O `app.register(cidadaoRoutes, { prefix: PREFIXO_DA_API })` não foi acrescentado ao `app.ts`.
2. O arquivo de rotas declara `/api/v1/cidadaos` **e** o register também passa o prefixo — aí o
   endereço real virou `/api/v1/api/v1/cidadaos`. O arquivo de rotas declara só `/cidadaos`.

### A rota nova não aparece na documentação

Ela foi registrada **antes** do `registerDocs(app)`. O gerador não lê o código-fonte: ele escuta
cada rota sendo registrada, então rota que entra antes dele não é vista. E o defeito é
silencioso — a página abre normalmente, só que sem a rota. É o mesmo alerta da Aula 08.

### O cadastro grava, mas a data sai um dia antes

Você formatou a data com o fuso da máquina em vez do padrão ISO. `toISOString().slice(0, 10)`
devolve exatamente o dia gravado, porque a coluna é `DATE` e o banco entrega meia-noite em UTC.

### `Unknown argument \`cpf\`` ao alterar um cadastro

O `PATCH` não aceita CPF de propósito, e o schema o remove com `.omit({ cpf: true })`. Se a
requisição manda CPF, a resposta é 400 — e está certo.

---

## 🏋️ Exercícios

### Exercício 1 — o campo que não devia sair

Acrescente `criadoPor: 'teste'` ao objeto que o `montarResumo` devolve, no controller. Rode a
listagem e observe a resposta.

**Pergunta:** o campo apareceu? Por quê? E o que isso diz sobre onde mora a garantia de que dado
interno não vaza?

### Exercício 2 — reativar em vez de recadastrar

Hoje, quem tenta cadastrar o CPF de alguém excluído recebe 409 com uma orientação — mas a API
não oferece o caminho para segui-la.

Desenhe (**sem implementar**) a rota de reativação: qual método HTTP, qual endereço, qual código
de resposta, e o que acontece com os campos do cadastro antigo.

**Pergunta difícil:** reativar é criar ou alterar? A resposta muda o método.

### Exercício 3 — a mudança que obriga a `v2`

Para cada mudança abaixo, decida se ela obriga a subir de versão, e escreva o motivo em uma
frase:

1. Acrescentar o campo `profissao` (opcional) na resposta da consulta individual.
2. Passar a exigir `email` no cadastro.
3. Renomear `nomeCompleto` para `nome` na resposta.
4. Passar a recusar telefone com 9 dígitos (hoje já recusa, mas por outro caminho).
5. Fazer o `POST` responder 202 em vez de 201.

### Exercício 4 — o índice que não existe

A listagem filtra por `excluidoEm IS NULL`, e por isso essa coluna ganhou índice. A consulta por
CPF usa o índice único que já existia.

**Pergunta:** se amanhã surgir uma rota que busca cidadão **por município**, o que precisa
acontecer? Escreva a linha do schema e diga qual seria o custo de não fazer nada.

---

## 📌 O que vem depois

A **Aula 16** traz autenticação. Ela paga três dívidas que esta aula deixou anotadas:

- `criadoPor` e `atualizadoPor` deixam de nascer nulas — passa a existir de quem tirar o nome;
- `/health/ready` deixa de ser aberta (item **A-09** do checklist);
- a documentação volta a subir em produção, protegida por login em vez de desligada
  (item **A-10**).
