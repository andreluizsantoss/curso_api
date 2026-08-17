# Gabarito — Aula 05

---

## 1. Veja falhar antes de confiar

Alterando, por exemplo, `expect(service.getStatus().status).toBe('ok')` para `'okay'`:

```
 FAIL  src/modules/health/health.spec.ts > HealthService > responde com status "ok"

AssertionError: expected 'ok' to be 'okay' // Object.is equality

Expected: "okay"
Received: "ok"

 ❯ src/modules/health/health.spec.ts:16:42
     14|   it('responde com status "ok"', () => {
     15|     const service = new HealthService()
     16|     expect(service.getStatus().status).toBe('ok')
       |                                        ^
```

### Quatro informações que a mensagem entrega

| O que aparece                                    | Para que serve                                                             |
| :----------------------------------------------- | :------------------------------------------------------------------------- |
| **O nome do teste**                              | Diz qual comportamento quebrou, em português, antes de você olhar código   |
| **`Expected` e `Received`**                      | Mostra os dois valores lado a lado. Metade dos bugs se resolve só com isso |
| **O arquivo e a linha** (`health.spec.ts:16:42`) | Leva direto ao ponto, sem procurar                                         |
| **O trecho de código com o `^`**                 | Aponta a coluna exata da asserção que falhou                               |

### Por que este exercício existe

Um teste que você **nunca viu falhar** não é confiável.

Imagine um teste escrito errado, que passa sempre — mesmo com o código quebrado. Ele te dá
uma sensação de segurança falsa, que é pior do que não ter teste nenhum: você deixa de
verificar na mão porque "tem teste cobrindo".

Por isso o hábito: ao escrever um teste novo, **quebre-o de propósito uma vez** e confirme
que ele acusa. Depois conserte. Leva cinco segundos e vale por muito.

---

## 2. Um teste novo, do zero

```typescript
it('informa o ambiente em que está rodando', () => {
  const service = new HealthService()

  expect(service.getStatus().environment).toBe('test')
})
```

O valor é **`'test'`**.

### Como descobrir sem adivinhar

Foi o que a dica do enunciado sugeria: escreva o valor errado de propósito e deixe a mensagem
te contar.

```
Expected: "development"
Received: "test"
```

Pronto — o próprio teste respondeu.

Essa é uma técnica legítima e muito usada, inclusive por gente experiente: quando você não
sabe qual valor esperar, escreva qualquer um, rode, e leia o `Received`. É mais rápido e mais
confiável do que procurar na documentação.

### Por que `'test'` e não `'development'`

O Vitest define `NODE_ENV=test` sozinho, antes de carregar os arquivos. Faz sentido: é assim
que uma aplicação sabe que está rodando dentro de uma bateria de testes e pode se comportar de
acordo — usar um banco de testes em vez do real, por exemplo.

Repare que isso só funciona porque o nosso `envSchema`, da Aula 04, **aceita** `'test'` na
lista de ambientes válidos. Se aceitasse apenas `development` e `production`, a validação
derrubaria a aplicação no primeiro teste.

Aquela decisão foi tomada quatro capítulos antes de existir um teste sequer. Não foi sorte —
foi pensar em qual seria o próximo passo.

---

## 3. Teste o método HTTP errado

```typescript
it('devolve 404 quando o método HTTP não é o esperado', async () => {
  const app = buildApp({ logger: false })

  const resposta = await app.inject({ method: 'POST', url: '/health' })

  expect(resposta.statusCode).toBe(404)

  await app.close()
})
```

**O status é 404**, e o corpo é:

```json
{ "message": "Route POST:/health not found", "error": "Not Found", "statusCode": 404 }
```

### Se você respondeu 405, seu raciocínio estava certo

Existe um código HTTP feito exatamente para esta situação: o **405 Method Not Allowed**, que
significa "esse endereço existe, mas não aceita esse método".

Muita gente experiente responderia 405. **O Fastify devolve 404** — e o motivo é interessante.

Para o Fastify, uma rota é a combinação **método + caminho**. `GET /health` e `POST /health`
são duas rotas completamente diferentes. Como só registramos a primeira, a segunda
simplesmente **não existe** — e o que não existe é 404.

Repare na mensagem: `Route POST:/health not found`. O método faz parte da identidade da rota.

