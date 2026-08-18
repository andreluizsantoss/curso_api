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
- [ ] **[07 — Validação de entrada e contrato de resposta](./07-validacao-e-contrato.md)**
      Nunca confiar no que chega de fora, e devolver exatamente o que foi combinado.

## Fase C — Abrindo a API para o mundo

Deixar de ser um projeto que roda na sua máquina e passar a ser um serviço que outras pessoas
conseguem usar — e que aguenta ser usado.

- [ ] **[08 — Documentação viva com Swagger](./08-documentacao-da-api.md)**
      Gerar, a partir dos schemas que você já escreveu, uma documentação navegável que não
      consegue ficar desatualizada.
- [ ] **[09 — Segurança HTTP](./09-seguranca-http.md)**
      Derrubar a própria API com cinco linhas, e depois impedir isso — com limite de
      requisições, controle de origem e os cabeçalhos que o navegador respeita.
- [ ] **[10 — Docker: empacotando a API](./10-docker.md)**
      Acabar com o "na minha máquina funciona": código, dependências e a versão exata do Node
      dentro de um pacote só, que sobe com um comando em qualquer computador.
- [ ] **[11 — Produção de verdade: desligamento, proxy e logs](./11-producao-de-verdade.md)**
      Fazer a API terminar o que começou antes de morrer, descobrir com quem ela realmente
      está falando atrás de um proxy, e garantir que nenhum segredo vá parar no log.
- [ ] **[12 — Docker Compose: o ambiente completo](./12-docker-compose.md)**
      Descrever o ambiente inteiro — API e banco de dados — em um arquivo versionado, e
      descobrir por que "o container subiu" não significa "o serviço está pronto".
- [ ] **[13 — Banco de dados: Prisma, MySQL e a camada Repository](./13-banco-de-dados.md)**
      Fazer a API guardar dado de verdade, com a estrutura do banco versionada em migrations
      e testes rodando contra um MySQL de verdade, sem destruir o banco de trabalho.

---

## O que vem depois

Esta trilha termina na Aula 13, com uma API que sobe, é padronizada por ferramenta, lê
configuração validada, tem testes automatizados, não vaza informação interna, não confia no
que chega de fora, se documenta sozinha, se defende de quem abusa dela, roda igual em
qualquer máquina, morre sem cortar a requisição de ninguém, registra sem entregar segredo e
traz o ambiente inteiro de pé com um comando só e guarda dado em um banco cuja estrutura é
versionada junto com o código.

Os assuntos abaixo continuam a construção. Não são "opcionais" — são os próximos degraus, e
cada um só faz sentido depois de você ter sentido a dor que ele resolve:

| Assunto                  | A dor que resolve                                                |
| :----------------------- | :--------------------------------------------------------------- |
| Migrations em produção   | Mudar tabela em banco com dado real não perdoa improviso         |
| Autenticação com JWT     | Qualquer um acessa qualquer coisa                                |
| Integração contínua      | Nada garante que alguém rodou os testes antes de enviar o código |
| Branches e Pull Requests | Todo mundo envia direto para a `main`, sem ninguém revisar       |

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
