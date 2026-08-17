# Índice do Curso — API do Curso

Bem-vindos. 👋

Este não é um curso de exercício jogado fora no fim. Vocês vão construir uma API completa,
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

## Fase B — Qualidade e robustez

Transformar "funciona na minha máquina" em código que aguenta o mundo real.

- [ ] **[04 — Variáveis de ambiente com Zod](./04-variaveis-de-ambiente.md)**
      Tirar configuração de dentro do código e falhar rápido quando ela estiver errada.
- [ ] **[05 — Testes automatizados com Vitest](./05-testes-automatizados.md)**
      Fazer o computador conferir se a API funciona, em vez de testar na mão toda vez.
- [ ] **[06 — Tratamento centralizado de erros](./06-tratamento-de-erros.md)**
      Responder erro de forma consistente, sem nunca vazar detalhe interno para quem chamou.

---

## O que vem depois

Esta trilha termina na Aula 06, com uma API que sobe, é padronizada por ferramenta, lê
configuração validada, tem testes automatizados e não vaza informação interna.

Os assuntos abaixo continuam a construção e ficam para uma **trilha avançada**. Não são
"opcionais" — são os próximos degraus, e cada um só faz sentido depois de você ter sentido a
dor que ele resolve:

| Assunto                       | A dor que resolve                                                     |
| :---------------------------- | :-------------------------------------------------------------------- |
| Validação de entrada com Zod  | Qualquer pessoa pode mandar qualquer coisa para a API                 |
| Documentação viva com Swagger | Ninguém de fora sabe quais rotas existem nem o que elas aceitam       |
| Segurança HTTP                | API pública sem limite de requisições cai com um script de dez linhas |
| Docker                        | "Na minha máquina funciona"                                           |
| Banco de dados com Prisma     | Hoje nada é guardado: o dado morre quando o servidor reinicia         |
| Integração contínua           | Nada garante que alguém rodou os testes antes de enviar o código      |
| Branches e Pull Requests      | Todo mundo envia direto para a `main`, sem ninguém revisar            |
| Autenticação com JWT          | Qualquer um acessa qualquer coisa                                     |

---

## Materiais de apoio

| Documento                    | Para quê                                          |
| :--------------------------- | :------------------------------------------------ |
| [Glossário](../GLOSSARIO.md) | Todo termo técnico explicado em linguagem simples |
| [Exercícios](./exercicios/)  | Gabaritos comentados dos desafios de cada aula    |

---

## Três combinados

**1. Trave sem vergonha.** Ficar 40 minutos preso em algo que um colega resolve em 2 é
desperdício, não esforço. Pergunte.

**2. Leia a mensagem de erro.** Inteira, até o fim. Na maioria das vezes ela diz exatamente
o que fazer. Esse hábito sozinho já separa quem evolui rápido de quem trava.

**3. Não siga em frente com o projeto quebrado.** Toda aula termina com uma seção
**"Como saber que deu certo"**. Se aquele comando não der o resultado esperado, pare ali e
resolva antes de abrir a próxima aula.
