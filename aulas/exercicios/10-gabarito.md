# 🐳 Gabarito — Aula 10: Docker

> Confira só depois de tentar. Os quatro exercícios foram executados antes de respondidos, e
> dois deles têm um resultado que contraria o que a maioria das pessoas espera.

---

## Exercício 1 — O cache das camadas

> Altere uma linha em qualquer arquivo de `src/` e reconstrua a imagem. Observe quais etapas
> dizem `CACHED` e quais rodam de novo. Depois altere o `package.json` e reconstrua. Compare.

### Alterando `src/`

```
#7  [build 2/7] WORKDIR /app                              CACHED
#8  [build 3/7] COPY package.json package-lock.json ./    CACHED
#9  [build 4/7] RUN npm ci                                CACHED
#10 [build 5/7] COPY tsconfig.json tsconfig.build.json ./ CACHED
#11 [build 6/7] COPY src ./src                            (rodou)
#12 [build 7/7] RUN npm run build                         (rodou)
#13 [producao 4/5] RUN npm ci --omit=dev                  CACHED
```

**Tempo: 7 segundos.**

### Alterando `package.json`

```
#8  [build 4/7] RUN npm ci                                (rodou)
```

**Tempo: 46 segundos.**

### A resposta

**Seis vezes e meia mais lento**, e o motivo cabe em uma frase: o cache de uma camada só vale
enquanto **todas as camadas anteriores** também valerem.

O Docker guarda cada instrução como uma camada empilhada sobre a anterior. Quando uma muda,
ela e **tudo o que vem depois** são refeitos — não porque o Docker seja pessimista, mas
porque ele não tem como saber se o resultado de `npm ci` continuaria o mesmo depois de um
`package.json` diferente.

Alterando `src/`, a mudança acontece na etapa 6 de 7: as cinco anteriores, incluindo a
instalação inteira das dependências, são reaproveitadas. Alterando `package.json`, a mudança
acontece na etapa 3: dali para frente, tudo é refeito.

**É exatamente por isso que o `Dockerfile` tem dois `COPY` separados**, com os manifestos
antes do código. Se houvesse um `COPY . .` só, toda alteração de código cairia na etapa mais
antiga possível, e **todo build custaria 46 segundos**. A ordem das linhas não é estética: é
a diferença entre 7 e 46.

---

## Exercício 2 — O contexto de build

> Crie um `Dockerfile.teste` que use `COPY . .` e construa com e sem o `.dockerignore`,
> anotando o "transferring context" das duas vezes. Depois procure o `.env` nas duas imagens.

`Dockerfile.teste`:

```dockerfile
FROM node:24-slim
WORKDIR /app
COPY . .
CMD ["node", "dist/server.js"]
```

```bash
docker build -f Dockerfile.teste -t teste_com .
# renomeie o .dockerignore e construa de novo
docker build -f Dockerfile.teste -t teste_sem .
```

### O resultado medido

| Com `COPY . .`          | Contexto enviado | Tempo     | Imagem | `.env`  |
| :---------------------- | :--------------- | :-------- | :----- | :------ |
| **Com** `.dockerignore` | 103,56 kB        | 0,1s      | 329 MB | não     |
| **Sem** `.dockerignore` | **130,83 MB**    | **50,8s** | 524 MB | **sim** |

Verificando por dentro:

```bash
docker run --rm teste_sem sh -c "test -f /app/.env && echo VAZOU || echo NAO"
```

```
VAZOU
```

Também foram parar lá dentro o `node_modules` inteiro e o `.git` com todo o histórico do
projeto.

### A parte que surpreende

Se você fez esse teste com o `Dockerfile` **da aula** em vez do `Dockerfile.teste`, quase nada
mudou. Isso é real, e a explicação importa: o Docker moderno envia o contexto **sob demanda**.
Como o nosso `Dockerfile` pede arquivos por nome (`COPY src ./src`), ele nunca chega a pedir o
`node_modules` — o `.dockerignore` não tem o que impedir.

