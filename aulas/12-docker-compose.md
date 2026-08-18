# 🎼 Aula 12: Docker Compose — o ambiente completo

Na Aula 10 a API virou uma imagem que sobe em qualquer máquina. O problema é que a API não vive
sozinha: na próxima aula ela vai precisar de um **banco de dados** ao lado.

E aqui aparece um incômodo que já vinha crescendo há duas aulas:

1. **Instalar o MySQL na mão, em cada máquina**, é demorado e erra diferente em cada
   computador — exatamente o problema que a Aula 10 desarmou para a API, ainda de pé para
   aquilo de que a API depende.
2. **Na Aula 11 você montou a cena do proxy na unha**: criou uma rede, subiu dois containers
   com nomes combinados entre si e apontou um volume para um arquivo de configuração, na ordem
   certa. Funcionou. Agora tente lembrar da ordem daqui a duas semanas.
3. **O comando de subir já não cabe na cabeça.** O `docker:run` do `package.json` tem porta,
   nome e política de remoção. Cada peça nova alonga um comando que precisa ser digitado igual
   por todo mundo, sempre.

Os três têm a mesma forma: **o conhecimento de como subir o ambiente mora fora do
repositório** — na memória de alguém, ou no histórico do terminal de alguém. Quem entra no time
não recebe esse conhecimento junto com o `git clone`.

O Docker Compose traz esse conhecimento para dentro do Git, onde ele é lido, revisado e
versionado como qualquer outro código.

> [!IMPORTANT]
> Como na aula passada, tudo aqui foi medido com cronômetro. Se o seu número der diferente do
> nosso, isso é informação, não erro — anote e siga. O que não pode mudar é a **conclusão**.

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Descrever um ambiente inteiro em um arquivo versionado e subi-lo com um comando só.
- Explicar por que `depends_on` **não** garante que o serviço esteja pronto — e provar isso com
  cronômetro, na sua máquina.
- Corrigir esse problema com `healthcheck` e `condition: service_healthy`.
- Usar o **nome do serviço como endereço de rede**, sem configurar IP nenhum.
- Guardar dados em volume nomeado e explicar a diferença entre `down` e `down -v`.
- Manter o `npm run dev` como comando de trabalho, apontando para o banco do Compose.

---

## 📋 Pré-requisitos

Você precisa ter concluído a **Aula 11** e ter:

- o Docker Desktop instalado e **em execução**;
- o projeto passando no `npm run check`;
- o terminal aberto na pasta do projeto.

Confirme antes de começar:

```bash
docker compose version
npm run check
```

O primeiro precisa responder algo como `Docker Compose version v2.x`. Repare que é
`docker compose`, com espaço: o Compose faz parte do Docker há anos, e não é mais um programa
separado que se instala à parte.

---

## 💣 Capítulo 1: O que você faria sem o Compose

Antes de escrever qualquer coisa, vale olhar para o tamanho do problema.

Para ter a API e um banco conversando, na mão, você precisaria de:

```bash
# 1. Uma rede para os dois se enxergarem
docker network create rede-do-projeto

# 2. O banco, com quatro variáveis, um volume e a rede certa
docker run -d --name mysql --network rede-do-projeto \
  -e MYSQL_ROOT_PASSWORD=... -e MYSQL_DATABASE=... \
  -e MYSQL_USER=... -e MYSQL_PASSWORD=... \
  -v dados-mysql:/var/lib/mysql -p 3306:3306 mysql:8.4

# 3. A API, na mesma rede, com o nome combinado com o passo anterior
docker run -d --name api --network rede-do-projeto -p 3333:3333 curso_api
```

Três comandos longos, em ordem obrigatória, com nomes que precisam bater entre si. É
**exatamente** o que você fez na Aula 11 para montar o proxy — e é por isso que aquela cena
não sobreviveria a duas semanas de esquecimento.

O problema não é a dificuldade. É que **nada disso está no repositório**. Quem clonar o projeto
amanhã recebe o código e não recebe o ambiente.

> [!NOTE]
> Não rode os comandos acima. Eles estão aqui para você ver o tamanho do que vai desaparecer
> nas próximas páginas.

---

## 📖 Capítulo 2: O que o Compose é

**A analogia.** O `Dockerfile` é a receita de **um prato**. O `docker-compose.yml` é o **menu do
jantar inteiro**: quais pratos vão à mesa, em que ordem saem da cozinha e o que cada um precisa
para ficar pronto.

