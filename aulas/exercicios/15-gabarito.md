# Gabarito — Aula 15: a primeira funcionalidade de negócio (e o endereço dela)

> Cada resposta aqui foi **executada** antes de ser escrita. Onde a execução contrariou a
> previsão, o gabarito diz isso em vez de esconder — foi assim que a Aula 14 ganhou o alerta
> mais importante dela (item M-83 do checklist).

---

## Exercício 1 — o campo que não devia sair

**O que era pedido:** acrescentar `criadoPor: 'teste'` ao objeto que `montarResumo` devolve, no
controller, e observar a listagem.

### O que aconteceu, de verdade

```
HTTP 200 | registros: 3
campos na resposta : id, nomeCompleto, nomeSocial, dataNascimento, municipio, uf, criadoEm
criadoPor apareceu?: false

{"id":"96bc62b3-...","nomeCompleto":"Ana Clara Dias","nomeSocial":null,
 "dataNascimento":"2001-07-04","municipio":"Itapetininga","uf":"SP",
 "criadoEm":"2026-08-18T22:57:47.249Z"}
```

**O campo não apareceu.** O controller devolveu, e a resposta saiu sem ele.

### Por quê

Quem monta a resposta não é o controller: é o **serializador** do Fastify, usando o
`cidadaoResumoSchema`. Ele constrói o JSON campo a campo, a partir do que o schema declara.
Campo que não está lá não é "removido" — ele simplesmente nunca é escrito.

### A pergunta que importa: onde mora a garantia

Esta é a resposta que vale a aula inteira.

Se a garantia dependesse de o controller lembrar de não devolver o campo, ela seria uma
**recomendação** — e recomendação falha no dia em que alguém com pressa faz um `...cidadao`
para "aproveitar tudo". A garantia mora no schema, que é o único caminho por onde a resposta
pode sair.

> **É o mesmo raciocínio da regra de CPF único da Aula 13:** a garantia não está no `if` do
> service, está na coluna `@unique`. O `if` existe para produzir mensagem em português; a
> coluna é o que continua valendo quando alguém escrever outro caminho e esquecer o `if`.

E é por isso que o teste correspondente confere uma **ausência**:

```ts
expect(lista[0]).not.toHaveProperty('cpf')
expect(lista[0]).not.toHaveProperty('criadoPor')
```

Teste de ausência é o único que percebe um vazamento acrescentado por engano. Nenhum teste de
"o campo X está correto" jamais reprovaria por causa de um campo a mais.

**Não esqueça de desfazer a alteração** antes de seguir. Se você commitou, o `npm run check`
avisa — o `check:aulas` compara o arquivo do disco com o que a aula mostra.

---

## Exercício 2 — reativar em vez de recadastrar

**O que era pedido:** desenhar (sem implementar) a rota de reativação.

Não existe uma resposta única aqui. Existe uma resposta **defensável**, e é a defesa que conta.

### O desenho proposto

| Item              | Escolha                                        |
| :---------------- | :--------------------------------------------- |
| Método            | `POST`                                         |
| Endereço          | `/api/v1/cidadaos/:id/reativacao`              |
| Resposta de êxito | **200**, com o cadastro completo               |
| Erros             | **404** (não existe) · **409** (já está ativo) |

### A pergunta difícil: reativar é criar ou alterar?

A pergunta está mal-posta de propósito, e perceber isso é a resposta.

**Não é nenhum dos dois — é uma ação.** E é por isso que o desenho acima não usa `PATCH` com
`{ "excluidoEm": null }`, que seria a tradução literal de "alterar um campo".

Três motivos:

1. **`PATCH` expõe a implementação.** Mandar `excluidoEm: null` obriga quem consome a saber que
   existe uma coluna com esse nome e que `null` significa ativo. No dia em que a exclusão lógica
   mudar de forma, todo mundo que integra quebra.
2. **Reativar não é editar um campo, é um evento.** Ele pode, amanhã, precisar registrar quem
   reativou, gravar um motivo, disparar um aviso. Nada disso cabe em "altere esta coluna".
3. **`PATCH` já é usado para os dados do cadastro**, e o cadastro excluído responde 404 nele —
   corretamente. Abrir uma exceção só para um campo criaria uma regra difícil de explicar.

O `POST` em um sub-recurso (`/reativacao`) é a forma usual de expressar "execute esta ação sobre
este recurso" quando a ação não é um simples CRUD.

### O que acontece com os campos do cadastro antigo

**Eles permanecem.** É todo o motivo de a exclusão ser lógica: o cadastro não foi perdido, foi
arquivado. Reativar traz a pasta de volta para a mesa, com o que estava dentro.

O que a implementação precisaria decidir, e vale escrever no plano:

