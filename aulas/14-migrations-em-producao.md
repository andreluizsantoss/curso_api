# 🚨 Aula 14: Migrations em produção

Na Aula 13 você criou tabela, alterou coluna e recomeçou do zero quantas vezes quis. Se algo
desse errado, `docker compose down -v` e pronto.

Esta aula é sobre o banco onde essa opção **não existe**.

A diferença não é técnica — os comandos são quase os mesmos. A diferença é que lá existe dado
de gente, e cada erro tem consequência que não se apaga. Quatro perguntas que a Aula 13 deixou
sem resposta de propósito:

1. **Em que momento do deploy a migration roda?**
2. **Como alterar uma tabela sem derrubar a versão que está no ar?**
3. **O que acontece se a migration falhar no meio?**
4. **Como se volta** — e quem escreveu o plano de volta **antes** de rodar?

> [!IMPORTANT]
> Tudo aqui foi executado, inclusive as falhas. Você vai **provocar** um erro em "produção" de
> propósito, ver o sistema travar e destravá-lo. É muito melhor fazer isso hoje, com um banco
> que ninguém usa, do que às onze da noite de uma sexta-feira.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar a diferença entre `migrate dev` e `migrate deploy` sem consultar nada.
- Aplicar migration com a API **no ar**, sem derrubar ninguém.
- Executar o padrão **expande/contrai** nos quatro passos, e dizer por que ele existe.
- Reconhecer os erros `P3018` e `P3009` e sair deles.
- Explicar por que `migrate resolve` **não desfaz nada**.
- Fazer backup e — o que importa — **provar que ele restaura**.
- Preencher um plano de migration antes de tocar em produção.

---

## 📋 Pré-requisitos

Você precisa ter concluído a **Aula 13** e ter o ambiente de pé:

```bash
npm run compose:up
npm run check
```

---

## 💣 Capítulo 1: um banco que dá para errar

Para praticar, precisamos de um banco que **já tenha dados**. Migration em banco vazio nunca
falha, e é justamente a falha que vamos estudar.

Acrescente ao script de inicialização do MySQL (arquivo completo no Capítulo 9):

```sql
CREATE DATABASE IF NOT EXISTS curso_api_producao;
GRANT ALL PRIVILEGES ON curso_api_producao.* TO 'curso_api'@'%';
FLUSH PRIVILEGES;
```

E, no `.env`, o endereço dele:

```bash
DATABASE_PRODUCAO_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api_producao"
```

Como o script de inicialização só roda quando o volume nasce:

```bash
docker compose down -v
npm run compose:up
```

> [!WARNING]
> **Este banco não é produção coisa nenhuma.** É um banco na sua máquina, com o mesmo servidor,
> a mesma senha e a mesma porta dos outros. O que torna produção diferente **não é o servidor**
> — é o dado que não dá para perder e a gente do outro lado esperando.
>
> Em um sistema de verdade, este endereço não estaria em arquivo nenhum do repositório: seria
> variável de ambiente da máquina, com credencial própria e acesso restrito.

### Ponha dado dentro

Aplique as migrations que já existem e insira 200 cidadãos fictícios. O comando completo está
no gabarito do Exercício 1; o que importa é o resultado:

```
producao com 200 cidadaos e 40 telefones repetidos
```

Guarde esse detalhe dos **40 telefones repetidos**. Ele vai derrubar um deploy daqui a pouco.

---

## 📖 Capítulo 2: `migrate dev` e `migrate deploy` são comandos diferentes

Você usou `migrate dev` a aula inteira. Ele **nunca** roda em produção. Compare:

|                  | `migrate dev`                               | `migrate deploy`                           |
| :--------------- | :------------------------------------------ | :----------------------------------------- |
| Onde roda        | Só na máquina de quem desenvolve            | No servidor                                |
| O que faz        | Compara schema × banco e **cria** migration | **Aplica** o que já está versionado        |
| Banco de sombra  | Usa                                         | Não usa                                    |
| Pode apagar dado | Sim, se você mandar (e ele avisa)           | Nunca faz nada que não esteja no arquivo   |
| Interativo       | Sim — faz perguntas                         | Não — roda calado, para caber numa esteira |

A consequência prática é boa: `migrate deploy` **não decide nada**. Ele lê os arquivos SQL, na
ordem, e executa. Tudo o que ele vai fazer você já leu antes, no Pull Request.