Na prática, o Compose é um arquivo que descreve **serviços**. Cada serviço vira um container. E
o Compose cuida sozinho de três coisas que você faria na mão:

| O Compose faz sozinho        | O que você faria sem ele                      |
| :--------------------------- | :-------------------------------------------- |
| Cria uma rede para o projeto | `docker network create` e `--network` em cada |
| Dá nome a cada container     | `--name`, combinado entre os comandos         |
| Lê o seu arquivo `.env`      | Repetir `-e VARIAVEL=valor` a cada subida     |

Três palavras que vão aparecer o tempo todo:

- **Serviço** — uma peça do ambiente (a API, o banco). Vira um container quando sobe.
- **Volume** — uma área de disco que **sobrevive** ao container ser destruído.
- **Healthcheck** — um comando que o Docker roda de tempos em tempos para perguntar ao próprio
  serviço se ele está bem. Você já viu um na Aula 10, dentro do `Dockerfile`.

---

## 🛠️ Capítulo 3: O primeiro `docker-compose.yml`

Vamos escrever a versão que **ainda tem o defeito**. Ela vai servir para você ver o problema
acontecer antes de aprender o conserto — do mesmo jeito que a Aula 11 mediu o desligamento
errado antes de corrigi-lo.

Crie o arquivo `docker-compose.yml` **na raiz do projeto**, ao lado do `Dockerfile`:

```yaml
services:
  api:
    build: .
    ports:
      - '${PORT:-3333}:3333'
    environment:
      NODE_ENV: production
      HOST: 0.0.0.0
      PORT: 3333
    depends_on:
      - mysql

  mysql:
    image: mysql:8.4
    ports:
      - '${MYSQL_PORT:-3306}:3306'
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - dados-mysql:/var/lib/mysql

volumes:
  dados-mysql:
```

### Linha por linha

| Trecho                          | O que faz                                                           |
| :------------------------------ | :------------------------------------------------------------------ |
| `services:`                     | Abre a lista de peças do ambiente                                   |
| `build: .`                      | Este serviço se constrói a partir do `Dockerfile` da pasta atual    |
| `image: mysql:8.4`              | Este serviço usa uma imagem pronta, baixada do registro público     |
| `ports: - '3333:3333'`          | Publica a porta do container na sua máquina                         |
| `${PORT:-3333}`                 | Usa a variável `PORT` do seu `.env`; se ela não existir, usa `3333` |
| `environment:`                  | Variáveis que o container recebe ao subir                           |
| `depends_on:`                   | Ordem: o `mysql` sobe antes da `api`                                |
| `volumes: - dados-mysql:/var/…` | Liga o volume nomeado à pasta onde o MySQL guarda os dados          |
| `volumes:` (no fim, sem recuo)  | Declara o volume nomeado que os serviços usam                       |

### Três ausências que são decisão, e não esquecimento

| O que **não** está no arquivo | Por quê                                                                                                                                        |
| :---------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------- |
| `version:` no topo            | Você vai ver isso em exemplos antigos pela internet. A especificação atual do Compose **ignora** esse campo e avisa que ele é obsoleto.        |
| `container_name:`             | O Compose nomeia os containers a partir do nome da pasta. Cravar o nome à mão cria colisão no dia em que o mesmo arquivo subir em outro lugar. |
| Uma seção `networks:`         | O Compose já cria uma rede para o projeto e liga os dois serviços nela. Escrever à mão é ruído que esconde a lição.                            |

### As variáveis do banco

Abra o seu `.env.example` e acrescente o bloco do banco de dados no fim. O arquivo completo está
no Capítulo 8; por enquanto, as quatro variáveis novas mais a porta:

```bash
MYSQL_ROOT_PASSWORD=troque-esta-senha
MYSQL_DATABASE=curso_api
MYSQL_USER=curso_api
MYSQL_PASSWORD=troque-esta-senha-tambem
MYSQL_PORT=3306
```

Depois copie os mesmos valores para o seu `.env` — que é o arquivo que o Compose lê de verdade,
e que **não** é versionado.

> [!IMPORTANT]
> Estas variáveis **não** entram no `src/shared/env/env.schema.ts`, e isso vai contra o seu
> instinto depois da Aula 04. O motivo: aquele schema é o contrato do que a **API** precisa — e
> a API ainda não fala com o banco. Quem consome essas variáveis é o serviço `mysql`, pelo
> Compose. O `DATABASE_URL` e a validação dele nascem na Aula 13, junto com quem os lê.
>
> Não é exceção à regra de "nada antes da hora": é a regra sendo aplicada. O que muda é
> **quem** lê a variável.

