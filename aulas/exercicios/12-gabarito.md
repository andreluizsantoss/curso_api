# 🎼 Gabarito — Aula 12: Docker Compose

> Confira só depois de tentar. Os quatro exercícios foram executados antes de respondidos, e
> os números abaixo saíram dessa execução.

---

## Exercício 1 — Prove que o volume é o que guarda o dado

> Crie uma tabela, derrube com `down` (sem `-v`), suba e confirme que ela sobreviveu. Depois
> repita a sequência usando `down -v` e explique, em uma frase, o que mudou.

### Os comandos

```bash
docker compose up -d
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api \
  -e "CREATE TABLE teste (id INT); INSERT INTO teste VALUES (1);"

# Primeira rodada: sem -v
docker compose down
docker compose up -d
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api -e "SELECT * FROM teste;"
```

```
id
1
```

```bash
# Segunda rodada: com -v
docker compose down -v
docker compose up -d
docker compose exec mysql mysql -uroot -ptroque-esta-senha curso_api -e "SELECT * FROM teste;"
```

```
ERROR 1146 (42S02) at line 1: Table 'curso_api.teste' doesn't exist
```

### A frase

O `down` destrói os **containers**; o `down -v` destrói também o **volume**, que é onde os
dados moram. Container é descartável por natureza — se o dado estivesse dentro dele, cada
`down` seria uma perda total.

### O detalhe que costuma passar batido

Repare que o banco `curso_api` **voltou a existir** depois do `down -v`, vazio. Ele é
criado pelo `MYSQL_DATABASE` toda vez que a pasta de dados nasce do zero. Ou seja: o erro
acima não é "o banco sumiu", é "o banco é outro". Confundir os dois leva a diagnósticos
errados quando isso acontecer em um ambiente compartilhado.

---

## Exercício 2 — Troque a porta publicada do banco

> Altere `MYSQL_PORT` no `.env` e suba de novo. Confirme que a API continua funcionando **sem
> nenhuma alteração** e explique por quê.

### O que foi feito

```bash
# no .env
MYSQL_PORT=3308

docker compose up -d
docker compose ps --format '{{.Service}} {{.Ports}}'
```

```
api    0.0.0.0:3333->3333/tcp
mysql  0.0.0.0:3308->3306/tcp
```

A API respondeu normalmente:

```
200 /health
```

E, de dentro da rede do projeto, o banco continua atendendo na **3306** de sempre:

```
handshake de mysql:3306 -> 8.4.11
```

### Por quê

Porque as duas portas são coisas diferentes:

| Lado                      | Porta                         | Quem usa                                |
| :------------------------ | :---------------------------- | :-------------------------------------- |
| Dentro da rede do projeto | `3306`, sempre                | A API, falando com o serviço `mysql`    |
| Publicada na sua máquina  | O que estiver em `MYSQL_PORT` | Você, com o `npm run dev` ou um cliente |

A linha `'${MYSQL_PORT:-3306}:3306'` se lê da direita para a esquerda: **a porta de dentro
não muda**; o que muda é em qual porta da sua máquina ela aparece. Publicar existe para o
mundo de fora do Compose — lá dentro, ninguém precisa disso.

É por isso, também, que trocar essa porta é a solução para o erro `port is already
allocated`: nada dentro do ambiente precisa saber.

---

## Exercício 3 — Veja o problema voltar

> Remova temporariamente o `condition: service_healthy`, rode
> `docker compose down -v && docker compose up -d` e meça quanto tempo o banco leva para
> aceitar conexão depois de o container aparecer como `Up`.

### O resultado medido

Com o `depends_on` na forma de lista, do lado de dentro da rede, tentando conectar a cada
200 ms:

```
[  1132ms] `docker compose up -d` retornou (codigo 0)
[  1429ms] container da api: running (mysql: running)
[  1820ms] de dentro da rede: RECUSADA ECONNREFUSED
[ 19884ms] de dentro da rede: HANDSHAKE 8.4.11
```

**Cerca de 18 segundos** entre "o container está `Up`" e "o banco atende", na primeira
subida. Com o volume já inicializado, o mesmo vão ficou em torno de 5 segundos.

### Como medir sem escrever programa nenhum

Se você não quiser montar um laço, dá para ver o mesmo fenômeno com dois comandos:

```bash
docker compose down -v && docker compose up -d && docker compose ps
docker compose exec mysql mysqladmin ping -h 127.0.0.1 --silent ; echo $?
```

Logo depois do `up`, o `ping` falha com código diferente de zero, enquanto o `ps` mostra
`Up`. Repita o `ping` de tempos em tempos até ele passar: o intervalo entre as duas coisas é
a resposta do exercício.

### O que isso ensina

`Up` responde "o container existe". `healthy` responde "o programa lá dentro está atendendo".
São perguntas diferentes, e só a segunda serve para decidir se dá para conectar.

Depois de medir, **desfaça**: volte o `condition: service_healthy`. E lembre que
`healthcheck` e `depends_on` são lidos quando o container é **criado** — para valer, use
`docker compose up -d --force-recreate` ou derrube antes.

---

## Exercício 4 — Traga a cena da Aula 11 para dentro do Compose

> Transforme o `nginx` da Aula 11 em um terceiro serviço, sem nenhum `docker network create`.
> Confirme que a API continua registrando o IP real do cliente com `TRUST_PROXY=1`.

### O serviço

```yaml
  proxy:
    image: nginx:alpine
    ports:
      - '8080:8080'
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
```

E a `api` ganha a variável:

```yaml
    environment:
      NODE_ENV: production
      HOST: 0.0.0.0
      PORT: 3333
      TRUST_PROXY: 1
```

O `nginx.conf` é o mesmo da Aula 11, sem uma vírgula de diferença — o `proxy_pass` continua
apontando para `http://api:3333`, porque `api` **já era** o nome do serviço.

### O resultado medido

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/health   # 200
docker compose exec api getent hosts proxy                              # 172.18.0.4  proxy
docker compose logs api | grep '"url":"/health"' | tail -1
```

| `TRUST_PROXY` | `remoteAddress` registrado | O que é                   |
| :------------ | :------------------------- | :------------------------ |
| `false`       | `172.18.0.4`               | O IP do **próprio proxy** |
| `1`           | `172.18.0.1`               | O IP real de quem chamou  |

Os mesmos dois resultados da Aula 11 — obtidos agora **sem** criar rede, sem `--name` e sem
combinar nomes entre comandos.

### O que desapareceu

| Aula 11, na mão                      | Aula 12, no arquivo             |
| :----------------------------------- | :------------------------------ |
| `docker network create rede-aula11`  | _(nada — o Compose cria)_       |
| `--name api`, `--name proxy`         | Os nomes dos serviços           |
| Dois `docker run` na ordem certa     | `docker compose up -d`          |
| Caminho absoluto para o `nginx.conf` | Um caminho relativo, versionado |

Esse é o ponto inteiro da aula: o que antes vivia no histórico do terminal agora vive no
repositório, e sobrevive ao esquecimento.

> [!NOTE]
> Se você deixou o `proxy` no arquivo, lembre de tirá-lo depois — ou de mantê-lo em um
> arquivo separado, que o Compose aceita com `-f`. O `docker-compose.yml` da aula tem dois
> serviços de propósito: o proxy é uma cena de demonstração, não parte do ambiente.