> [!TIP]
> Se você rodar `migrate dev` num terminal que não aceita resposta — dentro de uma esteira, por
> exemplo — ele recusa e diz para usar o `deploy`. Não é bug: é o comando se protegendo de ser
> usado onde não deve.

---

## ⏱️ Capítulo 3: em que momento a migration roda

A ordem que este projeto adota, e que você deve saber defender:

```
1. backup                     ← imediatamente antes, sempre
2. prisma migrate deploy      ← schema novo, com o código VELHO ainda no ar
3. sobe a versão nova         ← código novo, sobre schema já pronto
4. confere
```

**Repare no passo 2.** A migration roda **antes** de o código novo subir. E isso significa que,
entre 2 e 3, existe um período — segundos ou minutos — em que **o código velho está rodando
sobre o schema novo**.

Não é um detalhe: é a regra que governa tudo o que vem depois nesta aula.

### Prove que é seguro

Com a API rodando apontada para o banco de produção, aplique a migration que acrescenta uma
coluna nova. Medido aqui:

```
migrate deploy levou 1504ms, com a API no ar
HTTP 200 {"status":"ok"}                      ← a API continuou respondendo
gravou: Gravado Durante o Deploy | nomeCompleto: NULL
```

A última linha é a mais importante: o **código velho continuou gravando**, sem saber que existe
uma coluna nova. Foi por isso que a migration pôde rodar com o sistema no ar.

E ela pôde porque a coluna nasceu **opcional**. Se tivesse nascido obrigatória, o código velho
— que não a preenche — passaria a falhar em cada cadastro, e o deploy viraria um incidente.

---

## 🧩 Capítulo 4: expande/contrai

Agora o caso que parece trivial e derruba sistema: **renomear uma coluna**.

O jeito ingênuo é uma linha: `ALTER TABLE cidadaos RENAME COLUMN nome TO nomeCompleto`. Ele
funciona — e, entre o passo 2 e o passo 3 do deploy, o código velho procura uma coluna que não
existe mais. Toda requisição que toca cidadão quebra.

O padrão **expande/contrai** troca uma janela de erro por quatro passos sem erro nenhum:

| Passo                   | O que roda                                    | O sistema fica                     |
| :---------------------- | :-------------------------------------------- | :--------------------------------- |
| **1. Expande**          | Migration: **acrescenta** a coluna nova, nula | Código velho intacto               |
| **2. Escreve nas duas** | Deploy de código que grava nas duas colunas   | Dado novo já chega completo        |
| **3. Migra o dado**     | `UPDATE` que copia o que faltava              | As duas colunas ficam equivalentes |
| **4. Contrai**          | Migration: **remove** a coluna antiga         | Só depois que ninguém mais a lê    |

### Passo 1 — expande

No `schema.prisma`, a coluna nova entra **opcional**, mesmo que ela vá ser obrigatória no fim:

```prisma
  nome         String
  nomeCompleto String?
```

```bash
npx prisma migrate dev --name expande-nome-completo
```

```sql
-- AlterTable
ALTER TABLE `cidadaos` ADD COLUMN `nomeCompleto` VARCHAR(191) NULL;
```

Aplicada em produção com a API no ar — é a medição do Capítulo 3.

### Passo 2 — escreve nas duas

Agora o código passa a gravar nos dois lugares. No repository:

```ts
  async criar(dados: DadosDoNovoCidadao): Promise<CidadaoModel> {
    // TRANSIÇÃO — passo 2 do expande/contrai. Enquanto as duas colunas existirem,
    // todo cadastro novo grava nas duas. Esta linha nasce para ser apagada no
    // passo "contrai", e isso é o combinado, não um esquecimento.
    return prisma.cidadao.create({ data: { ...dados, nomeCompleto: dados.nome } })
  }
```

> [!NOTE]
> Este é **código de transição**: ele existe para ser apagado. Você não está errando ao
> escrevê-lo — está construindo um andaime. Andaime que fica de pé para sempre vira dívida, e
> por isso o passo 4 não é opcional.

Conferindo depois de um cadastro novo:

```
nome: Cadastrado Na Transicao | nomeCompleto: Cadastrado Na Transicao
total 202 | preenchidos 1
```

Uma linha preenchida: a que acabou de nascer. As 201 antigas continuam sem o campo novo — e é
disso que trata o passo seguinte.

### Passo 3 — migra o dado que já existia

