# 📖 Gabarito — Aula 08: Documentação Viva com Swagger

> Confira só depois de tentar. O valor destes exercícios está no que você observa antes de
> ler a resposta — três dos quatro produzem um resultado que a maioria das pessoas não prevê.

---

## Exercício 1 — Prove que a ordem importa

> No `app.ts`, mova o bloco `if (options.docs ...)` para **depois** do
> `app.register(healthRoutes)`. Rode `npm test` e depois abra a página. Qual dos dois avisou
> primeiro que algo estava errado — o teste ou o navegador?

### O que acontece

A aplicação sobe normalmente. `/health` e `/health/ready` continuam respondendo. A página
`/documentation` abre, com o título certo.

E a especificação sai **vazia**:

```
status: 200
paths: {}
quantidade de rotas documentadas: 0
/health continua respondendo? 200 {"status":"ok"}
```

No navegador, a página mostra a faixa **"No operations defined in spec!"**.

### A resposta

**O teste avisou primeiro, e avisou melhor.**

O navegador também mostrou o problema — mas só para quem foi olhar, e só para quem sabia o
que deveria estar lá. Se você tivesse feito essa alteração no meio de uma refatoração maior,
sem abrir a página, o commit passaria.

Os testes falham na hora — e são **três**, não um:

```
 ❯ src/shared/docs/docs.spec.ts (6 tests | 3 failed)
     × inclui as rotas que existem de verdade
     × descreve os quatro campos de /health/ready
     × separa a rota interna da rota pública por etiqueta

 FAIL  src/shared/docs/docs.spec.ts > Documentação ligada > inclui as rotas que existem de verdade
AssertionError: expected [] to deeply equal [ '/health', '/health/ready' ]
```

Três luzes acendendo pela **mesma** causa. Quando isso acontecer com você, comece pela
primeira falha da lista: normalmente as outras são consequência dela, e consertar a primeira
apaga as três de uma vez.

### Por que isso acontece

O `@fastify/swagger` não lê o seu código-fonte. Ele escuta o aviso que o Fastify dispara a
cada rota registrada e vai anotando.

Com as rotas registradas antes dele, não havia ninguém escutando quando os avisos passaram.
Ele subiu depois, sem nada anotado, e gerou uma especificação honesta: **vazia, porque ele
não viu rota nenhuma.**

### O que levar disso

Repare no tipo de defeito. Não houve erro, não houve mensagem, não houve nada vermelho. Tudo
continuou "funcionando" — inclusive a página, que abriu.

> **Defeito que quebra é fácil. O que continua parecendo certo é o caro.**

E note qual das duas verificações pegou o problema. O navegador depende de alguém lembrar de
olhar; o teste roda em toda execução do `npm run check`, para sempre, sem depender da memória
de ninguém.

---

## Exercício 2 — Uma rota nova sem documentação

> Acrescente uma rota qualquer (`/versao`) com `schema` de resposta, e rode `npm test`. Qual
> teste falha, e por quê? Ele estava certo em falhar?

### O que acontece

Falha o teste **"inclui as rotas que existem de verdade"**:

```
FAIL  src/shared/docs/docs.spec.ts > Documentação ligada > inclui as rotas que existem de verdade
AssertionError: expected [ '/health', '/health/ready', …(1) ] to deeply equal [ '/health', '/health/ready' ]

- Expected
+ Received

  [
    "/health",
    "/health/ready",
+   "/versao",
  ]
```

### A resposta: sim, ele estava certo em falhar

E esta é a parte que costuma incomodar. "Mas eu só adicionei uma rota, não quebrei nada!"

Não quebrou. Mas o teste não pergunta "está quebrado?". Ele pergunta **"a documentação
continua sendo exatamente o que decidimos publicar?"** — e a resposta mudou.

Olhe o que mais o experimento mostrou sobre a rota nova:

```
a rota nova tem tags? undefined
```

Sem `tags`, sem `summary` e sem `description`, a `/versao` entra na documentação num grupo
solto, sem título e sem explicação. Ela **aparece**, mas não informa nada.

### O que fazer quando esse teste falhar de verdade

Não é "conserte o teste". É um checklist de três itens:

1. Esta rota deve mesmo aparecer na documentação pública?
2. Se sim, ela tem `summary`, `description` e `tags`?
3. Só então acrescente o caminho à lista do teste.

O teste falhando é ele fazendo exatamente o trabalho dele: **transformar "esqueci de
documentar" em uma parada obrigatória**, em vez de uma descoberta feita meses depois por
quem tentou integrar.

