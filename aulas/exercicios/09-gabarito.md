# 🛡️ Gabarito — Aula 09: Segurança HTTP

> Confira só depois de tentar. Os quatro exercícios foram executados antes de respondidos, e
> três deles dão um resultado que contraria o que a maioria das pessoas espera.

---

## Exercício 1 — O CORS não protege o que você acha que protege

> Com `CORS_ORIGINS=http://localhost:5173` configurado, chame a API passando o cabeçalho
> `Origin: https://site-qualquer.example`. A requisição foi bloqueada?

### O que acontece

| Origem enviada                  | Status | `access-control-allow-origin` | Corpo             |
| :------------------------------ | :----: | :---------------------------- | :---------------- |
| `http://localhost:5173`         |  200   | `http://localhost:5173`       | `{"status":"ok"}` |
| `https://site-qualquer.example` |  200   | _(ausente)_                   | `{"status":"ok"}` |
| _(nenhuma)_                     |  200   | _(ausente)_                   | `{"status":"ok"}` |

### A resposta: **não foi bloqueada. E a resposta veio inteira.**

Olhe a última coluna. A origem não autorizada recebeu **status 200 e o corpo completo**. A
única diferença é um cabeçalho que **não** veio.

### Por que isso acontece

CORS não é um porteiro na entrada da sua API. É a sua API **dando ou não dando uma
autorização** que o navegador exige antes de entregar a resposta ao JavaScript que a pediu.

A sequência real é esta:

1. O script de outro site pede a URL.
2. O navegador faz a requisição.
3. **A sua API responde normalmente**, com corpo e tudo.
4. O navegador procura o `access-control-allow-origin` na resposta.
5. Não achou a origem dele ali → **o navegador** joga a resposta fora e registra um erro no
   console do site que pediu.

Quem bloqueia é o **passo 5**, e ele acontece na máquina do cidadão, não no seu servidor.

### O que isso diz sobre CORS como proteção do servidor

Que ele não é uma. Se o `ataque.js` da aula mandasse um cabeçalho `Origin`, ele receberia a
resposta do mesmo jeito — porque ele não é um navegador, e não existe navegador nenhum ali
para jogar a resposta fora.

> **CORS protege o cidadão, não o servidor.** Ele impede que um site qualquer use a sessão do
> cidadão para chamar a sua API às escondidas. Contra um programa que fale HTTP direto, ele
> não faz absolutamente nada — e nunca teve a pretensão de fazer.

E é exatamente por isso que o rate limit existe: ele é a única das três proteções desta aula
que age no seu servidor, sem depender de ninguém colaborar.

---

## Exercício 2 — Descubra onde o limite é contado

> Estoure o limite, reinicie o servidor e tente de novo imediatamente. Você foi bloqueado?

### O que acontece

```
mesma instância, depois de estourar: 429
instância NOVA, mesmo IP           : 200
```

### A resposta: **a contagem morre junto com o processo**

Ela mora na **memória** do processo que está rodando. Reiniciar o servidor zera tudo: quem
estava bloqueado volta a ser bem-vindo no mesmo instante.

### O que isso significa quando a API roda em mais de um processo

Aqui está a parte que vale a pena guardar.

Em produção, uma API raramente roda em um processo só. São vários — várias cópias atrás de um
balanceador, para aguentar carga e não cair inteira quando uma delas trava.

E cada cópia tem **a sua própria memória**, com **a sua própria contagem**.

| Processos rodando | Limite que você escreveu | Limite real, na prática |
| :---------------: | :----------------------: | :---------------------: |
|         1         |      100 por minuto      |     100 por minuto      |
|         4         |      100 por minuto      |   **400 por minuto**    |
|        10         |      100 por minuto      |   **1000 por minuto**   |

O atacante não precisa fazer nada esperto: as requisições dele se espalham entre as cópias
sozinhas, e cada uma conta do zero.

### Como se resolve

Guardando a contagem **fora** do processo, num lugar que todas as cópias enxergam — um Redis,
tipicamente. O `@fastify/rate-limit` aceita isso pela opção `redis`.

Não fizemos aqui porque o projeto ainda não tem Redis, e a regra do curso é não instalar nada
antes de usar de verdade. Mas o limite existe, e agora você sabe procurá-lo.

> **A pergunta que fica para toda configuração de segurança:** _"isto continua valendo quando
> houver mais de uma cópia rodando?"_ Contagem em memória, sessão em memória e cache em
> memória falham todos pelo mesmo motivo.

---

## Exercício 3 — Isente a rota e veja o que se perde

> Troque `config: { rateLimit: RATE_LIMIT_HEALTH }` por `config: { rateLimit: false }`. Qual
> teste falha, e ele estava certo em falhar?

### O que acontece

```
 ❯ src/shared/security/security.spec.ts (9 tests | 1 failed)
     × mas o /health também tem teto — ele é maior, não infinito

AssertionError: expected 200 to be 429

- Expected
+ Received

- 429
+ 200
```

### A resposta: sim, e ele é o único que percebe

Repare no que **não** falhou. O teste _"deixa o monitoramento consultar /health além do limite
global"_ continua passando — porque sem limite nenhum o monitoramento realmente passa. A
mudança melhorou aquele teste, do ponto de vista dele.

Só o teste do teto percebe que a porta ficou escancarada.