- `excluidoEm` e `excluidoPor` voltam a `null`;
- `atualizadoEm` muda sozinho (é `@updatedAt`);
- os dados pessoais **não** são zerados — quem quiser corrigi-los usa o `PATCH` depois;
- **409 se já estiver ativo**, e não 200 calado: responder sucesso a uma ação sem efeito é o
  mesmo defeito do `PATCH` com corpo vazio.

---

## Exercício 3 — a mudança que obriga a `v2`

| #   | Mudança                                         | Sobe versão? | Motivo em uma frase                                                                            |
| --- | :---------------------------------------------- | :----------: | :--------------------------------------------------------------------------------------------- |
| 1   | Acrescentar `profissao` (opcional) na consulta  |    ❌ Não    | Quem já consome ignora o campo que não conhece; ninguém precisa mexer em nada                  |
| 2   | Passar a exigir `email` no cadastro             |    ✅ Sim    | Toda requisição que hoje passa sem e-mail passa a falhar — quebra quem já integra              |
| 3   | Renomear `nomeCompleto` para `nome` na resposta |    ✅ Sim    | Todo cliente que lê `nomeCompleto` recebe `undefined`, e a falha é silenciosa na outra ponta   |
| 4   | Passar a recusar telefone com 9 dígitos         |    ❌ Não    | **Cuidado com esta.** Ver a discussão abaixo                                                   |
| 5   | `POST` responder 202 em vez de 201              |    ✅ Sim    | 201 é "está criado"; 202 é "vou criar depois" — muda o que a outra ponta deve fazer em seguida |

### A número 4 é a interessante

O enunciado avisa: _"hoje já recusa, mas por outro caminho"_. O telefone é validado por
comprimento (10 ou 11 dígitos), então 9 dígitos **já é recusado hoje**.

Se a API já recusa, passar a recusar por outra regra **não muda o que o cliente observa**: as
mesmas requisições que falhavam continuam falhando, com o mesmo código 400. Nada quebra.

**A regra que decide não é "apertei a validação?", é "alguma requisição que passava parou de
passar?".** São perguntas diferentes, e só a segunda importa para quem consome.

Se o exercício dissesse "passar a recusar telefone **fixo** de 10 dígitos", aí sim: requisições
que passavam passariam a falhar, e a resposta seria ✅.

### O erro comum na número 1

É tentador responder "sim, mudou a resposta". Mas acrescentar campo é a mudança mais comum de
todas, e se ela exigisse versão nova, uma API viveria em `v40`. O contrato de uma resposta é
sobre **o que ela garante entregar**, não sobre ela ser idêntica para sempre.

---

## Exercício 4 — o índice que não existe

**O que era pedido:** o que precisa acontecer se surgir uma rota que busca cidadão por município.

### O estado de hoje, medido

```
SHOW INDEX FROM cidadaos:
  PRIMARY                  -> id
  cidadaos_cpf_key         -> cpf
  cidadaos_excluidoEm_idx  -> excluidoEm

EXPLAIN SELECT * FROM cidadaos WHERE municipio = 'Itapetininga' AND excluidoEm IS NULL:
  type: ref
  key:  cidadaos_excluidoEm_idx
  rows: 3
```

São **três** índices, e nenhum deles é sobre `municipio`.

Repare no que o `EXPLAIN` mostra: o banco usou o índice de `excluidoEm` — o único que serve para
alguma parte da consulta — e depois conferiu o município **linha por linha** entre as que
sobraram. Com três registros isso é instantâneo e não prova nada. Com trezentos mil, significa
percorrer todos os ativos a cada busca.

### A linha do schema

```prisma
  @@index([municipio])
```

Ou, melhor para esta consulta específica:

```prisma
  @@index([municipio, excluidoEm])
```

O índice composto cobre as duas condições da mesma consulta, na ordem em que ela filtra. A ordem
dentro do colchete importa: um índice em `[municipio, excluidoEm]` serve também para uma busca
só por município, mas um em `[excluidoEm, municipio]` não serve para uma busca só por município.

### O custo de não fazer nada

Duas coisas, e a segunda é a que surpreende:

1. **A consulta fica lenta** — proporcional ao número de cadastros ativos, e piorando sozinha à
   medida que o cadastro cresce.
2. **A lentidão aparece meses depois, sem ninguém ter mexido em nada.** É o pior tipo de defeito
   para diagnosticar: o código não mudou, a rota é a mesma, e "de repente" a tela demora. Em
   máquina de desenvolvimento, com trinta registros de seed, ela nunca vai reproduzir.

### E o custo de fazer

Índice não é grátis: ele ocupa espaço e torna cada gravação um pouco mais cara, porque o banco
precisa mantê-lo atualizado. Por isso não se cria índice "por precaução" em toda coluna.

A regra prática: **índice para o que a aplicação consulta de verdade**. É exatamente o critério
que criou o `@@index([excluidoEm])` desta aula — não porque `excluidoEm` parecesse importante,
mas porque **toda** listagem filtra por ele.