Repare também no `MYSQL_USER`: a aplicação nunca usa o administrador do banco. É o mesmo
raciocínio do `USER node` da Aula 10 — funciona dos dois jeitos, e um deles amplia o estrago de
qualquer falha.

---

## ⏱️ Capítulo 4: "Up" não é "pronto"

Agora suba o ambiente pela primeira vez e cronometre:

```bash
docker compose up -d
docker compose ps
```

O `-d` devolve o terminal (o `d` é de _detached_). O `ps` mostra os dois containers de pé:

```
NAME                            IMAGE                        STATUS
curso_api-api-1    curso_api-api   Up 2 seconds
curso_api-mysql-1  mysql:8.4                    Up 2 seconds
```

Os dois dizem `Up`. Parece pronto. **Não está.**

### A medição

Do lado de dentro da rede — que é o ponto de vista da API —, tentando alcançar o banco a cada
200 ms desde o instante anterior ao `up`:

```
[     1ms] t0 — antes do `docker compose up -d`
[  1132ms] `docker compose up -d` retornou (codigo 0)
[  1429ms] container da api: running (mysql: running)
[  1820ms] de dentro da rede: RECUSADA ECONNREFUSED
[ 19884ms] de dentro da rede: HANDSHAKE 8.4.11
```

Leia a terceira e a última linha juntas: **os dois containers estavam `running` 18 segundos
antes de o banco aceitar a primeira conexão.**

Uma API que subisse junto e tentasse conectar de imediato tomaria erro de conexão, com o banco
perfeitamente saudável ao lado. E o log diria "conexão recusada", que é a mensagem mais
enganosa possível: parece configuração errada, e é só pressa.

### Por que demora tanto

Na **primeira** subida, o MySQL não está apenas ligando: ele está **criando a pasta de dados**,
o banco, o usuário e as permissões. Nas subidas seguintes, com o volume já inicializado, o
mesmo trecho ficou assim:

```
[   924ms] container do mysql: running
[  6180ms] o banco aceitou conexao
```

Cinco segundos em vez de dezoito. **A janela diminui, mas não some** — e quem conectar dentro
dela toma erro do mesmo jeito.

> [!WARNING]
> Existe um jeito de medir isso que **mente**: testar a porta publicada na sua máquina.
>
> ```
> [  1493ms] porta TCP aceitou a conexao
> [ 19707ms] MySQL respondeu o handshake
> ```
>
> A porta "aceitou" em 1,5 s porque quem aceitou foi o **encaminhador de portas do Docker**, e
> não o MySQL. O cliente não recebe recusa: recebe **18 segundos de silêncio**, que é bem pior
> de diagnosticar.
>
> A lição é a mesma da Aula 10 com o `unhealthy`: um sinal só vale se ele estiver medindo o que
> você acha que ele mede.

---

## 🩺 Capítulo 5: A correção — perguntar ao serviço, não ao container

O `depends_on` na forma simples só promete uma coisa: **iniciar o `mysql` antes da `api`**. Ele
não sabe nada sobre o banco estar pronto — nem teria como saber, porque "pronto" significa
coisas diferentes para cada programa.

Quem sabe responder isso é o próprio serviço. É para isso que existe o `healthcheck`.

Altere o `docker-compose.yml` em dois lugares. Primeiro, o serviço `mysql` ganha um
`healthcheck`:

```yaml
    healthcheck:
      test: ['CMD', 'mysqladmin', 'ping', '-h', '127.0.0.1', '--silent']
      interval: 5s
      timeout: 3s
      start_period: 30s
      retries: 10
```

E o `depends_on` da `api` deixa de ser uma lista e passa a ter uma condição:

```yaml
    depends_on:
      mysql:
        condition: service_healthy
```

### O que cada campo faz

| Campo          | O que significa                                                            |
| :------------- | :------------------------------------------------------------------------- |
| `test`         | O comando que decide. Sai com 0 = saudável; qualquer outro código = doente |
| `interval`     | De quanto em quanto tempo perguntar                                        |
| `timeout`      | Quanto esperar por uma resposta antes de considerar aquela pergunta falha  |
| `start_period` | Uma folga inicial: falha aqui dentro **não** conta contra o serviço        |
| `retries`      | Quantas falhas seguidas, já fora da folga, para declarar `unhealthy`       |

