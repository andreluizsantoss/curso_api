# Curso de API — Node.js, TypeScript e Fastify

Um curso de seis aulas que sai de uma pasta vazia e chega a uma API que valida o que recebe,
não vaza informação interna quando algo falha e devolve exatamente o que o contrato declara —
construída do jeito que se constrói software de verdade, e não do jeito que cabe num tutorial
de vinte minutos.

**Comece por aqui:** [`aulas/00-indice.md`](./aulas/00-indice.md)

---

## Para quem é

Para quem está começando e nunca escreveu uma API. Não é preciso saber TypeScript, Node ou
Git: tudo o que for usado é explicado na aula em que aparece pela primeira vez.

O que se espera de você é outra coisa: **digitar o código**, em vez de copiar e colar. A
diferença entre os dois é a única razão pela qual este material existe.

---

## O que você vai construir

Uma API chamada `curso_api`, com:

- uma rota `/health` que informa se o sistema está no ar, separada de uma `/health/ready` que
  informa se ele está pronto para receber tráfego;
- código organizado por funcionalidade, cada camada com uma responsabilidade só;
- padronização automática, para o computador cuidar do estilo em vez de você;
- configuração validada, que faz a API recusar subir se algo estiver errado;
- tratamento de erros que nunca vaza detalhe interno para quem chamou;
- contrato de entrada e de saída declarado, que impede campo não previsto de escapar;
- as requisições que provam cada uma dessas garantias, versionadas junto do código.

---

## As seis aulas

| #   | Aula                                                         | O que resolve                                             |
| :-- | :----------------------------------------------------------- | :-------------------------------------------------------- |
| 01  | [Criando a API do zero](./aulas/01-criando-api-do-zero.md)   | Não existe nada. Ao fim, existe uma API respondendo       |
| 02  | [Subindo para o GitHub](./aulas/02-subindo-para-o-github.md) | O código só existe na sua máquina                         |
| 03  | [Padronizando o código](./aulas/03-padronizando-o-codigo.md) | Cada um escreve de um jeito, e ninguém revisa estilo      |
| 04  | [Variáveis de ambiente](./aulas/04-variaveis-de-ambiente.md) | Configuração escrita dentro do código, que falha calada   |
| 05  | [Tratamento de erros](./aulas/05-tratamento-de-erros.md)     | Quando algo falha, a API conta o que não deveria          |
| 06  | [Validação e contrato](./aulas/06-validacao-e-contrato.md)   | Qualquer um manda qualquer coisa, e a resposta vaza campo |

Cada aula termina com **"Como saber que deu certo"**, **"Erros Comuns"** e exercícios com
[gabarito comentado](./aulas/exercicios/).

---

## E depois da sexta?

A trilha continua — testes automatizados, documentação gerada a partir do código, segurança
HTTP, container, banco de dados, rota de negócio, autenticação. Ela não vem junto de
propósito.

Cada um desses assuntos resolve uma dor específica, e aula que resolve dor que ninguém sentiu
não ensina: faz decorar. Estas seis existem para você **sentir** essas dores primeiro. O
índice lista os próximos degraus, na ordem em que virão.

---

## Antes de começar

| Ferramenta | Para quê           | Onde                                                     |
| :--------- | :----------------- | :------------------------------------------------------- |
| Node.js    | Executar o código  | [nodejs.org](https://nodejs.org) — versão 24             |
| VS Code    | Escrever o código  | [code.visualstudio.com](https://code.visualstudio.com)   |
| Git        | Versionar o código | [git-scm.com](https://git-scm.com) — a partir da Aula 02 |

Cada aula confere as instalações que ela vai usar, antes de usar. Se algo faltar, você
descobre ali, com o comando que mostra a versão instalada — e não no meio de um erro
confuso.

---

## Como usar este material

**Siga na ordem.** Cada aula depende do que a anterior construiu, e nenhuma instala uma
ferramenta antes de existir um problema que a justifique.

**Não siga em frente com o projeto quebrado.** Se a seção "Como saber que deu certo" não der
o resultado esperado, pare ali. Aula seguinte sobre projeto quebrado só multiplica o
problema.

**Leia a mensagem de erro inteira, até o fim.** Na maioria das vezes ela diz exatamente o que
fazer. Esse hábito sozinho já separa quem evolui rápido de quem trava.

---

## Glossário

Todo termo técnico que aparece nas aulas está explicado em [`GLOSSARIO.md`](./GLOSSARIO.md),
em linguagem simples. Consulte sem cerimônia: ninguém nasce sabendo o que é "runtime".
