# 🐳 Aula 10: Docker — empacotando a API

Na Aula 01 você aprendeu a desconfiar da frase **"na minha máquina funciona"**.

Nesta aula você a elimina.

Hoje, para alguém rodar esta API em outro computador, essa pessoa precisa instalar a versão
certa do Node, instalar as dependências, compilar o TypeScript e configurar as variáveis de
ambiente — na ordem certa, sem errar nenhum passo. Se a versão do Node dela for diferente da
sua, o comportamento muda. E muda **em silêncio**: nada avisa, nada quebra na hora. O
problema aparece depois, no servidor, longe de quem escreveu o código.

O Docker resolve isso empacotando três coisas numa só: **o código compilado, as dependências
e a versão exata do Node**. O que roda na sua máquina passa a ser, byte a byte, o que roda no
servidor.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar a diferença entre **imagem** e **container** sem hesitar.
- Escrever um `Dockerfile` de dois estágios e dizer por que dois, e não um.
- Medir o tamanho da sua imagem e reduzi-lo em **42%** com uma mudança de estrutura.
- Provar que o `.dockerignore` impede que o seu `.env` com segredos entre na imagem.
- Subir a API em um container e acessá-la de fora.
- Fazer o Docker vigiar a saúde da sua API sozinho, com `HEALTHCHECK`.
- Rodar o processo como usuário sem privilégios, e não como `root`.

---

## 📋 Pré-requisitos

### O que você já precisa ter

Todas as aulas anteriores concluídas. Esta aula empacota o que já existe — se `npm run check`
não passar antes de começar, não é o Docker que vai consertar.

```bash
npm run check
```

### Instalando o Docker Desktop

Baixe em **<https://www.docker.com/products/docker-desktop/>** e instale.

Depois de instalar, **abra o Docker Desktop e espere ele terminar de iniciar.** A baleia na
barra de tarefas precisa parar de se mexer. O Docker tem duas partes: o comando que você
digita e um serviço que roda no fundo. O comando sozinho não faz nada.

> [!IMPORTANT]
> **Feche e reabra o terminal depois de instalar.** O instalador acrescenta o Docker à lista
> de programas que o terminal conhece, mas um terminal que já estava aberto continua com a
> lista antiga. Se você digitar `docker` e ler `command not found`, é quase sempre isto — e
> não uma instalação que deu errado.

### O teste que prova que está tudo certo

```bash
docker run --rm hello-world
```

Saída esperada na primeira vez:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
4f55086f7dd0: Pull complete
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Leia com atenção as duas primeiras linhas, porque elas respondem a uma dúvida que você vai
ter daqui a pouco: **"preciso baixar alguma coisa antes?"**

Não. `Unable to find image locally` → `Pulling` → pronto. O Docker percebeu que não tinha a
imagem, buscou sozinho e seguiu em frente. Você nunca vai precisar baixar nada à mão.

---

## 💣 Capítulo 1: A dor, escrita por extenso

Antes de qualquer `Dockerfile`, faça este exercício mental — ele é o argumento inteiro da
aula.

**Um colega vai rodar a sua API na máquina dele.** Escreva a lista do que ele precisa fazer:

1. Instalar o Node — **na versão certa**. Não "o Node"; a versão. Se ele tiver uma mais
   antiga, algo que você usa pode não existir lá.
2. Clonar o repositório.
3. Rodar `npm install` — e torcer para que nenhum pacote se comporte diferente no sistema
   operacional dele.
4. Copiar o `.env.example` para `.env`.
5. Preencher as variáveis, sabendo o que cada uma significa.
6. Rodar `npm run build`.
7. Rodar `npm start`.

São sete passos, e **cada um é uma chance de errar**. O passo 1 é o pior de todos, porque
quando ele erra, não dá erro: dá comportamento diferente.

Agora a mesma coisa, depois desta aula:

```bash
docker run -p 3333:3333 curso_api
```

Um comando. E a versão do Node não é escolha de quem roda — está dentro do pacote.

---

## 📖 Capítulo 2: Imagem, container, camada

Três palavras. Elas voltam o tempo todo, e confundi-las trava o entendimento de tudo o mais.

### A analogia

Pense em uma **receita de bolo impressa** e nos **bolos** feitos a partir dela.