Duas decisões neste `test` que valem mais que a sintaxe:

**Por que `-h 127.0.0.1`.** O `mysqladmin ping` sem esse trecho pergunta pelo soquete local, que
é um caminho que a API **não** usa. Verificar por uma porta que o cliente real não usa é
verificar outra coisa. Aqui, a pergunta vai pela rede, do mesmo jeito que a API vai perguntar.

**Por que `start_period: 30s`.** A primeira subida levou cerca de 20 segundos só para
inicializar a pasta de dados. Sem a folga, o container seria declarado doente por estar fazendo
exatamente o que devia fazer.

### Meça de novo

Derrube tudo, apagando também o volume, para repetir a **primeira** subida:

```bash
docker compose down -v
docker compose up -d
```

Agora o `up` demora — e isso é a correção funcionando, não uma lentidão nova:

```
[     0ms] t0 — antes do `docker compose up -d`
[  1005ms] container do mysql: running (saude: starting)
[ 21295ms] container do mysql: HEALTHY
[ 21700ms] container da api: running
[ 21853ms] `docker compose up -d` retornou (codigo 0)
[ 21864ms] de dentro da rede: HANDSHAKE 8.4.11
```

Compare com o Capítulo 4:

| Momento                       | Sem `service_healthy` | Com `service_healthy` |
| :---------------------------- | :-------------------- | :-------------------- |
| A API começa a rodar          | **1,4 s**             | 21,7 s                |
| O banco aceita conexão        | 19,9 s                | 21,3 s                |
| Conexão recusada pelo caminho | **sim, aos 1,8 s**    | **nenhuma**           |

O tempo total até tudo estar de pé é praticamente o mesmo. O que mudou é que **a API não existe
durante a janela em que o banco não responde** — então não há um único segundo em que ela possa
tentar e falhar.

Com o volume já inicializado, a mesma medição dá cerca de **7 segundos** no total.

> [!CAUTION]
> `healthcheck` **informa**; quem age é quem estiver esperando por ele. Um `mysql` marcado
> `unhealthy` continua rodando, exatamente como você viu na Aula 10. A diferença é que agora
> existe alguém escutando esse rótulo: o `condition: service_healthy` da `api`.
>
> Rótulo sem ninguém escutando não protege nada. Esta é a mesma família das lições das Aulas 09,
> 10 e 11 — vale a pena reler a tabela no fim desta aula.

---

## 🔌 Capítulo 6: Nome de serviço é endereço

Com o ambiente de pé, pergunte de dentro do container da API quem é `mysql`:

```bash
docker compose exec api getent hosts mysql
```

```
172.18.0.2      mysql
```

Ninguém configurou IP. O Compose criou uma rede para o projeto, colocou os dois containers
nela, e dentro dessa rede **o nome do serviço é o endereço**.

É a resposta direta ao trabalho manual da Aula 11: lá, você criou a rede e combinou os nomes na
mão para que o `nginx` achasse a API. Aqui, isso é consequência de ter escrito `mysql:` no
arquivo.

Confirme também que o banco responde de verdade:

```bash
docker compose exec mysql mysql -uroot -ptroque-esta-senha -e "SELECT 1 AS ok; SHOW DATABASES LIKE 'curso_api';"
```

```
ok
1
Database (curso_api)
curso_api
```

O banco declarado em `MYSQL_DATABASE` foi criado sozinho, na primeira subida.

E a API continua respondendo, do jeito que a Aula 10 deixou:

```bash
curl -i http://localhost:3333/health
curl -i http://localhost:3333/documentation
```

`200` na primeira, `404` na segunda — o Compose sobe a imagem de produção, e em produção a
documentação continua desligada.

### O comando de trabalho continua sendo o `npm run dev`

Isto costuma confundir, então vale dizer em voz alta:

| Comando             | Para quê                                                  |
| :------------------ | :-------------------------------------------------------- |
| `docker compose up` | Provar que o ambiente inteiro sobe em qualquer máquina    |
| `npm run dev`       | Trabalhar: recarregamento automático a cada arquivo salvo |