### A história desse teste, que vale mais que o exercício

Este teste nasceu **errado**. A primeira versão dele era assim:

```typescript
// Versão original — não servia para nada
expect(RATE_LIMIT_HEALTH.max).toBeGreaterThan(RATE_LIMIT_GLOBAL.max)
expect(Number.isFinite(RATE_LIMIT_HEALTH.max)).toBe(true)
```

Ela compara duas constantes. E o comentário ao lado afirmava, com todas as letras, que trocar
o limite da rota por `false` faria o teste cair.

**Não fazia.** Trocar `RATE_LIMIT_HEALTH` por `false` no arquivo de rotas não altera constante
nenhuma: os dois números continuam lá, um maior que o outro, e o teste continua verde sobre
uma rota sem teto.

Isso só apareceu quando o exercício foi **executado** para escrever este gabarito.

A correção foi trocar a comparação por comportamento: mandar 241 requisições e exigir o 429.

> **A lição é maior que o assunto da aula:** um teste que verifica a **configuração** em vez do
> **comportamento** dá a sensação de proteção sem a proteção. Ele passa exatamente nos casos em
> que você mais precisaria dele.
>
> A pergunta que separa os dois: _"se eu quebrar o que este teste protege, ele fica vermelho?"_
> Se você não souber responder, quebre de propósito e veja — foi o que fizemos aqui.

---

## Exercício 4 — O que a resposta conta sobre você

> Procure um cabeçalho que revele qual servidor ou framework está por trás. Você encontrou?

### O que acontece

A lista completa dos cabeçalhos de uma resposta:

```
content-security-policy         cross-origin-opener-policy
cross-origin-resource-policy    origin-agent-cluster
referrer-policy                 strict-transport-security
x-content-type-options          x-dns-prefetch-control
x-download-options              x-frame-options
x-permitted-cross-domain-policies    x-xss-protection
vary                            x-ratelimit-limit
x-ratelimit-remaining           x-ratelimit-reset
content-type                    content-length
date                            connection
```

### A resposta: **não existe. E isso é a resposta certa.**

Não há `server`, não há `x-powered-by`, não há versão de nada. O Fastify simplesmente não se
anuncia.

Isso não é sorte: muitos servidores e frameworks mandam esses cabeçalhos por padrão, e a
primeira recomendação de qualquer guia de segurança é removê-los. Aqui não houve o que
remover.

**Por que isso importa:** um cabeçalho `x-powered-by` com nome e versão entrega, de graça, a
lista de vulnerabilidades conhecidas daquela versão. Não é o fim do mundo — segurança que
depende de esconder o nome do software é fraca —, mas é informação dada sem necessidade, e
quem varre a internet procurando alvos usa exatamente isso para filtrar.

> Este exercício é um caso de **procurar e não achar**. Vale tanto quanto achar: agora você
> sabe qual cabeçalho procurar nas outras APIs em que trabalhar.

### E os `x-ratelimit`?

Esses você achou, e eles contam bastante:

| Cabeçalho               | O que diz                           |
| :---------------------- | :---------------------------------- |
| `x-ratelimit-limit`     | Quantas requisições cabem na janela |
| `x-ratelimit-remaining` | Quantas ainda restam                |
| `x-ratelimit-reset`     | Em quantos segundos o crédito volta |

**O lado bom:** um cliente bem-comportado consegue se regular sozinho. Ele vê que restam 3
requisições e diminui o ritmo, em vez de bater no muro e tomar 429. É a diferença entre um
integrador que trabalha com você e um que descobre o limite na marra.

**O lado ruim:** eles também contam ao atacante exatamente onde está o limite, quanto falta e
quando reinicia — informação suficiente para calibrar o ataque para ficar **logo abaixo** do
teto e nunca ser barrado.

E repare num detalhe: chame `/health` e depois `/health/ready`, e compare o
`x-ratelimit-limit`. Ele muda de **240** para **100** — a resposta acabou de revelar que uma
das rotas tem tratamento especial.

> **Não há resposta certa universal aqui**, e é isso que faz o exercício valer. É uma troca:
> ergonomia para quem integra de boa-fé contra informação para quem não integra. A maioria das
> APIs públicas escolhe manter, porque o ganho com clientes legítimos é diário e o custo é
> marginal — quem vai atacar descobre o limite em dois minutos de tentativa, com ou sem
> cabeçalho.
>
> O que **não** vale é a escolha acontecer sem ninguém perceber que existia uma.

---

## 🎯 O fio que liga os quatro

| Exercício | O que parecia                        | O que é                                                      |
| :-------- | :----------------------------------- | :----------------------------------------------------------- |
| 1         | CORS bloqueia quem não está na lista | A API responde tudo; quem descarta é o navegador do cidadão  |
| 2         | O limite é de 100 por minuto         | É de 100 **por processo**, e reinício zera                   |
| 3         | O teste protegia a rota              | Ele comparava constantes e passaria sobre uma porta aberta   |
| 4         | Deve haver um cabeçalho revelador    | Não há — e o que revela informação é outro, por outro motivo |

Os quatro são a mesma lição, de quatro ângulos: **em segurança, o que parece protegido e o que
está protegido são perguntas diferentes.** A única forma de saber qual é qual é quebrar de
propósito e olhar o que acontece.