| No mundo real                     | No Docker       |
| :-------------------------------- | :-------------- |
| A receita impressa                | A **imagem**    |
| O bolo que você assou hoje        | O **container** |
| Cada instrução escrita na receita | Uma **camada**  |

O que essa analogia acerta, e por isso vale a pena:

- Da mesma receita saem **muitos** bolos. Da mesma imagem sobem muitos containers, ao mesmo
  tempo, sem um atrapalhar o outro.
- **Jogar o bolo fora não apaga a receita.** Apagar um container não apaga a imagem — é por
  isso que subir de novo é instantâneo.
- A receita é **igual para todo mundo**. É exatamente essa propriedade que mata o "na minha
  máquina funciona".

### E a camada?

Cada instrução do `Dockerfile` vira uma camada, e o Docker **guarda cada uma em cache**.
Quando você reconstrói a imagem, ele reaproveita todas as camadas cujas instruções não
mudaram e refaz só da primeira alteração em diante.

Isso não é detalhe de otimização: é o que decide se o seu build leva 4 segundos ou 4 minutos.
E, como você vai ver no Capítulo 3, é o que determina **a ordem** das linhas do arquivo.

---

## 🐳 Capítulo 3: O primeiro `Dockerfile`

Vamos começar pelo jeito ingênuo — que **funciona** — e depois medir por que ele não serve.

Crie na raiz do projeto um arquivo chamado `Dockerfile` (sem extensão, com D maiúsculo):

```dockerfile
FROM node:24-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json tsconfig.build.json ./
COPY src ./src
RUN npm run build
EXPOSE 3333
CMD ["node", "dist/server.js"]
```

Linha por linha:

| Instrução | O que faz                                                                   |
| :-------- | :-------------------------------------------------------------------------- |
| `FROM`    | De onde partimos: uma imagem que já tem o Node 24 instalado no Linux        |
| `WORKDIR` | A pasta onde tudo acontece dentro do container. Cria se não existir         |
| `COPY`    | Traz arquivos da sua máquina para dentro da imagem                          |
| `RUN`     | Executa um comando **durante a construção** da imagem                       |
| `EXPOSE`  | Documenta em que porta esta imagem atende (não publica nada — veja adiante) |
| `CMD`     | O comando que roda **quando o container sobe**. Não durante a construção    |

A diferença entre `RUN` e `CMD` é a que mais confunde no começo: **`RUN` é enquanto a receita
é escrita; `CMD` é quando o bolo é servido.**

### Por que `npm ci` e não `npm install`

`npm ci` instala **exatamente** o que está no `package-lock.json` e falha se os dois arquivos
discordarem. É o que garante que o build de hoje e o de daqui a seis meses instalem as mesmas
versões — sem isso, o container que promete reprodutibilidade não a entrega.

### Por que os manifestos são copiados antes do `src/`

Repare que há dois `COPY` separados, e o `package.json` vem antes do código. Isso é
deliberado.

Enquanto `package.json` e `package-lock.json` não mudarem, o Docker reaproveita do cache a
camada com o `npm ci` inteiro já instalado. Se o código viesse antes, **cada linha alterada em
`src/` reinstalaria todas as dependências.** A ordem das linhas é uma decisão de desempenho.

### Construa e meça

```bash
docker build -t acapi_umestagio .
```

O `-t` dá um nome à imagem. O ponto no final é o **contexto de build** — a pasta que o Docker
recebe para trabalhar. Guarde esse detalhe: ele é o Capítulo 5 inteiro.

Na primeira vez o Docker baixa a imagem base (`node:24-slim`), o que leva alguns minutos e
uns 200 MB. Da segunda em diante, é instantâneo.

Agora meça:

```bash
docker images acapi_umestagio
```

```
REPOSITORY        TAG       SIZE
acapi_umestagio   latest    660MB
```

**660 MB.** A sua API tem cinco rotas.

---

## ✂️ Capítulo 4: Dois estágios

De onde vêm os 660 MB? De coisas que estão na imagem e **não têm nenhuma função lá dentro**:

- o TypeScript (compilador), usado só para gerar o `dist/`;
- o Vitest, o ESLint e o Prettier — ferramentas de quem escreve o código, não de quem o roda;
- o `src/` inteiro, em TypeScript, que ninguém executa;
- o `tsx`, que só serve para o modo de desenvolvimento.

