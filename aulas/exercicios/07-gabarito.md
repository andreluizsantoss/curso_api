# Gabarito — Aula 07

---

## 1. Descubra o que o contrato não protege

Trocando `uptime: z.number()` por `uptime: z.string()` no `readinessResponseSchema`, e
chamando `/health/ready`:

```
500 {"statusCode":500,"error":"Internal Server Error","message":"Erro interno do servidor. A equipe já foi avisada."}
```

E no log do servidor, o motivo real:

```
FST_ERR_RESPONSE_SERIALIZATION | Response doesn't match the schema
```

**A API devolveu erro.** Não converteu o número em texto, não ignorou a diferença, não
devolveu o valor "do jeito que veio". Reprovou.

### Por que isso é a resposta certa

O contrato vale nos **dois** sentidos, e é fácil enxergar só um deles:

| Situação                                | O que o schema faz    |
| :-------------------------------------- | :-------------------- |
| O código devolve um campo **a mais**    | Remove, em silêncio   |
| O código devolve um campo **diferente** | **Recusa a resposta** |

Remover o campo extra é seguro: ninguém prometeu aquilo, e tirá-lo não quebra quem consome a
API. Já devolver `uptime` como texto quando o contrato promete número **quebraria** quem
consome — o aplicativo do outro lado tentaria fazer conta com um texto.

### E por que o código é 500, e não 400

Repare em quem errou. O `400` significa "você, cliente, mandou algo errado". Aqui o cliente
não mandou nada: **nós** é que devolvemos fora do combinado. Erro nosso é `500`.

E o cliente recebe a mensagem genérica, não o detalhe técnico — exatamente a regra que a Aula
06 estabeleceu. O `Response doesn't match the schema` fica no log, onde a equipe procura.

**Não esqueça de desfazer** a alteração no schema antes de seguir.

---

## 2. Escreva uma mensagem melhor

Comparando os dois schemas com as mesmas três entradas inválidas:

```typescript
// Automático
const automatico = z.object({ numero: z.coerce.number().int().positive() })

// Com mensagens próprias
const proprio = z.object({
  numero: z.coerce
    .number({ error: 'precisa ser um número' })
    .int({ error: 'precisa ser um número inteiro, sem casas decimais' })
    .positive({ error: 'precisa ser maior que zero' }),
})
```

| Entrada | Mensagem automática                          | Mensagem própria                                  |
| :------ | :------------------------------------------- | :------------------------------------------------ |
| `"abc"` | Tipo inválido: esperado número, recebido NaN | precisa ser um número                             |
| `"-5"`  | Muito pequeno: esperado que number fosse >0  | precisa ser maior que zero                        |
| `"3.7"` | Tipo inválido: esperado int, recebido número | precisa ser um número inteiro, sem casas decimais |

### Quando vale o trabalho

**A mensagem automática é boa o suficiente quando quem lê é programador.** Ela é precisa e
diz exatamente o que a ferramenta esperava. Num endpoint interno, consumido por outro sistema,
não há razão para escrever nada.

**A mensagem própria vale quando um ser humano vai ler.** Compare as duas da linha do meio:

- _"Muito pequeno: esperado que number fosse >0"_ — mistura português com inglês, usa `>0` e
  fala de "number", que é o nome do tipo na linguagem.
- _"precisa ser maior que zero"_ — qualquer pessoa entende.

Se essa mensagem for aparecer na tela de um cidadão preenchendo um formulário, a segunda é a
única aceitável.

**A regra prática:** escreva a sua mensagem nos campos que vêm de formulário, e deixe a
automática nos campos que só outro sistema envia.

---

## 3. Duas violações de uma vez

Com uma rota que valida `params` **e** `querystring`, chamando
`/exemplo/protocolo/abc?formato=medio` — onde os dois estão errados:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Dados inválidos no endereço: numero — Tipo inválido: esperado número, recebido NaN"
}
```

**Só o primeiro problema aparece.** A `querystring` inválida não é mencionada.

### Por que o Fastify para no primeiro

Ele valida as partes da requisição **em sequência**: `params`, depois `querystring`, depois
`body`, depois `headers`. Na primeira que falha, ele interrompe e devolve o erro — as
seguintes nem chegam a ser verificadas.

Repare no nosso handler: a função `montarMensagemDeValidacao` usa `error.validationContext`,
no singular. Não existe plural ali, porque o Fastify só informa **um** contexto por resposta.

### Isso é um problema?

Depende de quem está do outro lado.

- **Para outro sistema:** não. Ele corrige, tenta de novo, e descobre o segundo erro.
- **Para uma pessoa preenchendo um formulário:** sim, e bastante. Corrigir um campo, enviar,
  descobrir outro erro, corrigir, enviar de novo é uma experiência ruim.

Dentro de um mesmo contexto, porém, **todos** os campos errados aparecem juntos — é por isso
que a função monta a lista com `campos.join('; ')`. Se dois campos do corpo estiverem errados,
os dois são citados.

Na prática, formulários mandam tudo no `body`, que é um contexto só. O caso do exercício —
errar o endereço e a query ao mesmo tempo — é mais raro do que parece.

---

## 4. O contrato como documentação

Lendo apenas o `health.schema.ts`:

- A rota de prontidão devolve **quatro** campos: `status`, `uptime`, `timestamp`,
  `environment`.
- O `environment` aceita exatamente três valores: `development`, `test` e `production`.

Você respondeu isso em alguns segundos, sem abrir o service, sem rodar a API, sem ler nenhum
`if`.

### O ganho que não é óbvio

Para descobrir o mesmo lendo o service, seria preciso: encontrar o método certo, ler o objeto
devolvido, seguir de onde vem cada valor, e ainda abrir `env.schema.ts` para descobrir quais
valores `NODE_ENV` aceita. Quatro arquivos para uma pergunta simples.

O schema responde porque ele **é** a resposta — não uma descrição da resposta, escrita à parte
e sujeita a envelhecer.

> **A diferença que importa:** documentação escrita à mão descreve o que alguém acreditava que
> o código fazia, no dia em que escreveu. O schema **é** o que o código faz — se ele mentir, a
> requisição falha na hora.

É por isso que a próxima aula consegue gerar uma documentação navegável a partir dele, sem
ninguém escrever nada.
