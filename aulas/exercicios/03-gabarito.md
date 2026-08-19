# Gabarito — Aula 03

---

## 1. Erro ou aviso?

Trocando `app.log.info(...)` por `console.log('teste')` em `src/server.ts`:

```
src/server.ts
  29:5  error  Unexpected console statement  no-console

✖ 1 problem (1 error, 0 warnings)
```

| Comando         | Código de saída | Resultado |
| :-------------- | :-------------: | :-------- |
| `npm run lint`  |      **1**      | Falhou    |
| `npm run check` |      **1**      | Falhou    |

**Foi um erro (`error`), e ele derrubou o `npm run check` inteiro.**

**Por quê:** no `eslint.config.js`, a regra `no-console` está configurada como `'error'`. O
ESLint termina com código de saída 1, e o `&&` do script `check` interrompe a sequência ali
mesmo — a verificação de formatação e o build **nem chegaram a rodar**.

**A distinção que importa:** `error` é "isto não pode entrar no projeto". `warning` é "olhe
para isto, provavelmente está errado". Usar tudo como erro faz o time desligar as
verificações quando tiver pressa; usar tudo como aviso faz ninguém olhar. O equilíbrio é uma
decisão consciente do time.

---

## 2. Afrouxando a regra

Com o `console.log` ainda no lugar, trocando para `'no-console': 'warn'`:

```
src/server.ts
  29:5  warning  Unexpected console statement  no-console

✖ 1 problem (0 errors, 1 warning)
```

| Comando         | Código de saída | Resultado |
| :-------------- | :-------------: | :-------- |
| `npm run lint`  |      **0**      | Passou    |
| `npm run check` |      **0**      | Passou    |

**O que mudou:** o mesmo `console.log`, o mesmo arquivo, a mesma linha — e agora o projeto
inteiro é aprovado. Nada no código mudou. Mudou o que o time decidiu tolerar.

**Por que este projeto escolheu `'error'`:** com `'warn'`, o `console.log` esquecido entra
no projeto e vira parte da paisagem. Aviso amarelo que não reprova nada é aviso que, em
poucas semanas, todo mundo aprende a rolar a tela para não ver.

Não existe resposta universalmente certa — existe a decisão registrada, com o motivo. Aqui o
motivo é concreto: `console.log` em código de servidor sempre tem uma alternativa melhor
(`app.log`), então não há caso legítimo a tolerar. Já o `any`, que continua como aviso, às
vezes é inevitável de verdade.

> [!NOTE]
> **Guarde este exercício.** Ele é a menor demonstração possível de uma ideia grande:
> ferramenta não decide nada sozinha. Alguém configurou aquele `'error'`, e essa escolha vale
> para o time inteiro, todo dia. Ler configuração de ferramenta é ler decisão de gente.

**Antes de seguir:** volte a regra para `'error'` e desfaça o `console.log` no `server.ts`.
Rode `npm run check` e confirme que ele passa de novo.

---

## 3. Investigação de versões

```bash
npm view eslint version
#   10.8.1 (ou mais recente, dependendo de quando você rodar)

npm view typescript-eslint peerDependencies
#   {
#     eslint: '^8.57.0 || ^9.0.0 || ^10.0.0',
#     typescript: '>=4.8.4 <6.1.0'
#   }
```

**"Se saísse hoje um ESLint 11, nosso projeto poderia atualizar na hora?"**

**Não.** E dá para saber isso sem tentar, apenas lendo a linha:

```
eslint: '^8.57.0 || ^9.0.0 || ^10.0.0'
```

O `||` significa "ou". O `typescript-eslint` declara que funciona com a versão 8.57 **ou** a
9 **ou** a 10. A 11 não está na lista. Teríamos que esperar o `typescript-eslint` publicar
uma versão nova declarando esse suporte.

**Como você descobriu:** lendo as `peerDependencies`. É exatamente o mesmo mecanismo que nos
obriga a usar TypeScript 6 em vez do 7.

**A habilidade que fica:** quando um `npm install` falhar com uma parede de texto vermelho
falando em `ERESOLVE` e `peer dependency`, você vai saber que a resposta está nessa
declaração — e não vai perder uma tarde tentando reinstalar tudo do zero.

---

## 4. Desconfie da regra decorada

**O que aconteceu:** nada. O `npm run lint` continua passando, sem imprimir absolutamente
nada, exatamente como antes.

Se você esperava ver erros aparecerem, ótimo — era isso mesmo que a leitura do Capítulo 4
sugeria. E é por isso que este exercício existe.

**Por que o resultado foi esse:**

O trabalho do `eslint-config-prettier` é **desligar** regras de formatação que outros
conjuntos tenham ligado. Só que as versões atuais do ESLint **removeram** as regras de
formatação de `js.configs.recommended`, e o `typescript-eslint` fez o mesmo. A comunidade
concluiu que formatação é trabalho de formatador, não de linter.

Ou seja: **hoje não sobrou quase nada para o `eslint-config-prettier` desligar.** Ele está
desligando um conjunto vazio, e por isso a ordem não faz diferença observável.

**Então por que continuamos deixando ele por último?**

Porque é seguro e custa zero. No dia em que alguém adicionar ao `extends` um conjunto de
regras que mexa com formatação — e isso acontece —, ele já vai estar na posição certa para
evitar o conflito. É uma proteção que só cobra o preço quando é necessária.

> **A lição maior desta questão** não é sobre o Prettier. É sobre o hábito de **verificar**.
> Muita prática em programação continua sendo repetida anos depois de o motivo original ter
> desaparecido. Quem confere descobre; quem só decora, repete.
>
> E repare no que você acabou de fazer: formulou uma hipótese, rodou um experimento e
> comparou o resultado com o esperado. Isso é depuração. É a habilidade central da profissão,
> e você acabou de praticá-la.

---

## 5. Por que não rodar o Prettier por dentro do ESLint?

Resposta esperada, com suas palavras:

> Porque misturar as duas coisas faz um espaço sobrando aparecer com a mesma cor e a mesma
> gravidade de um bug de verdade. Quando tudo é vermelho, a gente aprende a ignorar o
> vermelho — e um dia ignora justamente o erro que importava.
>
> Separados, a divisão fica clara: se o ESLint reclamou, é algo que merece atenção. Se era só
> formatação, o Prettier já resolveu sozinho e nem precisou me avisar. Além disso, rodar o
> formatador dentro do linter deixa o lint mais lento, porque cada arquivo passa duas vezes
> pelo processo.