Compilar precisa de tudo isso. **Rodar não precisa de nada disso.**

A solução se chama **build multi-estágio**: um `Dockerfile`, duas imagens temporárias. A
primeira compila. A segunda só recebe o resultado. Tudo o que ficou na primeira é descartado.

Substitua o `Dockerfile` inteiro por este:

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
RUN npm ci --omit=dev && npm cache clean --force

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

As duas instruções novas (`HEALTHCHECK` e `USER`) têm capítulo próprio adiante. Por ora,
concentre-se no `FROM` que aparece **duas vezes** e no `COPY --from=build`.

### O que a linha `COPY --from=build` faz

Ela é a fronteira. Diz: _"do estágio chamado `build`, traga a pasta `/app/dist` — e só ela."_

Tudo o que ficou no primeiro estágio (compilador, testes, código-fonte) é jogado fora quando
a construção termina. Não vai para a imagem final, não ocupa espaço, não pode ser lido por
ninguém que tenha acesso ao container.

### Construa e compare

```bash
docker build -t curso_api .
docker images
```

```
REPOSITORY               TAG       SIZE
curso_api   latest    380MB
acapi_umestagio          latest    660MB
```

| Estrutura     | Tamanho               |
| :------------ | :-------------------- |
| Um estágio    | 660 MB                |
| Dois estágios | **380 MB**            |
| Diferença     | 280 MB, **42% menor** |

Esses 280 MB viajam em cada cópia da imagem, toda vez que ela é enviada a um servidor. E,
mais importante que o tamanho: **o código-fonte da sua API não está mais lá dentro.**

Pode apagar a imagem antiga:

```bash
docker rmi acapi_umestagio
```

---

## 🚫 Capítulo 5: O `.dockerignore`

Lembra do ponto final em `docker build -t curso_api .`? Aquele ponto diz ao
Docker: _"a pasta do projeto é o seu material de trabalho"_. Isso se chama **contexto de
build**.

O `.dockerignore` é a lista do que **não** entra nesse material.

Crie o arquivo `.dockerignore` na raiz:

```
# =============================================================================
# O que NÃO é enviado ao Docker na hora de construir a imagem
# =============================================================================
#
# Antes de executar a primeira linha do `Dockerfile`, o Docker recebe uma cópia
# da pasta do projeto — o "contexto de build". Tudo o que não estiver listado
# aqui viaja, mesmo que o `Dockerfile` nunca copie o arquivo.
#
# São dois problemas em um: o build fica lento à toa, e coisa que não deveria
# sair da máquina sai.

# =============================================
# Dependências e código compilado
# =============================================
# Instalados dentro da imagem pelo `npm ci`, na plataforma certa. Os binários
# compilados aqui no Windows não funcionariam no Linux do container.
node_modules/

# Gerado dentro da imagem pelo `npm run build`. Mandar o `dist/` da máquina
# local arriscaria empacotar uma compilação velha, feita antes da última
# alteração no código.
dist/

# =============================================
# Segredos
# =============================================
# Este é o item mais importante da lista. O `.env` tem os valores reais desta
# máquina, senhas incluídas. No container, a configuração chega por variável de
# ambiente — o arquivo não tem por que atravessar.
.env
.env.*
!.env.example

# =============================================
# Histórico e configuração de ferramentas
# =============================================
.git/
.gitignore
.gitattributes
.vscode/
.editorconfig
.prettierrc.json
.prettierignore
eslint.config.js

# =============================================
# Documentação e material de trabalho
# =============================================
# Nada disso é executado pela API.
docs/
requisicoes/
*.md

# =============================================
# Testes
# =============================================
# O `tsconfig.build.json` já os mantém fora do `dist/`. Aqui eles nem chegam a
# ser enviados.
**/*.spec.ts
vitest.config.ts

# =============================================
# O próprio empacotamento
# =============================================
Dockerfile
.dockerignore
```

### A medição honesta, e uma armadilha

Se você apagar o `.dockerignore` e reconstruir a imagem **do jeito que a escrevemos**, quase
nada muda. Isso surpreende, e a explicação importa: o Docker moderno envia o contexto **sob
demanda**. Como o nosso `Dockerfile` pede arquivos por nome — `COPY src ./src`, e não
`COPY . .` —, ele nunca chega a pedir o `node_modules`.