### A lição que fica

O importante deste exercício não é decorar 404 ou 405. É perceber que **o comportamento real
nem sempre é o que a intuição diz**, e que o jeito de descobrir é **rodar e observar** — não
supor.

Essa é a mesma habilidade da questão 4 do gabarito da Aula 03. Ela vai se repetir a carreira
inteira.

---

## 4. Quebre o projeto e veja o portão funcionar

Trocando o `200` por `201` no `health.controller.ts` e rodando `npm run check`:

```
> npm run lint && npm run format:check && npm run test && npm run build

> eslint src                    ✅ passou

> prettier --check .            ✅ passou

> vitest run
 ⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯
 AssertionError: expected 201 to be 200 // Object.is equality

 Test Files  1 failed | 1 passed (2)
      Tests  1 failed | 18 passed (19)
```

**Parou na terceira etapa. O build não chegou a rodar.**

### O papel do `&&`

O `&&` significa **"só execute o próximo se o anterior deu certo"**.

```
lint  &&  format:check  &&  test  &&  build
 ✅         ✅            ❌       (nem roda)
```

Como o `test` falhou, a corrente parou ali. O `npm run check` inteiro terminou com código de
saída diferente de zero — o combinado universal para "deu errado".

### Por que parar em vez de continuar

Poderia parecer melhor rodar tudo e mostrar todos os problemas de uma vez. Mas parar na
primeira falha é melhor por dois motivos:

1. **Economia de tempo.** Se o teste falhou, o código está errado. Compilar código errado é
   trabalho jogado fora.
2. **Foco.** Uma lista com quinze problemas paralisa. Um problema por vez, na ordem, é o que
   se consegue realmente resolver.

### E por que isso importa de verdade

Repare que apenas **um** teste falhou, entre 19. Uma alteração de um único caractere.

Sem os testes, essa alteração passaria despercebida: o código compila, o lint aprova, a
formatação está certa. A API subiria devolvendo `201 Created` para uma simples consulta de
saúde — e o sistema de monitoramento, que espera `200`, começaria a reportar a API como fora
do ar.

**O portão pegou.** É para isso que ele existe.

Na Aula 06, esse mesmo `npm run check` vai rodar no GitHub a cada envio de código. Aí nem
"esquecer de rodar" será possível.

---

## 5. `app.inject()` × subir o servidor de verdade

Resposta esperada, com suas palavras. Qualquer dois destes motivos valem:

> **1. Velocidade.** Abrir e fechar uma porta de rede leva tempo. Com dezenas de testes, isso
> vira minutos de espera a cada execução — e teste lento é teste que ninguém roda.
>
> **2. Sem conflito de porta.** Se dois testes tentarem usar a porta 3333 ao mesmo tempo, um
> deles falha com `EADDRINUSE`, sem que exista nenhum defeito no código. O `inject()` não
> ocupa porta nenhuma, então esse problema simplesmente não existe.
>
> **3. Funciona em qualquer ambiente.** A máquina que roda a verificação automática do GitHub
> pode ter restrições de rede. Sem abrir porta, não há nada a restringir.
>
> **4. Isolamento.** Sem porta aberta, nenhum programa de fora consegue interferir no teste. O
> resultado depende só do nosso código.

### O que NÃO muda

Um ponto importante: o `inject()` **não é um atalho que testa menos**.

A requisição percorre exatamente o mesmo caminho de uma requisição real: passa pelo
roteamento do Fastify, pelos plugins, pelo controller, pelo service, e a resposta é montada e
serializada do mesmo jeito. A única coisa que não acontece é o tráfego sair pela placa de rede.

É a diferença entre testar um motor num dinamômetro e testar o carro na rua: o motor trabalha
de verdade nos dois casos.

### E o motivo mais bonito

Tudo isso só é possível porque, lá na Aula 01, `buildApp()` foi separado de `app.listen()`.

Se a montagem da aplicação e a abertura da porta estivessem no mesmo lugar, **não haveria como
testar a rota sem subir um servidor**. A aula de hoje seria muito mais difícil, ou
simplesmente não existiria.

Guarde essa relação de causa e efeito: uma decisão de arquitetura, cujo benefício não era
óbvio quando foi tomada, abriu uma porta quatro aulas depois. É assim que arquitetura boa se
paga.
