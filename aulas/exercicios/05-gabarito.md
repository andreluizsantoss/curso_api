# Gabarito — Aula 05

---

## 1. Prove que o log recebe o que a resposta esconde

Rota temporária, dentro do `buildApp()`, logo antes do `return app`:

```typescript
// ⚠️ TEMPORÁRIA — apagar depois do exercício.
app.get('/vazamento', async () => {
  throw new Error("Unknown column 'cpf' in field list: SELECT * FROM cidadaos")
})
```

**Na aba de resposta do REST Client:**

```json
{
  "statusCode": 500,
  "error": "Internal Server Error",
  "message": "Erro interno do servidor. A equipe já foi avisada."
}
```

**No terminal onde o `npm run dev` está rodando:**

```json
{
  "level": 50,
  "time": 1786800000000,
  "reqId": "req-1",
  "err": {
    "type": "Error",
    "message": "Unknown column 'cpf' in field list: SELECT * FROM cidadaos",
    "stack": "Error: Unknown column 'cpf' in field list: SELECT * FROM cidadaos\n    at ..."
  },
  "msg": "Erro não tratado durante a requisição"
}
```

**Onde está o stack trace:** só no terminal, dentro do campo `err.stack`.
**Onde está o nome da tabela:** no mesmo lugar, dentro de `err.message`.

Essa é a resposta que a aula inteira persegue. A informação **não foi perdida** — ela está
completa, com nome de tabela, nome de coluna e a linha exata do código onde o erro nasceu. O
que mudou foi o **canal**: ela vai para o log estruturado, que fica no servidor, e não para a
resposta HTTP, que vai para a internet.

Repare também no `"level": 50`. No Pino, 50 é o nível `error`. É por isso que o log é
estruturado: dá para pedir "me mostre tudo com level 50 da última hora" — algo impossível com
texto solto.

E repare no `reqId`. O Fastify numera cada requisição. Se o mesmo cidadão relatar "deu erro
às 14h32", dá para achar exatamente a requisição dele e ver o erro real por trás da mensagem
genérica que ele viu.

---

## 2. Um erro esperado de verdade

```typescript
// ⚠️ TEMPORÁRIA — apagar depois do exercício.
app.get('/protocolo/:numero', async (request) => {
  const { numero } = request.params as { numero: string }

  if (numero === '0') {
    throw new AppError('Protocolo não encontrado', 404)
  }

  return { numero, status: 'em andamento' }
})
```

O import no topo do `app.ts`:

```typescript
import { AppError } from './shared/errors/app-error.ts'
```

As duas requisições, acrescentadas ao `erros.http`:

```http
### TEMPORÁRIO — exercício 2: protocolo que não existe
GET {{host}}/protocolo/0

### TEMPORÁRIO — exercício 2: protocolo que existe
GET {{host}}/protocolo/42
```

**Disparando a primeira:**

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Protocolo não encontrado"
}
```

**A mensagem apareceu inteira. Por quê?**

Porque o handler perguntou `error instanceof AppError` e a resposta foi sim. Ao escrever
`new AppError(...)`, quem programou a rota assumiu a responsabilidade pela mensagem: ela foi
escrita em português, pensando em quem vai lê-la, e não contém nada sobre a estrutura interna
do sistema.

Compare com o exercício 1: a mensagem do MySQL foi engolida porque veio marcada como um
`Error` comum, e um `Error` comum pode conter qualquer coisa.

**A régua, em uma frase:** o handler não julga o conteúdo da mensagem — ele julga a
**procedência** dela. Julgar conteúdo exigiria adivinhar; julgar procedência é uma
verificação de uma linha, que nunca erra.

> Um detalhe do `request.params as { numero: string }`: essa conversão manual existe porque a
> rota ainda não tem schema. Quando a rota ganhar um, na próxima aula, o `as` desaparece — o
> Zod passa a inferir o tipo sozinho, e o TypeScript sabe que `numero` é texto sem ninguém
> prometer.

---

## 3. O código que você não escolheu

A rota:

```typescript
// ⚠️ TEMPORÁRIA — apagar depois do exercício.
app.get('/proibido', async () => {
  throw new AppError('Acesso negado', 403)
})
```

**A resposta:**

```json
{
  "statusCode": 403,
  "error": "Forbidden",
  "message": "Acesso negado"
}
```

**Você escreveu a palavra "Forbidden" em algum lugar? Não.** Você escreveu apenas o número
403 e a mensagem em português.

E é isso que o exercício queria mostrar. O `montarResposta` nunca soube o que é 403:

```typescript
error: STATUS_CODES[statusCode] ?? 'Error',
```

O `STATUS_CODES` do `node:http` é a tabela oficial e **completa** do protocolo. Ela já
contém os 60 e poucos códigos que existem, do 100 ao 511. Escrevemos a linha uma vez e ela
funciona para todos, inclusive para códigos que ninguém deste projeto vai usar.

**A lição:** quando você usa a solução que a plataforma já traz, ganha os casos que nem
pensou em cobrir. Se tivéssemos escrito a nossa tabela à mão, ela teria os 4 ou 5 códigos que
lembramos no dia — e o 403 seria um bug esperando alguém precisar dele.

> Experimente um código inventado, como `499`. O campo `error` vira `"Error"`, por causa do
> `?? 'Error'`. É a rede de segurança da linha, e ela existe justamente porque `STATUS_CODES`
> cobre o padrão, e não a criatividade de quem chama.

---

## 4. Investigue o 2º caso

A requisição, no `erros.http`:

```http
### TEMPORÁRIO — exercício 4: JSON quebrado
POST {{host}}/health
Content-Type: application/json