Então o `.dockerignore` é inútil aqui? Não. Ele é a rede de segurança para o momento em que
alguém escrever `COPY . .` — que é o jeito mais comum de errar, porque é o que aparece na
maioria dos exemplos da internet.

Veja a diferença medida, com um `Dockerfile` que usa `COPY . .`:

| Com `COPY . .`          | Contexto enviado | Tempo     | Imagem | O `.env` entrou? |
| :---------------------- | :--------------- | :-------- | :----- | :--------------- |
| **Com** `.dockerignore` | 103,56 kB        | 0,1s      | 329 MB | não              |
| **Sem** `.dockerignore` | **130,83 MB**    | **50,8s** | 524 MB | **sim**          |

Sem o arquivo, foram parar dentro da imagem: o `node_modules` inteiro, o `.git` com todo o
histórico do projeto e o **`.env` com os valores reais da sua máquina**.

Guarde esta frase: **um segredo que entra numa imagem está na imagem para sempre**, em todas
as cópias dela, mesmo que um comando posterior o apague — porque a camada anterior continua
lá, e qualquer pessoa com a imagem consegue lê-la.

---

## 🎬 Capítulo 6: Rodando de verdade

Imagem construída. Agora suba um container:

```bash
docker run --rm -p 3333:3333 --name curso_api curso_api
```

As quatro partes:

| Trecho                          | O que faz                                                     |
| :------------------------------ | :------------------------------------------------------------ |
| `--rm`                          | Apaga o container quando ele parar. Sem isso eles se acumulam |
| `-p 3333:3333`                  | Liga a porta **da sua máquina** à porta **do container**      |
| `--name curso_api` | Dá um nome, para você não precisar decorar o código gerado    |
| o último argumento              | O nome da **imagem** a partir da qual o container é criado    |

O `-p` é o que mais gera dúvida. Leia como `-p SUA_MÁQUINA:CONTAINER`. O container tem a
própria rede, isolada; sem `-p`, a API sobe, funciona perfeitamente, e **ninguém consegue
falar com ela**.

Em outro terminal:

```bash
curl http://localhost:3333/health
```

```
{"status":"ok"}
```

E a rota de prontidão:

```bash
curl http://localhost:3333/health/ready
```

```json
{
  "status": "ok",
  "uptime": 36.897871916,
  "timestamp": "2026-08-18T12:08:03.830Z",
  "environment": "production"
}
```

Repare no `"environment": "production"`. Você não passou `NODE_ENV` em lugar nenhum — a linha
`ENV NODE_ENV=production` do `Dockerfile` é o padrão da imagem.

E, como consequência disso, a documentação **não** sobe:

```bash
curl -i http://localhost:3333/documentation
```

```
HTTP/1.1 404 Not Found
x-frame-options: SAMEORIGIN
{"statusCode":404,"error":"Not Found","message":"Endereço não encontrado: GET /documentation"}
```

A decisão da Aula 08 continua valendo dentro do container, sem nenhuma configuração extra. E
repare que os cabeçalhos de segurança da Aula 09 estão lá — **inclusive na resposta de erro**.

### O `HOST=0.0.0.0` finalmente mostra para que serve

Na Aula 04 você escreveu, no `.env.example`:

```
HOST=0.0.0.0
```

E leu que `0.0.0.0` significa "todas as interfaces desta máquina". Agora dá para **ver** a
diferença. Pare o container e suba de novo forçando o outro valor:

```bash
docker run --rm -p 3333:3333 -e HOST=127.0.0.1 curso_api
```

A API sobe normalmente, o log diz que está ouvindo, e o `curl` falha. Motivo: `127.0.0.1`
dentro do container significa **"só eu mesmo, aqui dentro"**. O `-p` entrega a requisição na
porta do container, mas não há ninguém escutando em uma interface acessível de fora.

Esse é o erro que mais consome tempo de quem está começando com container, porque nada dá
mensagem de erro. Volte para `0.0.0.0` e siga.

### Os atalhos

Agora que você digitou os comandos e sabe o que cada flag faz, vale guardá-los. No
`package.json`, dentro de `"scripts"`:

```json
"docker:build": "docker build -t curso_api .",
"docker:run": "docker run --rm -p 3333:3333 --name curso_api curso_api",
```

```bash
npm run docker:build
npm run docker:run
```

> [!NOTE]
> Os atalhos vêm **depois** do comando completo de propósito. Um atalho é economia de
> digitação para quem já entendeu o que está sendo economizado. Quem só sabe apertar o botão
> não sabe operar o container no servidor, onde não existe `package.json` para ajudar.

---

## 🩺 Capítulo 7: `HEALTHCHECK` — o Docker vigiando sozinho

Um processo pode estar **vivo** e ao mesmo tempo **inútil**: travado, sem conseguir responder,
sem ter morrido. Para o sistema operacional, ele está ótimo. Para quem depende da API, não.

O `HEALTHCHECK` resolve isso mandando o Docker perguntar, de tempos em tempos, se a API
responde de verdade:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||3333)+'/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
```

| Opção                | Significado                                                   |
| :------------------- | :------------------------------------------------------------ |
| `--interval=30s`     | Pergunta a cada 30 segundos                                   |
| `--timeout=3s`       | Se não responder em 3s, conta como falha                      |
| `--start-period=10s` | Carência inicial: falhas nos 10 primeiros segundos não contam |
| `--retries=3`        | Só declara `unhealthy` depois de 3 falhas seguidas            |

A regra do jogo é o **código de saída**: `0` significa saudável, qualquer outro significa
doente. É por isso que o comando termina em `process.exit(...)`.

### Três decisões dentro dessa linha

**1. Bate no `/health`, nunca no `/health/ready`.** A rota de prontidão devolve `uptime`,
timestamp e ambiente — informação interna. A Aula 07 separou as duas rotas exatamente por
isso, e aqui a separação rende.

**2. Lê a porta do ambiente.** Se cravássemos `3333`, o container apareceria como `unhealthy`
no dia em que alguém o subisse em outra porta, com a API perfeitamente no ar. Uma verificação
que mente é pior que verificação nenhuma.

**3. Não usa `curl`.** A imagem `slim` não o traz. Instalá-lo só para isso engordaria a
imagem e aumentaria a superfície de ataque. O Node já tem `fetch` embutido — usamos o que já
está lá.

### Veja funcionar

```bash
npm run docker:run
```

Em outro terminal, espere passar o `start-period` e rode:

```bash
docker ps
```

```
NAMES                    IMAGE                    STATUS
curso_api   curso_api   Up 23 seconds (healthy)
```

O `(healthy)` é o Docker respondendo que perguntou e a API respondeu. Para ver a última
verificação com detalhe:

```bash
docker inspect --format "{{json .State.Health}}" curso_api
```

```json
[{ "Start": "...", "End": "...", "ExitCode": 0, "Output": "" }]
```

### A verificação lê mesmo a variável `PORT`?

Não acredite: teste. Suba em outra porta e confira o estado:

```bash
docker run -d --rm --name porta4000 -e PORT=4000 -p 3444:4000 curso_api
docker inspect --format "{{.State.Health.Status}}" porta4000
```

```
healthy
```

A API subiu na 4000, a verificação leu a variável e foi bater na 4000. Se a porta estivesse
cravada no `Dockerfile`, este container apareceria como doente sem ter nada de errado.

```bash
docker stop porta4000
```

### E quando fica doente?

Suba um container com uma verificação que falha de propósito:

```bash
docker run -d --rm --name quebrado --health-cmd "node -e \"process.exit(1)\"" \
  --health-interval=3s --health-retries=2 --health-start-period=1s \
  curso_api

