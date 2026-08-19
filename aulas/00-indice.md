# Índice do Curso — API com Node.js, TypeScript e Fastify

Bem-vindos. 👋

Este não é um curso de exercício jogado fora no fim. Vocês vão construir uma API de verdade,
tratada com o **rigor de um sistema em produção**: segurança, qualidade e cada decisão
explicada, nunca apenas aplicada. É esse rigor que separa quem sabe fazer de quem sabe
copiar.

**Como usar este índice:** siga na ordem. Cada aula depende do que veio antes.

As caixas são **suas**: marque o `[ ]` conforme for concluindo cada aula. Elas começam todas
vazias de propósito — o progresso registrado aqui é o seu, não o do projeto.

---

## Fase A — Fundação

Montar o projeto e as ferramentas que sustentam todo o resto.

- [ ] **[01 — Criando a API do zero](./01-criando-api-do-zero.md)**
      O que é uma API, o que é Node, o que é TypeScript. Montar o projeto e fazer a primeira
      rota responder.
- [ ] **[02 — Subindo para o GitHub](./02-subindo-para-o-github.md)**
      Versionar o código, entender commits e mandar o projeto para a nuvem.
- [ ] **[03 — Padronizando o código](./03-padronizando-o-codigo.md)**
      ESLint e Prettier: fazer o computador cuidar da qualidade e da aparência do código.

## Fase B — O que a API promete, e o que ela esconde

Sair de "funciona quando tudo dá certo" e passar a cuidar das bordas: a configuração, a falha
e as duas portas por onde os dados entram e saem.

- [ ] **[04 — Variáveis de ambiente com Zod](./04-variaveis-de-ambiente.md)**
      Tirar configuração de dentro do código e falhar rápido quando ela estiver errada.
- [ ] **[05 — Tratamento centralizado de erros](./05-tratamento-de-erros.md)**
      Responder erro de forma consistente, sem nunca vazar detalhe interno para quem chamou.
- [ ] **[06 — Validação de entrada e contrato de resposta](./06-validacao-e-contrato.md)**
      Nunca confiar no que chega de fora, e devolver exatamente o que foi combinado.

---

## Onde esta trilha termina, e por quê

Seis aulas. Ao final delas você tem uma API que sobe, é padronizada por ferramenta, lê
configuração validada, não vaza informação interna quando algo falha, não confia no que chega
de fora e devolve exatamente o que o contrato declara.

**Isso é um assunto inteiro fechado**, e é onde a trilha para por enquanto.

Não para porque acabou o material. Para porque o que vem depois — fazer o computador testar a
API sozinho, gerar documentação a partir do código, defender a API de quem abusa dela,
empacotá-la em container, guardar dado em banco — só faz sentido depois que estas seis
estiverem firmes. Cada um desses assuntos resolve uma dor que você ainda não sentiu, e aula
que resolve dor que ninguém sentiu vira decoreba.

Quando estas seis estiverem sólidas — e "sólidas" quer dizer que você consegue explicar cada
arquivo do seu projeto para outra pessoa —, a trilha continua. A ordem já está definida:

| Próximos degraus              | A dor que resolve                                               |
| :---------------------------- | :-------------------------------------------------------------- |
| Testes automatizados          | Conferir tudo na mão a cada alteração, e um dia esquecer        |
| Documentação viva             | Ninguém de fora sabe quais rotas existem nem o que elas aceitam |
| Segurança HTTP                | Um script de dez linhas derruba a API                           |
| Empacotamento em container    | "Na minha máquina funciona" — rodar em outro computador dói     |
| Desligamento, proxy e logs    | A API morre no meio da requisição, e o log guarda segredo       |
| Ambiente completo por comando | Subir tudo depende de alguém lembrar a ordem certa              |
| Banco de dados                | A API não guarda nada                                           |
| Alterar banco em produção     | Mudar tabela com dado de gente dentro não perdoa improviso      |
| Rota de negócio e versão      | A API só sabe dizer que está viva                               |
| Autenticação                  | Qualquer um acessa qualquer coisa                               |

---

## Materiais de apoio

| Documento                    | Para quê                                          |
| :--------------------------- | :------------------------------------------------ |
| [Glossário](../GLOSSARIO.md) | Todo termo técnico explicado em linguagem simples |
| [Exercícios](./exercicios/)  | Gabaritos comentados dos desafios de cada aula    |

O glossário cobre a trilha inteira, inclusive termos que só aparecem nos degraus seguintes.
Encontrar ali uma palavra que nenhuma aula sua usou ainda é normal — não é sinal de que você
pulou alguma coisa.

---

## Três combinados

**1. Trave sem vergonha.** Ficar 40 minutos preso em algo que um colega resolve em 2 é
desperdício, não esforço. Pergunte.

**2. Leia a mensagem de erro.** Inteira, até o fim. Na maioria das vezes ela diz exatamente
o que fazer. Esse hábito sozinho já separa quem evolui rápido de quem trava.

**3. Não siga em frente com o projeto quebrado.** Toda aula termina com uma seção
**"Como saber que deu certo"**. Se aquele comando não der o resultado esperado, pare ali e
resolva antes de abrir a próxima aula.
