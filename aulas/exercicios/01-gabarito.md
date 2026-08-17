# Gabarito — Aula 01

> Tente resolver sozinho antes de ler. Errar e descobrir o porquê ensina muito mais do que
> acertar copiando.

---

## 1. Leia o log com atenção

Uma linha de `incoming request` tem estes campos:

| Campo               | O que significa                                         |
| :------------------ | :------------------------------------------------------ |
| `level`             | Gravidade. `30` é informação, `40` é aviso, `50` é erro |
| `time`              | Momento exato, em milissegundos desde 1970              |
| `pid`               | Número do processo no sistema operacional               |
| `hostname`          | Nome da máquina que atendeu                             |
| `reqId`             | Identificador da requisição (`req-1`, `req-2`...)       |
| `req.method`        | O método HTTP (`GET`)                                   |
| `req.url`           | O caminho pedido (`/health`)                            |
| `req.host`          | O endereço que o cliente chamou (`localhost:3333`)      |
| `req.remoteAddress` | O IP de quem chamou                                     |
| `req.remotePort`    | A porta de saída de quem chamou, sorteada pelo sistema  |
| `msg`               | A mensagem para humanos                                 |

São **11 campos** no total: seis no primeiro nível e cinco dentro de `req`.

**O `responseTime`** aparece na linha `request completed` e diz **quantos milissegundos a API
levou** para responder. No `/health` fica abaixo de 5 ms, e normalmente **abaixo de 1 ms** a
partir da segunda requisição. A primeira costuma ser a mais lenta, porque o processo ainda
está aquecendo; não se assuste se você medir 3 ms na primeira e 0,3 ms nas seguintes.

Por que isso importa: é esse número que denuncia uma rota lenta. Quando uma consulta ao banco
está mal feita, o `responseTime` sobe de 3 ms para 800 ms, e o log mostra isso sem que
ninguém precise reclamar.

**O `reqId` é o campo mais subestimado.** Ele permite juntar todas as linhas de log de uma
mesma requisição, mesmo com centenas de pessoas usando o sistema ao mesmo tempo.

---

## 2. Uma informação nova na resposta

Você precisou mexer em **dois lugares dentro do mesmo arquivo**
(`src/modules/health/health.service.ts`):

```typescript
export interface HealthStatus {
  status: string
  uptime: number
  timestamp: string
  environment: string
  versao: string //  ← 1. o contrato
}

export class HealthService {
  getStatus(): HealthStatus {
    return {
      status: 'ok',
      uptime: process.uptime(),
      timestamp: new Date().toISOString(),
      environment: process.env.NODE_ENV ?? 'development',
      versao: '1.0.0', //  ← 2. o valor de verdade
    }
  }
}
```

**Por que dois lugares?** Porque eles respondem a perguntas diferentes:

- A **interface** é a _promessa_: "toda resposta de saúde terá um campo `versao` que é texto".
- O **objeto retornado** é a _entrega_: o valor concreto.

E é exatamente aí que o TypeScript te protege. Se você adicionar só na interface, ele acusa:
_"está faltando a propriedade `versao`"_. Se adicionar só no objeto, ele acusa que você está
devolvendo algo que não estava combinado.

Em JavaScript puro nada disso aconteceria: o campo simplesmente sumiria da resposta, e você
descobriria quando o aplicativo do cidadão quebrasse.

---

## 3. Descubra o erro de propósito

Ao trocar `status: 'ok'` por `status: 123`, o TypeScript acusa:

```
Type 'number' is not assignable to type 'string'.
```

**Onde aparece:**

- Um sublinhado ondulado vermelho embaixo do `123`.
- Com o **Error Lens** instalado, a mensagem aparece escrita **na própria linha**, em vermelho.
- Sem o Error Lens, você só vê ao passar o mouse por cima ou no painel "Problems".
- Também aparece se você rodar `npm run build`, que **falha**.

**Atenção a uma coisa que surpreende quase todo mundo:** se você estiver com o `npm run dev`
ligado, o servidor **continua funcionando**. Acesse `/health` depois de salvar e você verá:

```json
{ "status": 123, "uptime": 10.2, "timestamp": "...", "environment": "development" }
```

Resposta HTTP 200, valor errado, nenhum aviso. Isso não é um defeito do projeto — é a divisão
de trabalho entre duas ferramentas:

- O **`tsx`**, que o `npm run dev` usa, só **traduz** TypeScript para JavaScript. Ele é rápido
  justamente porque **não confere tipo nenhum**. Para ele, `status: 123` é um valor como
  qualquer outro.
- O **`tsc`**, que o `npm run build` usa, é quem **confere**. E ele falha.

**A lição:** a tipagem te avisa **antes de o código chegar ao cidadão**, mas quem dá esse
aviso é o **editor** (o sublinhado vermelho, na hora em que você digita) e o **`npm run
build`** — nunca o servidor de desenvolvimento. Por isso o `build` não é opcional: é ele que
decide se o código pode ou não ir para produção. Na Aula 06 nós vamos colocar esse mesmo
`build` para rodar automaticamente a cada envio de código, para que ninguém consiga publicar
uma versão que não compila.

---

## 4. Mude a porta sem alterar código

A linha `const PORT = Number(process.env.PORT) || 3333` lê uma **variável de ambiente**. Basta
defini-la ao rodar o comando:

**Windows (PowerShell):**

```powershell
$env:PORT=4000; npm run dev
```

**Linux / Mac / Git Bash:**

```bash
PORT=4000 npm run dev
```

O log vai mostrar `"port":4000`, e a API responde em `http://localhost:4000/health`.

**Por que isso é importante:** em produção, ninguém edita código para mudar configuração. O
servidor define as variáveis de ambiente e a mesma aplicação, sem uma linha alterada, sobe
com a configuração certa em cada lugar.

> **Um problema que ainda não resolvemos:** tente rodar com `PORT=abc`. A API sobe na porta
> **3333**, calada, como se nada tivesse acontecido. O `Number('abc')` vira `NaN`, o operador
> `||` engole o problema e usa o valor reserva.
>
> Em produção isso é grave: o balanceador de carga aponta para 4000, a API está em 3333, e o
> sistema fica fora do ar sem nenhuma mensagem de erro. **É exatamente esse buraco que a Aula
> 04 vai tapar.**

---

## 5. Por que o controller recebe o service pelo construtor?

Resposta esperada, com suas palavras:

> Porque assim o controller não fica preso a uma implementação específica. Ele declara o que
> precisa e alguém de fora decide o que entregar. Nos testes, podemos entregar um service
> falso que devolve dados controlados, e verificar o controller sozinho, sem depender do
> comportamento real. Se o controller criasse o service com `new` lá dentro, não haveria como
> trocar, e testar as duas coisas separadamente seria impossível.

**Um detalhe a mais:** repare que o `HealthController` importa o `HealthService` com
`import type`. Isso é uma pista forte da arquitetura — ele conhece apenas o **formato** do
service, não o código dele. Essa linha desaparece por completo quando o projeto é compilado.