docker inspect --format "{{.State.Health.Status}}" quebrado
```

Acompanhe. Primeiro:

```
starting
```

Depois das tentativas:

```
unhealthy
```

**E aqui vem a parte que quase todo mundo entende errado:** o container continua rodando. O
`unhealthy` é um **aviso**, não uma execução. Quem age a partir dele é quem estiver
orquestrando os containers — que pode reiniciar, tirar do balanceador ou chamar alguém. O
Docker sozinho apenas informa.

```bash
docker stop quebrado
```

---

## 🔒 Capítulo 8: Não rode como `root`

Por padrão, o processo dentro do container roda como **`root`** — o usuário que pode tudo.

Isso não é um defeito do Docker; é o padrão mais permissivo, e ele existe para não atrapalhar
quem está começando. Mas se alguém conseguir executar algo dentro do seu container, faz muita
diferença se esse alguém é `root` ou não.

As imagens oficiais do Node já vêm com um usuário sem privilégios chamado `node`. Usar é uma
linha:

```dockerfile
USER node
```

E ela vem acompanhada de um detalhe no `COPY`:

```dockerfile
COPY --from=build --chown=node:node /app/dist ./dist
```

O `--chown` entrega os arquivos já pertencendo ao usuário `node`. Sem ele, o `dist/` chegaria
pertencendo ao `root`, e o processo poderia não conseguir lê-lo.

### A prova

```bash
npm run docker:run
```

Em outro terminal:

```bash
docker exec curso_api whoami
```

```
node
```

E, de quebra, confirme que o código-fonte não está lá dentro:

```bash
docker exec curso_api ls /app
```

```
dist  node_modules  package-lock.json  package.json
```

Nenhum `src/`. Nenhum arquivo `.ts`. Nenhum teste. Só o que é preciso para rodar.

> [!TIP]
> **Se você estiver no Git Bash, no Windows**, comandos com caminho absoluto podem ser
> convertidos errado — `ls /app` vira algo como `C:/Program Files/Git/app`. Use o PowerShell,
> ou escreva `//app` com duas barras.

---

## ⏱️ Capítulo 9: o container revelou um problema nos testes

Ao construir imagens, você vai reparar em uma coisa: **dentro do container, nada está
aquecido.** Nenhum cache, nenhum arquivo já lido pelo sistema. Toda partida é uma partida
fria.

Isso expôs um problema que já existia no projeto e ninguém tinha visto. Em uma execução do
`npm run check`, três dos 52 testes falharam assim:

```
Error: Test timed out in 5000ms.
Duration  9.83s (transform 1.23s, import 13.08s, tests 24.12s)
```

Nenhum teste ficou lento. O que estourou foi a **primeira importação** do Fastify e dos
plugins: 13 segundos, contra os 2 a 3 segundos habituais. E como o Vitest dá 5 segundos a
cada teste por padrão, os que carregaram primeiro morreram esperando.

Rodando de novo, tudo passou. **E esse é exatamente o problema.** Um teste que reprova por
velocidade de máquina ensina o time a reexecutar até passar — e a partir daí ninguém acredita
mais em falha nenhuma.

Crie na raiz o arquivo `vitest.config.ts`:

```typescript
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

import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
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

### Como saber que o arquivo está sendo lido

Um arquivo de configuração criado no lugar errado não dá erro: ele é simplesmente ignorado, e
você segue achando que configurou algo. Prove que não é o caso.

Troque `testTimeout: 15_000` por `testTimeout: 1` e rode `npm test`:

```
⎯⎯⎯⎯⎯⎯ Failed Tests 31 ⎯⎯⎯⎯⎯⎯⎯
Error: Test timed out in 1ms.
```

Se os testes reprovarem, o arquivo está sendo lido. **Desfaça a alteração** e siga.

---

## 📄 Capítulo 10: os arquivos que mudaram junto

### `package.json`

Duas mudanças: os atalhos do Docker e o `engines`.

O `engines` dizia `>=22`, enquanto o `.nvmrc` dizia `v24.2.0` e a imagem agora diz `node:24`.
Três lugares, três respostas — enquanto ninguém empacotava, a divergência não custava nada.
Com o `Dockerfile`, a versão declarada passa a ser a versão que roda de verdade, e as três
precisam concordar.

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
    "docker:build": "docker build -t curso_api .",
    "docker:run": "docker run --rm -p 3333:3333 --name curso_api curso_api",
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

### `.vscode/extensions.json`

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
    "ms-azuretools.vscode-docker"
  ]
}
```

### `README.md`

Acrescente ao final da seção de comandos:

````markdown
### Rodando por container

Requer o Docker Desktop instalado e em execução.

```bash
npm run docker:build   # constrói a imagem
npm run docker:run     # sobe a API na porta 3333
```