> **Um teste que só falha quando algo quebra vale metade.** O outro metade do valor está em
> falhar quando uma decisão consciente precisa ser tomada de novo.

---

## Exercício 3 — O `hide: true` na prática

> Coloque `hide: true` no schema do `/health/ready` e recarregue a página. A rota sumiu da
> documentação — agora chame `http://localhost:3333/health/ready` no navegador.

### O que acontece

```
paths na documentacao: []
a rota escondida responde? 200 {"status":"ok"}
```

A rota **sumiu da documentação** e **continua respondendo normalmente**.

### A resposta

`hide: true` mexe em **uma** coisa: se aquela rota entra ou não no documento OpenAPI.

Ele não mexe no roteamento, não exige autenticação, não bloqueia ninguém. Quem souber o
endereço — ou quem testar caminhos comuns, o que é a primeira coisa que qualquer ferramenta
de varredura faz — chega lá igual.

### O que isso mostra sobre "esconder" como estratégia

Esconder a rota trocaria um problema visível por um problema invisível:

| Antes de esconder                               | Depois de esconder                              |
| :---------------------------------------------- | :---------------------------------------------- |
| A rota está aberta, e está escrito que ela está | A rota está aberta, e ninguém mais lembra disso |
| Quem lê a documentação sabe o que vai encontrar | Só quem escreveu o código sabe que ela existe   |
| A dívida tem dono e prazo                       | A dívida virou surpresa para alguém, um dia     |

**O risco não diminuiu em nada. Só a nossa consciência dele diminuiu** — e essa é a pior
troca possível em segurança.

### Então quando `hide: true` é legítimo?

Quando a rota **realmente não faz parte do contrato**, e não quando ela faz parte e é
constrangedora:

- um endpoint que só o próprio servidor chama;
- uma rota antiga em processo de remoção, que você não quer que ninguém novo comece a usar;
- uma rota interna que já exige autenticação — aí a proteção é o login, e o `hide` é só
  arrumação da página.

Repare no terceiro caso: `hide` aparece **junto** de uma proteção de verdade, nunca no lugar
dela.

A pergunta que separa os dois usos: _"se eu escondo isto, alguma porta continua destrancada?"_

---

## Exercício 4 — Documentação sem página

> Comente **apenas** o `app.register(fastifySwaggerUi, ...)` no `src/shared/docs/index.ts` e
> tente acessar os três endereços.

### O que acontece

Os três respondem **404**, no formato de erro da API:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Endereço não encontrado: GET /documentation/json"
}
```

Inclusive o `/documentation/json`, que muita gente aposta que viria do `@fastify/swagger`.

### A resposta

Isso confirma a divisão de trabalho da tabela do Capítulo 3:

| Pacote                | O que faz                                  | Publica endereço? |
| :-------------------- | :----------------------------------------- | :---------------- |
| `@fastify/swagger`    | Gera o documento e o guarda **na memória** | **Não**           |
| `@fastify/swagger-ui` | Serve a página, o `/json` e o `/yaml`      | **Sim**           |

Com só o primeiro registrado, a especificação **existe** — o Fastify passa a ter um método
`app.swagger()` que a devolve como objeto —, mas nenhum endereço HTTP foi criado. A planta
foi desenhada e ficou na gaveta.

### Por que isso importava para a decisão do Capítulo 8

Porque respondeu uma pergunta prática: **o que exatamente precisa ser desligado em produção?**

Se o `/json` viesse do `@fastify/swagger`, desligar só o `swagger-ui` deixaria a especificação
inteira disponível para quem soubesse o caminho — a página some, o mapa fica. Seria a mesma
ilusão do Exercício 3, em outra roupa.

Como os três endereços vêm do mesmo plugin, desligar é uma decisão só. E foi por isso que o
teste "não publica nenhum dos três endereços" confere **os três**, e não apenas a página:
verificar dois e esquecer o terceiro seria pior do que não verificar, porque daria a
sensação de estar coberto.

---

## 🎯 O fio que liga os quatro

Os exercícios 1, 3 e 4 mostram a mesma coisa por ângulos diferentes: **coisas que parecem
resolvidas e não estão.**

| Exercício | A aparência                            | A realidade                      |
| :-------- | :------------------------------------- | :------------------------------- |
| 1         | A página abriu, então funciona         | Está vazia                       |
| 3         | A rota sumiu da documentação           | Continua aberta para qualquer um |
| 4         | O gerador está registrado, então serve | Não publica endereço nenhum      |

E o exercício 2 mostra o remédio: **um teste que falha quando uma decisão precisa ser tomada
de novo.** É a única coisa nos quatro casos que não depende de alguém lembrar de conferir.
