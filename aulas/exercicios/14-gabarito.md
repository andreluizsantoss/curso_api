# 🚨 Gabarito — Aula 14: Migrations em produção

> Confira só depois de tentar. Os quatro exercícios foram executados antes de respondidos — e o
> **Exercício 2 não deu o resultado que eu esperava**. A resposta abaixo é o que de fato
> aconteceu, e ela é mais interessante que a previsão.

---

## Exercício 1 — Encha o seu banco de produção

> Escreva o comando que insere 200 cidadãos fictícios no banco de produção simulado, com
> telefones repetidos de propósito. Explique por que 40 repetidos são suficientes para derrubar
> a migration do Capítulo 5.

### O comando

Com a `DATABASE_URL` apontando para o banco de produção:

```bash
node --input-type=module -e "
const { PrismaMariaDb } = await import('@prisma/adapter-mariadb')
const m = await import('./src/generated/prisma/client.ts')
const prisma = new m.PrismaClient({ adapter: new PrismaMariaDb(process.env.DATABASE_URL) })

await prisma.cidadao.createMany({ data: Array.from({ length: 200 }, (_, i) => ({
  nomeCompleto: 'Cidadao de Teste ' + String(i + 1).padStart(3, '0'),
  cpf: String(10000000000 + i),
  email: i % 3 === 0 ? null : 'cidadao' + i + '@exemplo.gov.br',
  telefone: i % 5 === 0 ? '11999990000' : null,
})) })

console.log('producao com', await prisma.cidadao.count(), 'cidadaos')
await prisma.\$disconnect()"
```

```
producao com 200 cidadaos
```

### Por que 40 repetidos derrubam a migration

Não são 40 — **bastariam dois**. Um índice único recusa a criação assim que encontra o primeiro
par repetido, e o banco responde `1062 Duplicate entry`. Os outros 38 não acrescentam nada
além de tornar o problema fácil de ver.

O detalhe que vale reparar é o `i % 5 === 0`: as demais linhas ficam com `telefone` **nulo**, e
`NULL` **não** conflita com `NULL` num índice único do MySQL. Se todos os 200 tivessem telefone
nulo, a migration passaria — e a diferença entre passar e falhar seria invisível no schema.

---

## Exercício 2 — Faça a migration falhar por outro motivo

> Provoque um `P3018` diferente do da aula: crie uma coluna **obrigatória sem valor padrão** em
> uma tabela que já tem linhas. Anote o código de erro do banco e compare com o `1062`.

### O que eu esperava

Um erro parecido com o que a Aula 13 mostrou no `migrate dev`: "Added the required column
without a default value. There are N rows in this table".

### O que aconteceu

```sql
ALTER TABLE `cidadaos` ADD COLUMN `orgaoEmissor` VARCHAR(191) NOT NULL;
```

```
All migrations have been successfully applied.
```

**Passou.** E o resultado é pior do que ter falhado:

```
linhas  com_string_vazia
200     200
```

O MySQL aceitou a coluna obrigatória e preencheu as 200 linhas existentes com **string vazia**.
Nenhum erro, nenhum aviso, e um cadastro em que "órgão emissor obrigatório" agora significa
"200 pessoas com órgão emissor vazio".

### Por que a Aula 13 mostrou um erro, então?

Porque quem recusou lá **não foi o banco** — foi o `prisma migrate dev`, que compara schema e
dados **antes** de gerar o SQL e se recusa a criar uma migration que ele sabe que vai machucar.

O `migrate deploy` não faz essa conferência: ele executa o SQL que está no arquivo. É a mesma
divisão do Capítulo 2 desta aula — `dev` opina, `deploy` obedece.

### A resposta do exercício

| Caso                                  | O que acontece                   | Quem barra         |
| :------------------------------------ | :------------------------------- | :----------------- |
| Índice único com valores repetidos    | Falha com `1062`                 | O **banco**        |
| Coluna obrigatória em tabela com dado | **Não falha**: preenche com `''` | Ninguém, no deploy |

**A lição que sai daqui é mais valiosa que o erro que eu procurava:** nem toda migration
perigosa falha. Algumas passam e deixam o dado errado — e essas são as caras, porque só
aparecem semanas depois, quando alguém pergunta por que 200 cadastros têm o campo vazio.

O que protege contra isso não é a ferramenta: é **ler o SQL antes de aplicar**, que é o hábito
do `--create-only` da Aula 13.

---

## Exercício 3 — Meça o custo do expande/contrai

> Cronometre os quatro passos do Capítulo 4 e some. Depois cronometre o `RENAME COLUMN` direto.
> Com os dois números, explique o que se está comprando com a diferença.