```sql
UPDATE cidadaos SET nomeCompleto = nome WHERE nomeCompleto IS NULL;
```

```
o UPDATE de 201 linhas levou 350ms
total 202 | preenchidos 202 | divergentes 0
```

`divergentes 0` é a conferência que autoriza o passo 4. Sem ela, você estaria removendo uma
coluna na esperança de que o dado tivesse sido copiado.

> [!CAUTION]
> Em uma tabela grande, esse `UPDATE` não se roda de uma vez: ele tranca linhas e pode segurar o
> banco. Faz-se em lotes — `WHERE nomeCompleto IS NULL LIMIT 1000`, repetido — com respiro entre
> eles. Aqui são 201 linhas e o assunto não aparece; com um milhão, ele é o assunto inteiro.

### Passo 4 — contrai

Só agora o código para de usar a coluna antiga, e só depois disso ela cai. No schema:

```prisma
  nomeCompleto String
```

O SQL gerado vem com um aviso que vale ler inteiro:

```sql
/*
  Warnings:

  - You are about to drop the column `nome` on the `cidadaos` table. All the data in the column will be lost.
  - Made the column `nomeCompleto` on table `cidadaos` required. This step will fail if there are existing NULL values in that column.

*/
-- AlterTable
ALTER TABLE `cidadaos` DROP COLUMN `nome`,
    MODIFY `nomeCompleto` VARCHAR(191) NOT NULL;
```

"This step will fail if there are existing NULL values" — é exatamente o que o passo 3 garantiu
que não aconteceria. O aviso deixa de ser assustador quando você fez a lição de casa.

```
Field: id cpf email criadoEm atualizadoEm telefone nomeCompleto | total 202
```

Coluna antiga fora, 202 linhas intactas, e em nenhum momento o sistema ficou fora do ar.

---

> [!CAUTION]
> **Nem toda migration perigosa falha — e as que passam são as caras.**
>
> Ao preparar o Exercício 2 desta aula, eu esperava ver um erro ao acrescentar uma coluna
> **obrigatória** a uma tabela com 200 linhas. Não houve erro:
>
> ```
> All migrations have been successfully applied.
>
> linhas  com_string_vazia
> 200     200
> ```
>
> O MySQL aceitou e preencheu as 200 linhas com **string vazia**. Um cadastro em que "órgão
> emissor é obrigatório" passou a significar "200 pessoas com órgão emissor vazio", sem uma
> linha de aviso.
>
> Quem recusou isso na Aula 13 foi o `migrate dev`, que confere os dados antes de gerar o SQL. O
> `migrate deploy` não confere nada: ele executa o arquivo. **É por isso que ler o SQL antes de
> aplicar não é preciosismo** — é a única etapa em que alguém olha.

## 💥 Capítulo 5: quando a migration falha no meio

Agora o cenário que a Aula 13 não podia mostrar. Você vai criar uma migration que **passa na sua
máquina e falha em produção** — que é como isso acontece de verdade.

A ideia: tornar o telefone único. No seu banco de trabalho, com três registros de seed e nenhum
telefone repetido, passa. Em produção, com os 40 repetidos do Capítulo 1, não.

```sql
-- CreateIndex
CREATE UNIQUE INDEX `cidadaos_telefone_key` ON `cidadaos`(`telefone`);
```

```bash
npm run db:deploy      # na sua máquina: passa
```

E em produção:

```
Error: P3018

A migration failed to apply. New migrations cannot be applied before the error is
recovered from.

Migration name: 20260818202500_telefone_unico

Database error code: 1062

Database error:
Duplicate entry '11999990000' for key 'cidadaos.cidadaos_telefone_key'
```

### O que ficou para trás

O Prisma guarda o histórico numa tabela do próprio banco, a `_prisma_migrations`. Olhe a linha
que a falha deixou:

```
     migration_name: 20260818202500_telefone_unico
applied_steps_count: 0
        finished_at: NULL
     rolled_back_at: NULL
```

`finished_at` vazio: **começou e não terminou**. É esse registro, e não o banco, que define o
que acontece a seguir.

### E o próximo deploy para

```
Error: P3009

migrate found failed migrations in the target database, new migrations will not be applied.
```

Parece um segundo problema, e é a coisa mais sensata que a ferramenta poderia fazer: com uma
migration em estado desconhecido, aplicar as próximas seria empilhar mudança sobre um banco que
ninguém sabe descrever.

