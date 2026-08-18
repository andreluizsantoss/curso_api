# Curso de API — Node.js, TypeScript e Fastify

Um curso de catorze aulas que sai de uma pasta vazia e chega a uma API que valida o que recebe,
não vaza informação interna, se documenta a partir do próprio código, se defende de quem
abusa dela, roda igual em qualquer computador e guarda dado em um banco cuja estrutura é
versionada junto com o código — construída do jeito que se constrói software de verdade, e
não do jeito que cabe num tutorial de vinte minutos.

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
- **87 testes automatizados**, alguns deles contra um banco de dados de verdade;
- tratamento de erros que nunca vaza detalhe interno para quem chamou;
- contrato de entrada e de saída declarado, que impede campo não previsto de escapar;
- documentação navegável gerada a partir do código, que não consegue ficar desatualizada;
- limite de requisições, controle de origem e os cabeçalhos de segurança do navegador;
- uma imagem Docker que empacota tudo, com a versão exata do Node junto;
- um ambiente completo — API e MySQL — que sobe com um comando em qualquer máquina;
- a estrutura do banco versionada em migrations, ao lado do código que depende dela.

---

## As quinze aulas

| #   | Aula                                                             | O que resolve                                                                 |
| :-- | :--------------------------------------------------------------- | :---------------------------------------------------------------------------- |
| 01  | [Criando a API do zero](./aulas/01-criando-api-do-zero.md)       | Não existe nada. Ao fim, existe uma API respondendo                           |
| 02  | [Subindo para o GitHub](./aulas/02-subindo-para-o-github.md)     | O código só existe na sua máquina                                             |
| 03  | [Padronizando o código](./aulas/03-padronizando-o-codigo.md)     | Cada um escreve de um jeito, e ninguém revisa estilo                          |
| 04  | [Variáveis de ambiente](./aulas/04-variaveis-de-ambiente.md)     | Configuração escrita dentro do código, que falha calada                       |
| 05  | [Testes automatizados](./aulas/05-testes-automatizados.md)       | Testar na mão, no navegador, a cada alteração                                 |
| 06  | [Tratamento de erros](./aulas/06-tratamento-de-erros.md)         | Quando algo falha, a API conta o que não deveria                              |
| 07  | [Validação e contrato](./aulas/07-validacao-e-contrato.md)       | Qualquer um manda qualquer coisa, e a resposta vaza campo                     |
| 08  | [Documentação da API](./aulas/08-documentacao-da-api.md)         | Ninguém de fora sabe quais rotas existem nem o que aceitam                    |
| 09  | [Segurança HTTP](./aulas/09-seguranca-http.md)                   | Um script de dez linhas derruba a API                                         |
| 10  | [Docker: empacotando a API](./aulas/10-docker.md)                | "Na minha máquina funciona" — rodar em outro computador dói                   |
| 11  | [Produção de verdade](./aulas/11-producao-de-verdade.md)         | A API morre no meio da requisição, e o log guarda segredo                     |
| 12  | [Docker Compose](./aulas/12-docker-compose.md)                   | Subir o ambiente inteiro depende de alguém lembrar a ordem                    |
| 13  | [Banco de dados](./aulas/13-banco-de-dados.md)                   | A API não guarda nada, e a estrutura do banco vive na memória de alguém       |
| 14  | [Migrations em produção](./aulas/14-migrations-em-producao.md)   | Alterar tabela em banco com dado de gente não perdoa improviso                |
| 15  | [Negócio e versionamento](./aulas/15-negocio-e-versionamento.md) | A API só sabe dizer que está viva, e não tem endereço que sobreviva a mudança |

Cada aula termina com **"Como saber que deu certo"**, **"Erros Comuns"** e exercícios com
[gabarito comentado](./aulas/exercicios/).

---

## Antes de começar

| Ferramenta     | Para quê           | Onde                                                                                |
| :------------- | :----------------- | :---------------------------------------------------------------------------------- |
| Node.js        | Executar o código  | [nodejs.org](https://nodejs.org) — versão 24                                        |
| VS Code        | Escrever o código  | [code.visualstudio.com](https://code.visualstudio.com)                              |
| Git            | Versionar o código | [git-scm.com](https://git-scm.com) — a partir da Aula 02                            |
| Docker Desktop | Empacotar a API    | [docker.com](https://www.docker.com/products/docker-desktop/) — a partir da Aula 10 |

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