No dia a dia você sobe o Compose e **deixa o banco rodando**, enquanto edita o código com o
`npm run dev` na sua máquina, apontando para a porta publicada do MySQL. O recarregamento
automático que você tem desde a Aula 01 não se perde em lugar nenhum — ele apenas não vem do
container.

> [!NOTE]
> Você vai encontrar por aí um arquivo chamado `docker-compose.override.yml`, que serve para
> rodar o modo de desenvolvimento **dentro** do container. Ele existe, é legítimo, e não é o
> assunto desta aula: seriam dois arquivos para explicar de uma vez.

---

## 💾 Capítulo 7: `down` e `down -v` são comandos diferentes

Derrube o ambiente:

```bash
docker compose down
```

```
 Container curso_api-mysql-1  Removed
 Network curso_api_default    Removed
```

Os containers foram destruídos. Agora veja o que **não** foi:

```bash
docker volume ls
```

```
curso_api_dados-mysql
```

O volume continua lá, com o banco inteiro dentro. Suba de novo e os dados estarão no lugar —
foi por isso que o serviço `mysql` recebeu `volumes:` lá no Capítulo 3. Sem volume nomeado, os
dados moram dentro do container, e o container é descartável por natureza.

**O `-v` muda tudo:**

```bash
docker compose down -v    # apaga o volume, e com ele o banco inteiro
```

Não há confirmação. Não há aviso. Não há como desfazer.

> [!CAUTION]
> É por isso que o `npm run compose:down` **não** leva o `-v`. Um atalho existe para ser
> digitado no automático, e comando que destrói dado não pode ser digitado no automático.
>
> O `-v` continua existindo e às vezes é exatamente o certo — como no Capítulo 5, para repetir a
> primeira subida. A diferença é que ele se digita por inteiro, olhando.

### Os dois atalhos

Acrescente ao `package.json`, junto dos `docker:*` da Aula 10:

```json
    "compose:up": "docker compose up -d",
    "compose:down": "docker compose down",
```

Só dois, porque só dois se digitam todo dia. O arquivo completo está no Capítulo 8.

---

## 📄 Capítulo 8: os arquivos, por inteiro

Confira cada um contra o seu. Estes são os quatro arquivos que esta aula alterou.

### `docker-compose.yml`

Esta é a versão final, com o `healthcheck` do Capítulo 5 e os comentários que explicam cada
decisão:

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
```

Lembre-se de copiar as variáveis novas para o seu `.env` também. O `.env.example` é o modelo
versionado; o `.env` é o que o Compose lê.

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
    "compose:up": "docker compose up -d",
    "compose:down": "docker compose down",
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
    "pino-pretty": "^13.1.3",
    "prettier": "^3.9.6",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0",
    "vitest": "^4.1.10"
  }
}
```

### `README.md`

Quem clona o projeto precisa saber que existe um ambiente inteiro esperando por um comando:

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
````

---

## ✅ Como saber que deu certo

### 1. A verificação completa passa

```bash
npm run check
```

Precisa terminar sem erro. Ela não toca no Docker — mas se o `package.json` ficou com vírgula
sobrando, é aqui que você descobre.

### 2. O ambiente sobe do zero

```bash
docker compose down -v
npm run compose:up
docker compose ps
```

Os dois serviços em `Up`, e o `mysql` marcado `(healthy)`.

### 3. O banco responde

```bash
docker compose exec mysql mysql -uroot -ptroque-esta-senha -e "SELECT 1 AS ok;"
```

Precisa devolver `ok` e `1`.

### 4. O nome do serviço resolve como endereço

```bash
docker compose exec api getent hosts mysql
```

Precisa devolver um IP seguido de `mysql`.

### 5. A API responde de dentro do Compose

```bash
curl -i http://localhost:3333/health
```

`HTTP/1.1 200 OK` e `{"status":"ok"}`.

### 6. O dado sobrevive ao `down`

```bash
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api -e "CREATE TABLE teste (id INT); INSERT INTO teste VALUES (1);"
npm run compose:down
npm run compose:up
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api -e "SELECT * FROM teste;"
```

A linha `1` precisa continuar lá.

### 7. E o desligamento da Aula 11 continua valendo

```bash
docker compose down
echo $?
```

Código de saída **0**: ninguém levou `SIGKILL`.

---

## 🚨 Erros Comuns

### "`variable is not set. Defaulting to a blank string`"

O Compose não achou a variável no seu `.env`. Confira se você copiou o bloco novo do
`.env.example` para lá — e lembre que o `.env` **não** é versionado, então ele não veio pronto.