**Isso torna o `.dockerignore` dispensável?** Não, e a conclusão errada aqui é cara. Ele é a
rede de segurança para o dia em que alguém — você daqui a seis meses, ou um colega — escrever
`COPY . .`, que é o jeito mais comum de errar justamente porque é o que aparece na maioria dos
exemplos da internet. A proteção precisa existir **antes** do erro, não depois.

E o dano, quando acontece, não se desfaz: **um segredo que entra numa imagem está nela para
sempre**, em todas as cópias, mesmo que uma instrução posterior o apague — a camada anterior
continua lá, e quem tiver a imagem consegue lê-la.

---

## Exercício 3 — Dois containers de uma vez

> Suba dois containers da mesma imagem ao mesmo tempo, em portas diferentes, e prove que os
> dois respondem. O que isso diz sobre a relação entre imagem e container?

```bash
docker run -d --rm --name api_a -p 3333:3333 curso_api
docker run -d --rm --name api_b -p 3334:3333 curso_api
docker ps
```

```
api_b | 0.0.0.0:3334->3333/tcp | Up 6 seconds (healthy)
api_a | 0.0.0.0:3333->3333/tcp | Up 7 seconds (healthy)
```

```bash
curl http://localhost:3333/health/ready
curl http://localhost:3334/health/ready
```

```
porta 3333 -> 200 | uptime=12.313s | env=production
porta 3334 -> 200 | uptime=11.826s | env=production
```

### A resposta

Repare que **os dois lados do `-p` são diferentes entre si, mas o lado direito é igual**:
`3333:3333` e `3334:3333`. O número da direita é a porta **dentro** do container — e lá dentro
as duas APIs escutam na 3333, sem conflito nenhum, porque cada container tem a própria rede
isolada. Quem precisa ser único é o lado esquerdo, a porta da **sua** máquina.

Os `uptime` diferentes provam o resto: são **dois processos independentes**, saídos da mesma
imagem. Um não sabe da existência do outro. Derrubar um não afeta o outro.

É a receita e os bolos do Capítulo 2, agora medido. E é também a base de como se aguenta mais
tráfego em produção: não se "torna a API maior", sobem-se mais cópias dela.

---

## Exercício 4 — Doente de propósito

> Faça um container ficar `unhealthy` e observe o que acontece com ele em seguida. Ele para
> sozinho? Quem deveria agir?

```bash
docker run -d --rm --name quebrado --health-cmd "node -e \"process.exit(1)\"" \
  --health-interval=3s --health-retries=2 --health-start-period=1s \
  curso_api

docker inspect --format "{{.State.Health.Status}}" quebrado
```

Acompanhando:

```
t+4s  -> starting
t+8s  -> unhealthy
```

E então:

```bash
docker exec quebrado node -e "process.exit(0)"
```

```
(sem erro — o container continua vivo)
```

### A resposta

**Não, ele não para sozinho.** E essa é a parte que quase todo mundo entende errado.

`unhealthy` é um **rótulo**, não uma ação. O Docker prometeu perguntar de tempos em tempos e
anotar a resposta — e foi só isso que ele fez. O container continua rodando, continua com a
porta publicada e continua recebendo requisições normalmente.

**Quem deveria agir é quem estiver orquestrando os containers**: um Docker Compose com política
de reinício, um orquestrador em produção, ou o balanceador de carga, que pode parar de mandar
tráfego para uma cópia doente sem derrubá-la.

A lição prática: colocar um `HEALTHCHECK` na imagem **não** deixa a API auto-recuperável. Ele
produz a informação a partir da qual alguém decide. Confundir "está monitorado" com "está
protegido" é o mesmo erro conceitual da Aula 09, quando ficou claro que Helmet e CORS são
**pedidos ao navegador**, e não regras que a API impõe.

```bash
docker stop quebrado
```