### Destravando

```bash
npx prisma migrate resolve --rolled-back 20260818202500_telefone_unico
```

```
Migration 20260818202500_telefone_unico marked as rolled back.
```

> [!CAUTION]
> **Este comando não desfaz nada.** Ele só corrige o registro do histórico, dizendo "considere
> que essa migration não foi aplicada". Se ela tivesse alcançado a metade do trabalho, essa
> metade continuaria lá — desfazer o efeito é trabalho humano, comando a comando.
>
> A variante `--applied` diz o contrário: "eu corrigi na mão, considere aplicada". Escolher a
> errada é fazer o histórico mentir, e o próximo deploy vai acreditar no histórico.

E é por isso que rodar o deploy logo em seguida **falha de novo**: o registro está limpo, o dado
repetido continua lá. A ferramenta não tinha como consertar o que era problema de dado.

O conserto de verdade é decidir o que fazer com os 40 repetidos — aqui, esvaziá-los:

```sql
UPDATE cidadaos SET telefone = NULL WHERE telefone = '11999990000' AND id <> (
  SELECT id FROM (SELECT MIN(id) AS id FROM cidadaos WHERE telefone = '11999990000') AS p
);
```

```
All migrations have been successfully applied.
```

---

## 🔁 Capítulo 6: migration não se apaga

Terminado o susto, vem a conclusão desconfortável: **a restrição estava errada**. Duas pessoas
dividem telefone com frequência — mãe e filho, marido e mulher, o telefone do trabalho.

O instinto é apagar a pasta da migration. **Não faça isso.** Ela já rodou em produção: apagar o
arquivo deixaria o índice lá para sempre, e o histórico passaria a mentir sobre o que aconteceu
com aquele banco.

O certo é acrescentar a migration que **desfaz**:

```sql
-- DropIndex
DROP INDEX `cidadaos_telefone_key` ON `cidadaos`;
```

```
All migrations have been successfully applied.
índice único em produção: 0
```

O histórico agora conta a verdade inteira: alguém achou que telefone era único, isso foi ao ar,
e depois foi corrigido. Daqui a um ano, quem investigar um problema de telefone duplicado vai
encontrar essa história — em vez de um mistério.

> [!IMPORTANT]
> **A regra:** migration aplicada é fato consumado. Corrige-se com outra migration, nunca
> reescrevendo a anterior. Isso vale inclusive para o SQL: editar uma migration já aplicada faz
> o Prisma acusar diferença de conteúdo na próxima execução.

---

## 💾 Capítulo 7: backup, e a restauração provada

Toda a aula até aqui pressupôs uma coisa: que existia um backup. Agora ele.

```bash
docker compose exec mysql mysqldump -uroot -ptroque-esta-senha \
  --databases curso_api_producao > backups/producao-antes-da-migration.sql
```

```
mysqldump: 362ms | 40K
cidadaos 200 | com_email 133
```

Agora provoque o pior caso — uma migration que apaga coluna com dado:

```sql
ALTER TABLE cidadaos DROP COLUMN email;
```

Os 133 e-mails **sumiram**. Não há `Ctrl+Z`, não há lixeira.

```bash
docker compose exec -T mysql mysql -uroot -ptroque-esta-senha < backups/producao-antes-da-migration.sql
```

```
restaurou em 612ms
cidadaos 200 | com_email 133
```

Voltaram.

> [!IMPORTANT]
> **Backup que nunca foi restaurado é esperança, não backup.** O arquivo pode estar truncado,
> com a senha errada, no formato errado, ou simplesmente vazio — e ninguém descobre isso no dia
> em que precisa dele. Restaure de tempos em tempos, num banco descartável.

E o `.gitignore` ganha `backups/`. Um dump traz **todas** as linhas da tabela, inclusive dado
pessoal: versioná-lo é publicar o banco inteiro num lugar onde apagar depois não adianta.

---

## 📋 Capítulo 8: o plano escrito antes

Nada nesta aula é difícil. O que é difícil é fazer tudo isso às onze da noite, com o sistema
fora do ar e três pessoas perguntando quando volta.

Por isso existe o [`modelo-plano-de-migration.md`](../referencia/modelo-plano-de-migration.md):
um formulário curto, preenchido **antes**, que responde:

- o que muda, em uma frase;
- é retrocompatível? se não, qual a janela e quem é avisado;
- o backup foi feito quando, está onde, e **já foi restaurado alguma vez**;
- a consulta exata que confirma que deu certo;
- como voltar, comando a comando;
- quem executa, quem confere, quem decide voltar atrás.

Dez minutos de escrita hoje contra uma noite de improviso depois. É a mesma troca do plano de
implementação que este projeto faz desde a primeira aula.

---

## 📄 Capítulo 9: os arquivos, por inteiro

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

-- O "banco de produção" das aulas seguintes.
--
-- Ele não é produção coisa nenhuma: é um banco nesta mesma máquina, com dados
-- dentro, para que dê para praticar migration em algo que não é vazio. O que
-- torna produção diferente não é o servidor — é o dado que não dá para perder.
--
-- Em um servidor de verdade, o endereço deste banco não moraria em arquivo
-- nenhum do repositório: seria variável de ambiente da máquina, com credencial
-- própria e acesso restrito.
CREATE DATABASE IF NOT EXISTS curso_api_producao;
GRANT ALL PRIVILEGES ON curso_api_producao.* TO 'curso_api'@'%';
FLUSH PRIVILEGES;
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

# Endereço do "banco de produção" usado na Aula 14 para praticar migration em um
# banco que já tem dados. Ele é um banco nesta mesma máquina — o que torna
# produção diferente não é o servidor, é o dado que não dá para perder.
#
# Em um servidor de verdade este endereço NÃO estaria em arquivo do repositório:
# seria variável de ambiente da máquina, com credencial própria e acesso
# restrito a quem opera o sistema.
DATABASE_PRODUCAO_URL="mysql://curso_api:troque-esta-senha-tambem@localhost:3306/curso_api_producao"
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
# Cópias de segurança do banco
# =============================================
# Um dump traz TODAS as linhas da tabela — inclusive dado pessoal. Ele nunca vai
# para o Git: versionar backup é publicar o banco inteiro, para sempre, em um
# lugar onde apagar depois não adianta.
backups/

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
    "db:deploy": "prisma migrate deploy",
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

/// ATENÇÃO: este modelo é uma DEMONSTRAÇÃO, e não o cadastro definitivo.
///
/// Ele tem os campos mínimos para sustentar a aula de banco de dados. Um cadastro
/// de cidadão de verdade precisa de coisas que ninguém levantou ainda: nome
/// social, data de nascimento, endereço, trilha de auditoria (quem cadastrou e
/// quem alterou), exclusão lógica em vez de apagar a linha, e o tratamento de CPF
/// exigido pela LGPD. Os tipos também são os padrões do Prisma, e não os que a
/// tabela pediria — CPF tem tamanho fixo, e o `id` é um uuid.
///
/// O modelo real entra na Aula 15, junto da primeira funcionalidade de negócio,
/// quando existir requisito em vez de suposição. Item **P-17** do checklist.
///
/// A janela para trocar isso de graça é enquanto **nenhum servidor tiver rodado
/// `migrate deploy`**: aí basta apagar e recriar o histórico de migrations. Depois
/// do primeiro deploy, o mesmo ajuste vira `DROP TABLE` com dado de gente dentro.
model Cidadao {
  id           String   @id @default(uuid())
  nomeCompleto String
  cpf          String   @unique
  email        String?
  telefone     String?
  criadoEm     DateTime @default(now())
  atualizadoEm DateTime @updatedAt

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
  { nomeCompleto: 'Maria Aparecida Souza', cpf: '00000000191', email: 'maria@exemplo.gov.br' },
  { nomeCompleto: 'João Batista Lima', cpf: '00000000272', email: null },
  { nomeCompleto: 'Ana Clara Dias', cpf: '00000000353', email: 'ana@exemplo.gov.br' },
]