### Os números medidos, com 200 linhas

| Caminho                            | Tempo                               | Sistema fora do ar             |
| :--------------------------------- | :---------------------------------- | :----------------------------- |
| `RENAME COLUMN` direto             | **393 ms**                          | Sim — durante o deploy inteiro |
| Expande (migration)                | 1.504 ms                            | Não                            |
| Migra o dado (`UPDATE` 201 linhas) | 350 ms                              | Não                            |
| Contrai (migration)                | ~1.500 ms                           | Não                            |
| **Expande/contrai, total**         | **~3,4 s** + dois deploys de código | **Não**                        |

### O que se compra com a diferença

Tempo de máquina é o que menos importa aqui: 393 ms contra 3,4 s não muda o dia de ninguém.

O que se compra são **duas janelas de indisponibilidade que deixam de existir** — a do deploy em
que o código velho procuraria uma coluna que sumiu, e a do retorno, caso fosse preciso voltar
atrás com o sistema já quebrado.

E o que se paga não é tempo: é **complexidade**. São quatro passos em vez de um, dois deploys
de código, e um trecho de transição que precisa ser removido depois. Em uma tabela com um
milhão de linhas o cálculo muda de figura — o `UPDATE` deixa de custar 350 ms e passa a exigir
lotes, e o `RENAME` deixa de custar 393 ms e passa a trancar a tabela por minutos.

**A regra prática:** o expande/contrai se justifica sempre que a alteração não for
retrocompatível e o sistema não puder parar. Se ele pode parar às três da manhã de domingo, o
caminho curto é uma escolha legítima — desde que seja escolha, e não descoberta.

---

## Exercício 4 — Preencha o plano para "tornar o e-mail obrigatório"

> Em especial a seção 2 (é retrocompatível?) e a seção 5 (como voltar).

### Seção 1 — O que muda

A coluna `email` da tabela `cidadaos` passa de opcional para obrigatória.

### Seção 2 — É retrocompatível?

**Não.** E por dois motivos independentes, que costumam ser confundidos:

1. **O dado existente não atende.** Hoje 67 dos 200 cadastros têm `email` nulo. Tornar a coluna
   obrigatória exige decidir o que fazer com eles — e "preencher com string vazia" é o que o
   banco faz sozinho se ninguém decidir (ver Exercício 2).
2. **O código velho não preenche.** Entre o `migrate deploy` e a subida da versão nova, todo
   cadastro feito sem e-mail passaria a falhar.

Por isso a alteração vira, ela também, um expande/contrai:

| Passo | O que roda                                                               |
| :---- | :----------------------------------------------------------------------- |
| 1     | Decidir o destino dos 67 sem e-mail (buscar, marcar como pendente, etc.) |
| 2     | Deploy do código que **exige** e-mail na entrada e sempre o grava        |
| 3     | `UPDATE` que resolve os 67 antigos, conforme a decisão do passo 1        |
| 4     | Migration que torna a coluna `NOT NULL`                                  |

### Seção 3 — Backup

```bash
docker compose exec mysql mysqldump -uroot -p<senha> \
  --databases curso_api_producao > backups/antes-email-obrigatorio.sql
```

Restauração testada: **sim** — a sequência do Capítulo 7, que levou 612 ms para 200 linhas.

### Seção 4 — Como conferir que deu certo

```sql
SELECT COUNT(*) AS total, COUNT(email) AS com_email, SUM(email = '') AS vazios
FROM cidadaos;
-- esperado: total = com_email, e vazios = 0
```

O `vazios = 0` é o que o Exercício 2 ensinou a conferir: coluna obrigatória preenchida com
string vazia passa em qualquer verificação que só olhe `NOT NULL`.

### Seção 5 — Como voltar

| Situação                                | O que fazer                                                                                  |
| :-------------------------------------- | :------------------------------------------------------------------------------------------- |
| A migration do passo 4 falha            | `migrate resolve --rolled-back`, decidir o dado pendente, aplicar de novo                    |
| O código novo rejeita cadastro legítimo | Voltar a versão anterior do código; o schema continua compatível                             |
| Precisa desfazer a obrigatoriedade      | Migration compensatória com `MODIFY email VARCHAR(191) NULL` — **nunca** apagando a anterior |
| O dado foi corrompido                   | Restaurar o arquivo da seção 3; tempo estimado ~1 s por 200 linhas                           |

### Seção 6 — Quem acompanha

Em uma equipe de duas pessoas, as três funções não podem ser a mesma pessoa em silêncio: quem
executa avisa antes, e a outra pessoa confere a seção 4 depois. O que não pode existir é
migration em produção que ninguém além de quem a rodou soube que aconteceu.