{ "isto": "não fecha"
```

**Resposta obtida:**

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Endereço não encontrado: POST /health"
}
```

**Surpresa?** Era para ser. E a explicação é a parte que vale do exercício:

**O Fastify decide a rota ANTES de interpretar o corpo.** A rota `/health` só existe para
`GET`. Um `POST /health` não é rota nenhuma, então o corpo nem chega a ser olhado — o
`notFoundHandler` responde antes.

Para ver o caso que o exercício pedia, é preciso uma rota que aceite `POST`:

```typescript
// ⚠️ TEMPORÁRIA — apagar depois do exercício.
app.post('/eco', async (request) => request.body)
```

Agora, com `POST /eco` e o mesmo corpo quebrado:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Body is not valid JSON but content-type is set to 'application/json'"
}
```

**A mensagem saiu inteira**, e caiu no **2º caso** —
`error.statusCode !== undefined && error.statusCode < 500`.

**Justificativa:** quem lançou esse erro foi o próprio Fastify, ao tentar interpretar o corpo
da requisição antes de entregá-la à rota. Ele marcou o erro com `statusCode: 400`, que na
especificação do HTTP significa literalmente _"o problema está do seu lado"_.

E leia o que a mensagem diz: ela fala **exclusivamente sobre o que o cliente acabou de
enviar** — o corpo dele e o cabeçalho dele. Não revela nome de tabela, caminho de arquivo nem
versão de biblioteca. Devolvê-la é seguro, e é útil: é ela que permite a quem integrou o
sistema descobrir sozinho que esqueceu uma chave.

**O detalhe que vale guardar:** este caso é a razão de o `errorHandler` ter três ramos, e não
dois. Com apenas "é nosso / não é nosso", toda mensagem do Fastify viraria um 500 genérico, e
quem consome a API perderia a única informação que realmente o ajudaria a se corrigir.

---

## 5. Pergunta para responder por escrito

**Por que o handler global foi registrado antes das rotas?**

Por causa da ordem em que o Fastify monta as coisas. O `setErrorHandler` vale para o escopo
em que foi registrado e para tudo o que é registrado **depois** dele. Colocando-o no topo do
`buildApp()`, ele passa a valer para o `healthRoutes` e para todo módulo que vier no futuro.

Se ele fosse registrado depois das rotas, o comportamento passaria a depender da posição da
linha em um arquivo — e "funciona se você colocar na ordem certa" é o tipo de regra que
ninguém lembra na segunda vez.

**E se alguém criasse um módulo novo e esquecesse de tratar erros dentro dele?**

**Nada de ruim aconteceria.** É exatamente esse o ponto.

O módulo novo é registrado através do `buildApp()`, que é o único caminho por onde uma rota
entra nesta API. E o `buildApp()` já ligou o handler antes. A rota nasce protegida sem que
quem a escreveu precise saber que o handler existe.

Compare com o outro desenho possível — um `try/catch` dentro de cada rota:

|                           | Centralizado (o nosso) | Espalhado (`try/catch` por rota) |
| :------------------------ | :--------------------- | :------------------------------- |
| Rota nova está protegida? | Sempre                 | Só se a pessoa lembrar           |
| Mudar o formato do erro   | Um arquivo             | Todos os arquivos                |
| Uma rota esquecida        | Impossível             | Vaza, e ninguém percebe          |
| Código de tratamento      | 1 lugar                | Repetido em cada rota            |

A diferença não é de estilo, é de **garantia**. Um sistema espalhado depende de disciplina
humana, e disciplina humana falha exatamente no dia de pressa. Um sistema centralizado
depende da estrutura, que não tem dia ruim.

A lição vale muito além do tratamento de erro: **quando a proteção depende de alguém
lembrar, ela já falhou** — só ainda não descobrimos quando.

---

> **Antes de fechar:** apague todas as rotas temporárias que você criou nestes cinco
> exercícios, e os blocos `### TEMPORÁRIO` do `erros.http`. Rode `git diff` e confira que
> sobrou apenas o que a aula mandou escrever. Deixar andaime de pé é o defeito mais fácil de
> cometer e o mais chato de descobrir seis meses depois.