async function semear(): Promise<void> {
  for (const cidadao of CIDADAOS) {
    await prisma.cidadao.upsert({
      // O CPF é a chave natural do cadastro: é por ele que se sabe se aquela
      // pessoa já está no banco. Usar o `id` aqui não serviria, porque ele é
      // gerado no momento da criação e ninguém o conhece de antemão.
      where: { cpf: cidadao.cpf },
      update: { nomeCompleto: cidadao.nomeCompleto, email: cidadao.email },
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

### `src/modules/cidadao/cidadao.repository.ts`

Sem o código de transição: o passo 4 já aconteceu.

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
  nomeCompleto: string
  cpf: string
  email?: string | undefined
}

export class CidadaoRepository {
  /**
   * Grava um cidadão novo.
   *
   * @param dados Nome completo, CPF e, opcionalmente, e-mail.
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
   * @param dados Nome completo, CPF e, opcionalmente, e-mail.
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
    const cidadao = await repository.criar({ nomeCompleto: 'Maria Souza', cpf: CPF_A })

    expect(cidadao.id).toMatch(/^[0-9a-f-]{36}$/)
    expect(cidadao.nomeCompleto).toBe('Maria Souza')
    expect(cidadao.criadoEm).toBeInstanceOf(Date)
    expect(cidadao.atualizadoEm).toBeInstanceOf(Date)
  })

  it('guarda o e-mail como nulo quando ele não é informado', async () => {
    const cidadao = await repository.criar({ nomeCompleto: 'João Lima', cpf: CPF_A })

    // O campo é opcional no schema (`String?`), e o que chega do banco é `null`,
    // não `undefined`. São coisas diferentes em TypeScript, e confundi-las
    // produz comparação que nunca dá certo.
    expect(cidadao.email).toBeNull()
  })

  it('encontra pelo CPF o que acabou de gravar', async () => {
    await repository.criar({ nomeCompleto: 'Ana Dias', cpf: CPF_A, email: 'ana@exemplo.gov.br' })

    const encontrado = await repository.buscarPorCpf(CPF_A)

    expect(encontrado?.nomeCompleto).toBe('Ana Dias')
    expect(encontrado?.email).toBe('ana@exemplo.gov.br')
  })

  it('devolve nulo quando o CPF não está cadastrado', async () => {
    expect(await repository.buscarPorCpf(CPF_B)).toBeNull()
  })

  it('lista do cadastro mais recente para o mais antigo', async () => {
    await repository.criar({ nomeCompleto: 'Primeira', cpf: CPF_A })
    await repository.criar({ nomeCompleto: 'Segunda', cpf: CPF_B })

    const lista = await repository.listar()

    expect(lista.map((cidadao) => cidadao.nomeCompleto)).toEqual(['Segunda', 'Primeira'])
  })

  it('recusa CPF repetido, porque o banco tem índice único', async () => {
    await repository.criar({ nomeCompleto: 'Original', cpf: CPF_A })

    // Este é o teste que só um banco de verdade consegue provar. A garantia não
    // está no código: está na coluna, criada pela migration a partir do `@unique`.
    await expect(repository.criar({ nomeCompleto: 'Cópia', cpf: CPF_A })).rejects.toThrow()
  })
})

describe('CidadaoService — as regras que o banco não conhece', () => {
  it('cadastra quando o CPF é novo', async () => {
    const cidadao = await service.cadastrar({ nomeCompleto: 'Carlos Reis', cpf: CPF_A })

    expect(await repository.buscarPorId(cidadao.id)).not.toBeNull()
  })

  it('recusa CPF já cadastrado com mensagem em português e status 409', async () => {
    await service.cadastrar({ nomeCompleto: 'Carlos Reis', cpf: CPF_A })

    // O banco também recusaria — mas com uma mensagem escrita para
    // desenvolvedor, que a Aula 06 proibiu de sair para quem chama a API.
    await expect(service.cadastrar({ nomeCompleto: 'Outro', cpf: CPF_A })).rejects.toThrow(
      new AppError('Já existe um cidadão cadastrado com este CPF.', 409),
    )
  })

  it('não grava nada quando o cadastro é recusado', async () => {
    await service.cadastrar({ nomeCompleto: 'Carlos Reis', cpf: CPF_A })

    await expect(service.cadastrar({ nomeCompleto: 'Outro', cpf: CPF_A })).rejects.toThrow(AppError)

    expect(await repository.listar()).toHaveLength(1)
  })

  it('responde 404 ao buscar id inexistente', async () => {
    await expect(service.buscarPorId('nao-existe')).rejects.toThrow(
      new AppError('Cidadão não encontrado.', 404),
    )
  })
})
```

### `README.md`

O README do projeto ganha o comando novo e as duas regras que esta aula fixou.

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

### 1. A verificação completa passa

```bash
npm run check
```

### 2. Os quatro bancos existem

```bash
docker compose exec mysql mysql -ucurso_api -ptroque-esta-senha-tambem -e "SHOW DATABASES;"
```

`curso_api`, `_test`, `_shadow` e `_producao`.

### 3. Produção está em dia, sem migration falha

```bash
npx prisma migrate status
```

Com a `DATABASE_URL` apontando para o banco de produção: "Database schema is up to date!".

### 4. O índice único de telefone não existe

```bash
docker compose exec mysql mysql -uroot -ptroque-esta-senha -e \
  "SELECT COUNT(*) FROM information_schema.statistics WHERE table_schema='curso_api_producao' AND index_name='cidadaos_telefone_key';"
```

Precisa ser `0` — a migration compensatória fez o trabalho dela.

### 5. O backup restaura

A sequência inteira do Capítulo 7.

---

## 🚨 Erros Comuns

### "P3009: migrate found failed migrations"

Existe migration em estado falho no histórico. Descubra qual, resolva a **causa** e só então use
o `migrate resolve`. O Capítulo 5 inteiro é sobre isso.

### "P3018: A migration failed to apply"

A migration começou e não terminou. Leia o código de erro do banco na mensagem — `1062` é
duplicidade, `1451` é chave estrangeira, `1146` é tabela inexistente. O erro do banco diz o que
o Prisma não tem como adivinhar.

### "A migration passou, mas o dado ficou errado"

Coluna obrigatória acrescentada a tabela com linhas **não** falha no MySQL: ela preenche o que
já existia com o valor vazio do tipo. Confira com `SUM(coluna = '')`, e não apenas com
`IS NOT NULL` — os dois respondem coisas diferentes.

### "A migração rodou fora de ordem"

As migrations são aplicadas na **ordem alfabética do nome da pasta**, e o nome começa com data e
hora. O Prisma gera esse carimbo em **UTC**.

Se você criar uma migration à mão usando a hora local do seu computador, ela pode receber um
nome "anterior" às que já existem — e passar a rodar antes delas em qualquer banco novo. O
sintoma é bizarro: uma migration que altera tabela tentando rodar antes da que a cria.

Use UTC ao nomear à mão. Ou, melhor, deixe o Prisma nomear.

### "`prisma migrate dev` diz que é um comando interativo e sai"

Você o rodou em terminal que não aceita resposta. Em produção o comando é o `deploy`; na sua
máquina, rode em um terminal comum. Para só gerar o arquivo sem aplicar, use `--create-only`.

### "Editei uma migration já aplicada e agora tudo reclama"

O Prisma guarda a impressão digital de cada migration aplicada. Alterar o arquivo depois faz a
conferência falhar — e ela existe justamente para impedir que o histórico e o banco divirjam.
Acrescente uma migration nova.

---

## 🏋️ Exercícios

### 1. Encha o seu banco de produção

Escreva o comando que insere 200 cidadãos fictícios no banco de produção simulado, com telefones
repetidos de propósito. Explique por que 40 repetidos são suficientes para derrubar a migration
do Capítulo 5.

### 2. Faça a migration falhar por outro motivo

Provoque um `P3018` diferente do da aula: crie uma coluna **obrigatória sem valor padrão** em uma
tabela que já tem linhas. Anote o código de erro do banco e compare com o `1062`.

### 3. Meça o custo do expande/contrai

Cronometre os quatro passos do Capítulo 4 na sua máquina e some. Depois cronometre o
`RENAME COLUMN` direto. Com os dois números, explique em uma frase o que se está comprando com
a diferença.

### 4. Preencha o plano para uma mudança de verdade

Pegue o `modelo-plano-de-migration.md` e preencha-o para esta alteração: **tornar o e-mail
obrigatório**. Responda em especial a seção 2 — ela é retrocompatível? — e a seção 5.

Os gabaritos comentados estão em [`exercicios/14-gabarito.md`](./exercicios/14-gabarito.md).

---

## 📌 O que vem depois

Você já sabe alterar um banco que não pode parar. O que ainda não existe é **uma rota de
negócio** — a API guarda cidadão, e ninguém de fora consegue cadastrar um.

A **Aula 15** cria a primeira funcionalidade de verdade e, com ela, o versionamento da API
(`/api/v1`): o que obriga a subir de versão, por quanto tempo a anterior fica no ar, e por que
`/health` fica **fora** do prefixo.

E é lá que a tabela `cidadaos` deixa de ser demonstração: o modelo real, com os requisitos de
quem conhece o cadastro, nasce junto da primeira rota.
