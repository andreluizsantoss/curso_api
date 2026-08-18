# 🛑 Gabarito — Aula 11: Produção de verdade

> Confira só depois de tentar. Os quatro exercícios foram executados antes de respondidos, e
> o quarto tem um resultado que quase todo mundo prevê ao contrário.

---

## Exercício 1 — Descubra o prazo padrão da sua máquina

> Meça, na sua instalação do Docker, quanto tempo o `docker stop` sem `-t` espera antes do
> `SIGKILL`. Use um container que ignore sinais e cronometre.

### O comando

```bash
docker run -d --name teste-pid1 node:24-slim node -e "setInterval(()=>{},1000)"
docker stop teste-pid1
docker inspect teste-pid1 --format "{{.State.ExitCode}}"
docker rm teste-pid1
```

O `setInterval` serve só para o processo não terminar sozinho. Ele não trata sinal nenhum,
que é justamente o que precisamos: sendo PID 1, ele **ignora** o `SIGTERM`, e o único jeito
de derrubá-lo é o `SIGKILL` do fim do prazo. O que estamos cronometrando, portanto, é o prazo
inteiro.

### O resultado medido (Docker 29.7.2)

```
-t 1   ->  1397ms   exit=137
-t 10  -> 10376ms   exit=137
padrão ->  3391ms   exit=137
```

Três medições do padrão, em execuções diferentes, deram 3391 ms, 3363 ms e 3418 ms. O padrão
desta instalação é, portanto, de **cerca de 3,4 segundos**.

### O ponto do exercício

A documentação do Docker diz que o padrão é **10 segundos**. A medição diz outra coisa. Nenhum
dos dois está mentindo — a documentação descreve o comportamento histórico, e a versão
instalada aqui se comporta de outro jeito.

A lição não é o número. É que **o número não pode ser suposto**: ele é metade de um acordo do
qual a outra metade está no seu `server.ts`. Se o seu deu 10 segundos, ótimo; a conclusão
continua sendo a mesma, que é passar o `-t` explicitamente.

O `exit=137` em todas as linhas é o detalhe que confirma a leitura: 128 + 9, e 9 é o
`SIGKILL`. Nenhuma daquelas paradas foi um encerramento — todas foram execuções.

---

## Exercício 2 — Faça a API estourar o próprio prazo

> Sem alterar o `server.ts`, faça o log registrar a linha `O encerramento passou do prazo`.
> Explique, em uma frase, o que foi preciso para isso.

### O que é preciso

Duas coisas ao mesmo tempo:

1. uma requisição que demore **mais** que o prazo interno de 10 segundos;
2. um prazo externo (`docker stop -t`) **maior** que os 10 segundos, para que o `SIGKILL` não
   chegue antes e roube a cena.

O jeito mais direto é aumentar o `DEMORA_MS` do andaime para `20_000`, reconstruir a imagem e
derrubar com `docker stop -t 60`.

### O resultado medido

```
CLIENTE: |HTTP=000|CURL=52 | duracao=11239ms

{"sinal":"SIGTERM","msg":"Sinal de desligamento recebido. Encerrando com calma."}
{"level":50,"prazoMs":10000,"msg":"O encerramento passou do prazo. Saindo à força, com requisições em andamento."}
```

Os carimbos de tempo das duas linhas ficaram a **10.005 ms** de distância. O relógio disparou
quando devia.

### O que isso ensina

Compare com a medição do Capítulo 1, em que a requisição também foi cortada. O cliente viu
exatamente a mesma coisa nos dois casos: `HTTP=000`, `CURL=52`.

A diferença está toda **de dentro**. Aqui existe uma linha de log em nível de erro (`50`)
dizendo o que aconteceu, com quanto tempo se esperou. No caso do `SIGKILL`, não existe linha
nenhuma — o processo simplesmente para de existir no meio de uma frase.

Um sistema que falha e **conta** que falhou é investigável. Um que falha em silêncio, não.

---

## Exercício 3 — Prove que o CPF não vai para o log

> A lista `PARAMETROS_SENSIVEIS` inclui `cpf`. Escreva o teste que prova isso para uma URL com
> dois parâmetros, sendo um deles o CPF, e rode.

### O teste

```ts
it('oculta o CPF sem estragar o resto da URL', () => {
  expect(mascararUrl('/protocolos?cpf=12345678900&pagina=3')).toBe(
    `/protocolos?cpf=${VALOR_OCULTADO}&pagina=3`,
  )
})
```

### O resultado real

```
/protocolos?cpf=[Redacted]&pagina=3
```

### Por que o CPF está nessa lista

Token e senha são segredos óbvios. O CPF não é segredo — é dado **pessoal**, e essa é uma
categoria diferente, com uma regra própria: ele identifica uma pessoa específica.

Um log cheio de CPF é um cadastro de cidadãos guardado em um lugar que não foi projetado para
guardar cadastro de cidadãos: sem controle de acesso fino, copiado para toda ferramenta de
monitoramento, replicado em todo backup.

O `pagina=3` continuar visível é parte da resposta, e não um detalhe. Mascarar a URL inteira
seria mais fácil e tornaria o log inútil justamente para o que ele serve: entender o que foi
pedido.

---

## Exercício 4 — Descubra o que acontece com dois proxies

> Com `TRUST_PROXY=1` e um cabeçalho `X-Forwarded-For: 1.1.1.1, 2.2.2.2, 3.3.3.3`, qual IP a
> API vai registrar? Preveja antes de testar, depois teste e compare.

### A previsão que a maioria faz

`1.1.1.1` — porque é o primeiro da lista, e porque o cabeçalho "guarda o IP original do
cliente", que é como ele costuma ser explicado.

A tabela do Capítulo 6 já tinha entregado que não é assim, mas com dois endereços só: "o
último" e "o segundo" são a mesma posição, e dá para acertar pelo motivo errado. Com três, as
duas leituras se separam.

### O resultado medido

```
TRUST_PROXY=1    -> 3.3.3.3
TRUST_PROXY=2    -> 2.2.2.2
TRUST_PROXY=true -> 1.1.1.1
```

### Por que é ao contrário

O cabeçalho é escrito da **esquerda para a direita**, na ordem em que a requisição atravessou
a rede: o cliente à esquerda, e cada proxy acrescentando quem falou com ele. O último a
escrever é o proxy mais próximo da API.

Contar "um salto" é olhar **de trás para a frente**, a partir da API. O primeiro salto é o
vizinho imediato — `3.3.3.3`. Dois saltos, `2.2.2.2`.

E é isso que torna o número uma escolha de segurança, e não um detalhe: as entradas da
esquerda são as que **qualquer cliente pode ter inventado**, porque foram escritas antes de a
requisição encostar em qualquer proxy seu. `TRUST_PROXY=true` acredita justamente nelas.

### A regra que sai daqui

O número que você configura é **quantos proxies seus** existem na frente da API — nunca
quantos endereços aparecem no cabeçalho. Se você tem um proxy, é `1`, mesmo que o cabeçalho
chegue com dez endereços. Os outros nove são exatamente aquilo em que você **não** pode
confiar — porque qualquer um pode tê-los escrito.