A imagem é multi-estágio: o código-fonte e as ferramentas de desenvolvimento ficam fora da
imagem final. O processo roda como usuário sem privilégios, e o Docker verifica a saúde da
API pela rota `/health` a cada 30 segundos.
````

---

## 💾 Fechando o ciclo: mande para o GitHub

```bash
git add .
git status
git commit -m "feat: empacota a API em imagem Docker multi-estágio"
git push
```

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Precisa terminar sem erro, com os 52 testes passando.

### 2. A imagem constrói do zero

```bash
docker build -t curso_api .
```

### 3. A imagem tem o tamanho esperado

```bash
docker images curso_api
```

Perto de **380 MB**. Se estiver perto de 660 MB, o `Dockerfile` ficou com um estágio só.

### 4. O container responde de fora

```bash
npm run docker:run
curl http://localhost:3333/health
```

```
{"status":"ok"}
```

### 5. O Docker considera a API saudável

```bash
docker ps
```

A coluna `STATUS` precisa mostrar `(healthy)`.

### 6. O processo não é `root`

```bash
docker exec curso_api whoami
```

```
node
```

### 7. O código-fonte não está na imagem

```bash
docker exec curso_api ls /app
```

Não pode aparecer `src/`.

---

## 🚨 Erros Comuns

| Sintoma                                                   | Causa provável                                                                                    |
| :-------------------------------------------------------- | :------------------------------------------------------------------------------------------------ |
| `docker: command not found` logo depois de instalar       | O terminal estava aberto antes da instalação. Feche e abra de novo                                |
| `Cannot connect to the Docker daemon`                     | O Docker Desktop não está aberto, ou ainda está iniciando                                         |
| A API sobe, mas o `curl` não conecta                      | Faltou o `-p 3333:3333`, ou o `HOST` está em `127.0.0.1` em vez de `0.0.0.0`                      |
| `port is already allocated`                               | Já existe algo na 3333 — a API rodando fora do container, ou outro container. Pare antes          |
| Alterei o código e o container não mudou                  | A imagem é uma fotografia. Reconstrua com `npm run docker:build`                                  |
| `npm ci` falha dizendo que o lock não bate                | `package.json` e `package-lock.json` divergem. Rode `npm install` na sua máquina e commite o lock |
| O build demora muito toda vez                             | Provavelmente o `COPY src` está antes do `COPY package.json`, invalidando o cache do `npm ci`     |
| `ls /app` responde `C:/Program Files/Git/app`             | Git Bash convertendo o caminho. Use o PowerShell, ou `//app`                                      |
| O container fica `unhealthy` e a API responde normalmente | A porta da verificação e a porta da API divergem. Confira o `PORT`                                |
| A imagem ficou com mais de 500 MB                         | Falta o `.dockerignore`, ou o `--omit=dev` no segundo estágio                                     |

---

## 🏋️ Exercícios

**1. O cache das camadas.** Altere uma linha em qualquer arquivo de `src/` e reconstrua a
imagem. Observe quais etapas dizem `CACHED` e quais rodam de novo. Depois altere o
`package.json` (a versão, por exemplo) e reconstrua. Compare: por que o segundo caso demora
tanto mais?

**2. O contexto de build.** Crie um `Dockerfile.teste` que use `COPY . .` e construa com e sem
o `.dockerignore`, anotando o "transferring context" das duas vezes. Depois entre nas duas
imagens e procure o arquivo `.env`.

**3. Dois de uma vez.** Suba dois containers da mesma imagem ao mesmo tempo, em portas
diferentes, e prove que os dois respondem. O que isso diz sobre a relação entre imagem e
container?

**4. Doente de propósito.** Faça um container ficar `unhealthy` e observe o que acontece com
ele em seguida. Ele para sozinho? Quem deveria agir?

Os gabaritos estão em [`exercicios/10-gabarito.md`](./exercicios/10-gabarito.md).

---

## 📌 O que fica para a próxima aula

Você deve ter reparado que, ao parar o container com `Ctrl+C` ou `docker stop`, a API morre
**na hora**. Se houvesse uma requisição sendo processada naquele instante, ela seria cortada
no meio.

Isso tem nome — **desligamento gracioso** — e é o assunto da Aula 11. Ela só faz sentido
agora, porque só agora existe o `docker stop` para demonstrar o problema em vez de descrevê-lo.