### "`Bind for 0.0.0.0:3306 failed: port is already allocated`"

Já existe um MySQL na sua máquina ocupando a 3306. Troque o `MYSQL_PORT` no `.env` para `3307`
e suba de novo. Nada **dentro** do Compose precisa saber disso: lá dentro os serviços se falam
pelo nome, na porta 3306 de sempre.

**Este erro é o bom caso.** O ruim é o silencioso: se a porta estivesse ocupada e o Docker não
reclamasse, o seu `mysql -h 127.0.0.1` conversaria com o **outro** banco, e você passaria uma
tarde tentando entender por que a tabela criada não aparece.

### "`Access denied for user 'root'@'localhost'`"

A senha do comando não bate com a do `.env`. Repare também que as variáveis `MYSQL_*` só têm
efeito na **primeira** subida, quando a pasta de dados é criada: trocar a senha no `.env` depois
disso não muda a senha do banco que já existe. Para começar do zero, `docker compose down -v`.

### "Mudei o `healthcheck` e nada mudou"

O `healthcheck` é lido quando o container é **criado**, não a cada `up`. Rode
`docker compose up -d --force-recreate`, ou derrube com `down` antes.

### "O `docker compose up` fica parado muito tempo"

Se for a primeira subida, é o esperado: com `condition: service_healthy`, o Compose está
esperando o banco ficar pronto — cerca de 20 segundos aqui. Acompanhe com
`docker compose logs -f mysql` em outro terminal e veja o MySQL inicializando.

---

## 🏋️ Exercícios

### 1. Prove que o volume é o que guarda o dado

Crie uma tabela, derrube com `down` (sem `-v`), suba e confirme que ela sobreviveu. Depois
repita a sequência usando `down -v` e explique, em uma frase, o que mudou.

### 2. Troque a porta publicada do banco

Altere `MYSQL_PORT` no `.env` para `3307` e suba de novo. Confirme que a API continua
funcionando **sem nenhuma alteração** e explique por quê.

### 3. Veja o problema voltar

Remova temporariamente o `condition: service_healthy` (voltando ao `depends_on` em lista), rode
`docker compose down -v && docker compose up -d` e meça quanto tempo o banco leva para aceitar
conexão depois de o container aparecer como `Up`. Depois desfaça.

### 4. Traga a cena da Aula 11 para dentro do Compose

Transforme o `nginx` da Aula 11 em um terceiro serviço deste arquivo, com o volume de
configuração e sem nenhum `docker network create`. Confirme que a API continua registrando o IP
real do cliente com `TRUST_PROXY=1`.

Os gabaritos comentados estão em [`exercicios/12-gabarito.md`](./exercicios/12-gabarito.md).

---

## 🧠 A família de lições que esta aula fecha

Quatro aulas seguidas desarmaram a mesma confusão, em quatro roupas diferentes:

| Aula | A confusão que ela desarma                                                       |
| :--- | :------------------------------------------------------------------------------- |
| 09   | Helmet e CORS **pedem** ao navegador; não são regra que a API imponha            |
| 10   | `unhealthy` é **rótulo**, não ação: o container continua recebendo requisição    |
| 11   | O prazo de desligamento é **acordo entre dois lados**, e vale o menor deles      |
| 12   | `depends_on` espera o **container**, não o **serviço** — e os dois não coincidem |

Em todas, o erro é o mesmo: confundir **o sinal** com **a garantia**. Um rótulo, um pedido ou
uma ordem de partida só valem alguma coisa quando existe alguém do outro lado agindo em cima
deles.

---

## 📌 O que fica para a próxima aula

O banco está de pé, e ninguém instalou MySQL na mão. Só que a API ainda **não fala com ele**:
não há `DATABASE_URL`, não há tabela de negócio, não há uma linha de código que abra conexão.

A **Aula 13** liga os dois com o Prisma, cria a camada Repository e traz as **migrations** —
onde cada alteração de tabela vira um arquivo versionado no Git, ao lado do código que depende
dela.

Fica registrado o que **não** foi resolvido aqui: a contagem do limite de requisições continua
na memória do processo. Com o Compose no ar, o obstáculo caiu — um armazenamento compartilhado
agora são poucas linhas neste arquivo. O que impede não é mais a infraestrutura: é que misturar
isso com a primeira aula de Compose colocaria dois assuntos disputando a sua atenção.
