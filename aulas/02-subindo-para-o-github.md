# Aula 02: Subindo para o GitHub (O Guia Definitivo para Iniciantes) 🚀

Bem-vindos à segunda aula!

Hoje, vocês vão aprender uma das ferramentas mais importantes na vida de qualquer pessoa que trabalha com tecnologia: o **Versionamento de Código**.

Se vocês nunca usaram Git ou GitHub, não se preocupem.

Este tutorial foi escrito exatamente para vocês, que são estagiários e estão dando os primeiros passos no desenvolvimento.

Vamos entender cada detalhe, cada comando e o porquê de cada coisa, passo a passo, como se estivéssemos juntos em uma sala de aula presencial.

> [!TIP]
> Dica de ouro para o seu estágio: Leia cada parágrafo com atenção, não pule comandos e sempre tente entender **o que** o comando faz antes de apertar `Enter`.

---

## Índice da Aula

1. Capítulo 1: O que é Versionamento e por que precisamos dele?
2. Capítulo 2: Git vs GitHub — Qual a diferença?
3. Capítulo 3: Pré-requisitos e Configuração Inicial
4. Capítulo 4: Os dois arquivos que orientam o Git — `.gitignore` e `.gitattributes`
5. Capítulo 5: Comandos Git Locais — Passo a passo
6. Capítulo 6: Criando o repositório no GitHub
7. Capítulo 7: Conectando e Enviando
8. Capítulo 8: Fazendo tudo pelo VS Code
9. Capítulo 9: Resumo e Próximos Passos
10. Glossário e FAQ do Estagiário

---

## Capítulo 1: O que é Versionamento e por que precisamos dele? ⏪

Imagine que você está escrevendo um TCC ou um relatório muito importante no Word. Você começa a salvar os arquivos assim:

- `relatorio.docx`
- `relatorio_final.docx`
- `relatorio_final_agoravai.docx`
- `relatorio_final_revisado_2.docx`
- `relatorio_final_oficial.docx`

Isso é uma forma (muito ruim) de versionamento. Você está tentando guardar o histórico das alterações caso precise voltar atrás.

Na programação, nós lidamos com centenas de arquivos de texto (o nosso código).

Se fizermos isso copiando pastas inteiras (`projeto_v1`, `projeto_v2`), o caos se instaura rapidamente.

O **Versionamento de Código** é um sistema profissional e inteligente que gerencia todas as alterações feitas nos arquivos de um projeto ao longo do tempo.

### Cenários onde o versionamento salva sua vida (e seu emprego):

**1. "Quebrei tudo e não sei como arrumar" 💥**

_Sem versionamento:_
Você tentou melhorar uma função no código, mas algo deu errado.
O projeto inteiro parou de funcionar e começou a dar erro na tela.
Você tenta desfazer usando `Ctrl+Z`, mas já fechou o VS Code e perdeu o histórico de desfazer. Você está completamente perdido e bate o desespero.

_Com versionamento:_
Você simplesmente diz ao sistema: "Volte o código exatamente para como estava hoje de manhã, antes de eu começar a mexer".
E _puf!_ Em um segundo, seu projeto volta a funcionar.

**2. "Duas pessoas mexendo no mesmo arquivo" 👯‍♂️**

_Sem versionamento:_
Você e seu colega abrem o mesmo arquivo pelo pendrive ou Google Drive.
Você salva, ele salva logo depois. O trabalho dele sobrescreve o seu, e você perdeu seu dia todo de trabalho.

_Com versionamento:_
O sistema entende que vocês dois mexeram no mesmo arquivo e consegue unir o trabalho dos dois automaticamente, ou avisar onde há conflitos para vocês decidirem.

> [!NOTE]
> Pensem no versionamento como o sistema de **"Save State"** dos videogames. Antes de enfrentar o chefão (fazer uma alteração perigosa no código), você salva o jogo. Se morrer (quebrar o código), você carrega o jogo salvo e tenta de novo sem perder o progresso.

---

## Capítulo 2: Git vs GitHub — Qual a diferença? 🤔

Muitas pessoas confundem os dois, achando que são a mesma coisa, mas eles têm papéis completamente diferentes.

### Git (A Máquina do Tempo LOCAL) 🕰️

O Git é um programa (um software) que roda **no seu computador**. Ele foi criado por Linus Torvalds.

- Ele é o motor, a inteligência por trás do versionamento.
- Ele tira "fotos" (chamamos isso de _snapshots_ ou _commits_) do estado atual de todos os seus arquivos.
- Se você quebrar algo, o Git permite que você "volte no tempo" para qualquer foto anterior.
- **Importante:** O Git funciona offline. Você não precisa de internet para usar o Git, pois tudo acontece localmente no seu HD/SSD.

### GitHub (A Nuvem / O Álbum de Fotos Público) ☁️

O GitHub é um **site na internet** (hoje pertencente à Microsoft) que hospeda projetos que usam Git.

- É a plataforma onde você envia as suas "fotos" (commits) que o Git tirou.
- Ele serve para:

  1. **Trabalho em equipe:** Seus colegas podem baixar o seu código da nuvem e você pode baixar o deles.
  2. **Backup:** Se o seu computador pegar fogo hoje, seu código estará seguro no GitHub.
  3. **Portfólio:** Quando você procura emprego na área de TI, o seu perfil no GitHub funciona como um currículo prático. O avaliador pode entrar lá e ver os códigos que você escreveu.

> [!TIP]
> **A Grande Analogia:**
> Imagine que o **Git** é o seu _diário pessoal_, que fica na gaveta do seu quarto. Você anota tudo o que faz nele.
> O **GitHub** é a _biblioteca pública_ da cidade. Você faz uma cópia do seu diário e coloca lá. Assim, se você perder o original, tem uma cópia de segurança na biblioteca.

---

## Capítulo 3: Pré-requisitos e Configuração Inicial ⚙️

Antes de iniciarmos o versionamento, precisamos garantir que o motor (Git) está instalado e configurado corretamente no nosso sistema.

### Passo 1: Verificar se o Git está instalado

No VS Code, abra o terminal (`Ctrl + '` ou vá em `Terminal > New Terminal`).
Digite o seguinte comando e aperte Enter:

```bash
git --version
```

- **Se retornar `git version X.X.X`:** Ótimo, o Git está instalado!
- **Se retornar `comando não encontrado`:** Você precisa baixar e instalar. Vá até [https://git-scm.com](https://git-scm.com), baixe a versão para o seu sistema operacional e instale. Feche e abra o terminal novamente após a instalação.

### Passo 2: Configurando sua Identidade (OBRIGATÓRIO) 🆔

> [!IMPORTANT]
> Você DEVE configurar a sua identidade ANTES do primeiro commit. Se você tentar tirar uma foto do código sem configurar o seu nome e e-mail, o `git commit` **VAI DAR ERRO** e você não conseguirá avançar. O Git precisa saber a autoria de cada modificação.

Rode os comandos abaixo no terminal, substituindo os dados pelos seus dados reais:

```bash
git config --global user.name "Seu Nome Completo"
git config --global user.email "seu.email@exemplo.com"
```

**Por que usamos `--global`?**
A _flag_ `--global` significa que essa configuração de identidade será aplicada a **todos os projetos** que você criar no seu computador daqui para frente. Assim, você só precisa fazer essa configuração uma única vez na vida no seu notebook.

Se não apareceu nenhuma mensagem de erro após rodar os comandos, sucesso! Sua identidade foi gravada.

---

## Capítulo 4: Os dois arquivos que orientam o Git 🛡️

O Git obedece a dois arquivos de configuração que ficam na raiz do projeto. Um diz **o que
não entra** no repositório (`.gitignore`); o outro diz **como o conteúdo é gravado e
entregue** (`.gitattributes`). Vamos criar os dois agora, antes do primeiro commit.

### O Guarda-costas: o arquivo `.gitignore`

Todo projeto de software tem arquivos locais que não devem ir para o GitHub. Seja porque são pesados demais, inúteis para a equipe, ou porque contém segredos da empresa.

O **Guarda-costas** que barra esses arquivos de entrarem nas nossas fotos do Git é o arquivo `.gitignore`.

#### Como criar o arquivo corretamente

1. No VS Code, abra a barra lateral de arquivos (Explorer).
2. Clique com o botão direito na **raiz** do projeto (num espaço vazio abaixo dos arquivos).
3. Clique em **New File** (Novo Arquivo).
4. Digite **`.gitignore`** (com o ponto final no começo da palavra!).
5. Aperte Enter.

Agora, precisamos escrever as regras. Vamos entender seção por seção:

- **Dependências (`node_modules/`):** Essa pasta contém bibliotecas baixadas. Ela é enorme (centenas de megabytes) e pode ser recriada instantaneamente rodando `npm install`. NUNCA versionamos essa pasta.

- **Build (`dist/`):** A pasta onde ficam os arquivos compilados finais gerados pelo TypeScript. Ignoramos pois versionamos apenas o código-fonte original.

- **Variáveis de Ambiente (`.env`):** ATENÇÃO MÁXIMA AQUI.

> [!CAUTION]
> Os arquivos `.env` guardam senhas de banco de dados, chaves secretas de APIs e tokens. Se você enviar isso para um GitHub público, **qualquer hacker poderá acessar o banco de dados da empresa e vazar todos os dados**. Nunca remova o `.env` do `.gitignore`.

- **Logs:** Arquivos gerados para registrar erros do npm.
- **Sistema Operacional (`.DS_Store`, `Thumbs.db`):** Lixos visuais gerados por Macs e Windows.
- **IDE (`.idea/`, `*.swp`, `*.swo`):** Configurações pessoais do seu editor de texto, como temas escuros/claros, que não devem ser enfiadas goela abaixo na máquina do seu colega.

#### O Arquivo Completo para Copiar e Colar

Para não ter erro, copie o bloco exato abaixo e cole dentro do seu `.gitignore` e depois salve (`Ctrl + S`):

```text
# =============================================
# Dependências
# =============================================
node_modules/

# =============================================
# Build (arquivos compilados pelo TypeScript)
# =============================================
dist/

# =============================================
# Variáveis de ambiente (senhas, tokens, etc.)
# =============================================
.env
.env.local
.env.*.local

# =============================================
# Logs
# =============================================
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# =============================================
# Sistema operacional
# =============================================
.DS_Store
Thumbs.db

# =============================================
# IDE / Editor
# =============================================
.idea/
*.swp
*.swo
```

Nosso projeto agora está blindado contra o que não deve entrar.

---

### O tradutor: o arquivo `.gitattributes`

Existe uma diferença entre Windows e Linux que ninguém vê, e que por isso mesmo dá muito
trabalho: **o caractere que marca o fim de cada linha de um arquivo de texto.**

Quando você aperta Enter em um editor, o arquivo não guarda um "Enter" — ele guarda um
caractere invisível. E cada sistema operacional escolheu um jeito diferente:

| Sistema     | O que grava no fim da linha  | Nome curto |
| :---------- | :--------------------------- | :--------- |
| Windows     | dois caracteres: `CR` + `LF` | **CRLF**   |
| Linux e Mac | um caractere: `LF`           | **LF**     |

Parece detalhe sem importância. Não é. O Git compara arquivos **caractere por caractere**,
e esses caracteres contam. Sem uma regra combinada, três coisas acontecem:

1. **Diferenças falsas.** Você abre um arquivo que um colega escreveu no Linux, salva sem
   mudar nada, e o Git acusa que **todas as linhas mudaram**. Ninguém consegue revisar um
   `pull request` assim.
2. **Ferramentas reprovando projeto correto.** O Prettier, que você vai instalar na próxima
   aula, espera `LF`. Se o Git entregar `CRLF`, ele reprova o projeto inteiro — e a culpa
   não é de nenhuma linha de código.
3. **O clássico "aqui funciona".** O mesmo repositório se comporta de um jeito na sua
   máquina e de outro na do colega, sem que exista uma diferença visível para investigar.

A solução é combinar **uma** regra e escrevê-la no repositório, para valer igualmente para
todo mundo — hoje e para quem entrar no time daqui a dois anos.

Crie na raiz do projeto, do mesmo jeito que você criou o `.gitignore`, o arquivo
**`.gitattributes`** (com ponto no começo, e sem extensão):

```text
# =============================================
# Quebra de linha padronizada
# =============================================
#
# Windows termina cada linha com dois caracteres invisíveis (CR + LF); Linux e
# Mac terminam com um só (LF). Sem uma regra aqui, o Git entrega o formato de
# cada sistema operacional, e o mesmo arquivo passa a ter conteúdo diferente
# dependendo de quem clonou o projeto.
#
# `text=auto` deixa o Git decidir o que é texto e o que é binário (imagem, PDF).
# `eol=lf` manda entregar todo arquivo de texto com LF, em qualquer sistema.
#
# Sem isto, o `npm run format:check` reprova o repositório inteiro em um clone
# feito no Windows — em um projeto que está correto.
* text=auto eol=lf
```

Lendo a última linha da direita para a esquerda:

- `*` — vale para **todos** os arquivos do projeto.
- `text=auto` — o Git identifica sozinho o que é arquivo de texto e o que é binário. Uma
  imagem `.png` nunca deve ser "corrigida": trocar caracteres dentro dela a corromperia.
- `eol=lf` — todo arquivo de texto é entregue no disco com `LF`, em qualquer sistema
  operacional. `eol` é de _end of line_, fim de linha.

> [!NOTE]
> **Por que não basta configurar isso no seu Git?**
>
> Existe um ajuste de Git (`core.autocrlf`) que faz algo parecido, mas ele mora **na sua
> máquina**. Ele não viaja junto com o projeto: o colega que clonar o repositório não recebe
> a sua configuração e volta ao problema.
>
> O `.gitattributes` é versionado, ou seja, vai junto com o código. Regra que precisa valer
> para o time inteiro mora no repositório, nunca na máquina de uma pessoa. Essa ideia se
> repete o curso inteiro — é a mesma razão de existirem o `.editorconfig` e a pasta
> `.vscode` que você vai criar na próxima aula.

> [!TIP]
> Se o seu editor mostra "CRLF" ou "LF" no canto inferior direito, é exatamente disto que
> ele está falando. Depois desta aula, o esperado neste projeto é **LF**.

Com os dois arquivos no lugar, o Git sabe o que ignorar e em que formato gravar. Agora sim:
podemos tirar a primeira foto.

---

## Capítulo 5: Comandos Git Locais — O Passo a Passo Detalhado 📸

Agora sim, vamos botar a mão na massa no terminal. Certifique-se de que o terminal aponta para a pasta do projeto (ex: `C:\projeto\curso_api>`).

### 1. Ligando o Motor: `git init`

```bash
git init
```

**O que faz:** Inicializa um novo repositório Git local.
Ele cria uma pasta escondida chamada `.git` que vai gerenciar todo o histórico daquele momento em diante. Só precisa rodar isso uma vez na vida por projeto.

### 2. O Radar: `git status`

```bash
git status
```

**O que faz:** Mostra o estado atual dos seus arquivos.

- Arquivos vermelhos: Foram alterados, mas não estão preparados para a foto.
- Arquivos verdes: Estão preparados no palco (Staging) e prontos para a foto.

> [!TIP]
> Use o `git status` como uma mania. É o melhor comando para você nunca se perder e sempre saber o que está acontecendo antes de enviar código.

### 3. Arrumando o Palco: `git add .`

```bash
git add .
```

**O que faz:** O ponto `.` adiciona TUDO que está vermelho no palco, deixando verde. Significa "estou preparando todos estes arquivos para a foto".

> [!NOTE]
> **No Windows vão aparecer várias linhas dizendo `warning: ... LF will be replaced by CRLF`. Isso é normal e não é erro.** Windows e Linux marcam o fim de linha de formas diferentes, e o Git está apenas avisando que vai fazer a conversão sozinho. Pode seguir em frente: o commit do próximo passo vai funcionar normalmente.

### 4. Tirando a Foto: `git commit`

```bash
git commit -m "feat: cria a estrutura inicial e configuracoes base"
```

**O que faz:** Salva definitivamente a versão localmente na sua máquina. A tag `-m` é para adicionar uma mensagem (uma legenda para a foto).

**As Convenções de Commit (Fundamental):**
Nunca escreva mensagens como "mudança" ou "arrumei as coisas". O mercado usa o padrão _Conventional Commits_, colocando um prefixo:

- `feat:` -> Quando você cria uma feature/funcionalidade nova.
- `fix:` -> Quando você resolve um bug/erro do sistema.
- `docs:` -> Quando altera só a documentação (como este tutorial).
- `chore:` -> Quando atualiza pacotes ou faz manutenções simples.

> [!WARNING]
> Entenda: o commit salva APENAS no seu HD local! O código ainda NÃO está no GitHub. Se o seu notebook queimar agora, você perde tudo.

---

## Capítulo 6: Criando o repositório no GitHub (Pelo site) ☁️

Vamos alugar nosso espaço na biblioteca pública.

### Passo 1: Conta no GitHub

1. Acesse [https://github.com](https://github.com).
2. Se não tem conta, clique em "Sign up".
3. Use um e-mail profissional, crie uma senha forte e escolha um Username simples e adequado para o mercado de trabalho (ex: `andresilva`, evite nomes infantis).

### Passo 2: Criando o Repositório VAZIO

1. Logado no GitHub, clique no botão verde **New** (Novo repositório).
2. Em **Repository name**, digite: `curso_api`.
3. Escolha **Public** (Público). Duas razões: este projeto vira portfólio — código que você
   pode mostrar a um recrutador —, e quem estiver acompanhando o seu aprendizado consegue
   abrir o repositório sem depender de convite. A regra do passo seguinte é justamente o que
   torna isso seguro.
4. **REGRA VITAL:** NÃO marque a opção "Add a README file" e NÃO escolha nenhum ".gitignore" no select. Nós queremos criar um repositório **100% VAZIO** para não conflitar com os arquivos que já criamos localmente.
5. Clique em **Create repository**.
6. Uma tela cheia de comandos verdes aparecerá. Deixe-a aberta, vamos copiar códigos dela!

### E se o nome já estiver em uso? 🔁

O GitHub não deixa duas pastas com o mesmo nome na mesma conta. Se, ao digitar
`curso_api`, aparecer uma mensagem em vermelho dizendo que o nome já existe,
significa que você (ou a conta que está usando) já tem um repositório com esse nome.

**Não apague o repositório antigo e não reaproveite ele.** Escolha um nome livre, por
exemplo:

```text
curso_api_estudo
```

O nome do repositório na nuvem **não precisa ser igual** ao nome da pasta no seu computador.
São duas coisas independentes: a pasta local é sua, o nome do repositório é só o endereço
dele no GitHub.

> [!IMPORTANT]
> Anote o nome que você realmente usou. Ele aparece no endereço que você vai copiar no
> Capítulo 7, e é o erro número um desta aula: o aluno cria o repositório com um nome e
> depois digita outro no `git remote add`.

> [!CAUTION]
> Nunca aponte este projeto para um repositório que **já tem código dentro**. Os dois
> históricos não teriam nenhuma foto em comum, o `git push` seria recusado, e a única forma
> de "resolver" isso seria forçar o envio — o que apaga o conteúdo que estava lá. Repositório
> novo e vazio, sempre.

---

## Capítulo 7: Conectando e Enviando 🚀

Agora vamos criar a ponte mágica entre o nosso notebook e a Microsoft (GitHub). Volte para o terminal do VS Code.

### Passo 1: Nomeando a Branch Principal

```bash
git branch -M main
```

**O que faz:** Renomeia o galho principal (linha do tempo oficial do seu código) de `master` para `main`, seguindo o padrão atual da comunidade tech.

> [!CAUTION]
> **ERRO CLÁSSICO DE ESTAGIÁRIO:** Se você pular este comando, o seu galho continuará se chamando `master`. Quando você tentar rodar o comando de envio mais abaixo (`git push -u origin main`), o Git vai estourar um erro assustador dizendo `error: src refspec main does not match any`. Isso significa: "Você mandou eu enviar o main, mas aqui só tem master!". **Nunca pule este comando.**

### Passo 2: Criando a Ponte (Remote Add)

> [!WARNING]
> **ATENÇÃO:** Não copie o comando abaixo literalmente. Copie o endereço lá daquela tela do
> GitHub que você deixou aberta no navegador — é ele que está certo, sempre.

O endereço tem duas partes que mudam de pessoa para pessoa:

```bash
git remote add origin https://github.com/SEU-USUARIO/NOME-DO-REPOSITORIO.git
```

| Parte                 | Troque por                                                                                                             |
| :-------------------- | :--------------------------------------------------------------------------------------------------------------------- |
| `SEU-USUARIO`         | Seu nome de usuário real do GitHub                                                                                     |
| `NOME-DO-REPOSITORIO` | O nome que você realmente usou no Capítulo 6 — pode não ser `curso_api`, se aquele nome já estava ocupado |

**O que faz:** Diz para o Git local: "A partir de agora, a sua origem (origin) principal na internet para fazer backups é esse link maravilhoso aqui".

**Confira antes de seguir:**

```bash
git remote -v
```

Ele precisa imprimir o mesmo endereço que está no navegador, duas vezes (`fetch` e `push`).
Se você digitou errado, corrija com:

```bash
git remote set-url origin https://github.com/SEU-USUARIO/NOME-DO-REPOSITORIO.git
```

### Passo 3: O Primeiro Push (Enviando pro espaço)

```bash
git push -u origin main
```

**O que faz:**
O famoso `push` (empurrar) envia suas fotos locais para o repositório online. O `-u` cria um elo definitivo; da próxima vez, você precisará apenas digitar `git push`.

**O Popup de Autenticação (pode aparecer, pode não aparecer):**
Quando você apertar Enter para dar esse primeiro `push`, o sistema vai perceber que você quer enviar código para a conta.

- **Se for a primeira vez** que este computador conversa com o GitHub, uma pequena janela
  (popup) ou aba no navegador se abrirá dizendo "Sign in with your browser". Siga os passos,
  clique em **Allow** e depois **Authorize**. O sistema salva sua credencial com segurança e
  não vai pedir a senha todo dia.
- **Se nada aparecer** e o push simplesmente funcionar, está certo também: a credencial já
  estava guardada nesta máquina de alguma configuração anterior. Não é erro, é o passo já
  cumprido.

O que confirma o sucesso é a saída do comando, não o popup:

```text
branch 'main' set up to track 'origin/main'.
 * [new branch]      main -> main
```

Vá até o site do GitHub e aperte F5. **Mágica!** Seu código está nas nuvens.

### Passo 4: O cartão de visitas do projeto — `README.md`

Repare que o repositório mostra só a lista de arquivos. Falta o cartão de visitas.

Lembra que no Capítulo 6 mandamos **não** marcar "Add a README file"? Aquilo era sobre a tela de criação: um arquivo criado direto pelo GitHub nasceria com um histórico separado do seu, e o `push` seria recusado. Agora que o envio já funcionou, criamos o arquivo do lado certo — o seu.

Crie na **raiz do projeto** (na mesma altura do `package.json`) o arquivo `README.md`:

````markdown
# API do Curso

API RESTful do sistema API do Curso, construída com **Fastify + TypeScript**.

## Começando

```bash
npm install     # baixa as dependências
npm run dev     # sobe a API em http://localhost:3333
```

**Pré-requisito:** Node.js na versão registrada no `.nvmrc`.

## Comandos

| Comando         | O que faz                                          |
| :-------------- | :------------------------------------------------- |
| `npm run dev`   | Sobe a API recarregando a cada alteração salva     |
| `npm run build` | Compila o TypeScript para a pasta `dist`           |
| `npm start`     | Executa a versão compilada, como roda em produção  |

## Rotas

| Método | Rota      | O que devolve                                |
| :----- | :-------- | :-------------------------------------------- |
| `GET`  | `/health` | O estado da API: status, uptime e ambiente    |
````

> [!NOTE]
> O bloco acima começa e termina com **quatro** crases porque o conteúdo dele tem blocos de três crases dentro. Ao criar o seu `README.md`, copie apenas o que está **entre** as linhas de quatro crases — o arquivo começa em `# API do Curso`.

Este README descreve o projeto **como ele está hoje**. Ele vai crescer junto com o projeto: cada aula que acrescentar um comando novo ou uma rota nova acrescenta também a linha correspondente aqui. Documentação que descreve um projeto que não existe mais é pior do que documentação nenhuma.

Agora faça o ciclo completo que você acabou de aprender:

```bash
git add .
git commit -m "docs: adiciona README do projeto"
git push
```

Atualize o navegador: o GitHub mostra esse texto formatado logo abaixo da lista de arquivos. É assim que todo projeto sério se apresenta.

---

## Capítulo 8: Fazendo tudo pelo VS Code (O Modo Visual) 🖱️

No dia a dia corrido de um estagiário ou dev pleno, muitos usam a interface do editor ao invés do terminal, por agilidade.

1. **Aba Source Control:** Na barra da esquerda do VS Code, clique no terceiro ícone (Source Control), que lembra uma árvore/galho.

2. **O Add (Palco):** Quando você altera algo, o arquivo aparece em "Changes". Se você passar o mouse por cima dele e clicar no botão de **`+`**, você acabou de fazer um `git add .` visualmente. Ele vai para "Staged Changes".

3. **O Commit:** No topo da aba tem uma caixa de texto. Escreva lá a sua legenda (`feat: adiciona model de usuarios`). Clique no botão azul **Commit** (ou ícone de _Check_). Foto tirada!

4. **O Push:** O botão vai mudar para **Sync Changes**. Ao clicar nele, ele envia para a nuvem.

Simples, rápido e não requer decoreba!

### Uma extensão que deixa o histórico visível: GitLens

Instale a extensão **GitLens** (`eamodio.gitlens`) pela aba de extensões (`Ctrl + Shift + X`).

O que ela faz: ao colocar o cursor em qualquer linha do código, aparece no fim da linha, em
cinza claro, **quem escreveu aquela linha, quando e com qual mensagem de commit**.

Isso transforma tudo o que você aprendeu nesta aula em algo concreto. Aquele histórico que
parecia abstrato vira informação útil no dia a dia: você abre um arquivo confuso, vê que a
linha estranha foi escrita há dois anos com a mensagem `fix: contorna erro do gateway`, e já
sabe que existe um motivo por trás dela — em vez de simplesmente apagar e quebrar algo.

> [!NOTE]
> Parte dos recursos do GitLens passou a ser paga. A versão gratuita já cobre tudo o que
> usamos aqui, incluindo o histórico ao lado da linha.

---

## Capítulo 9: Resumo e Próximos Passos 🎯

Vocês chegaram ao final e agora têm um projeto versionado com sucesso.

### O Ciclo Diário Real de Trabalho (A Rotina)

A parte difícil (criar conta, repositório e linkar) foi feita hoje. Amanhã de manhã, seu dia será apenas isso:

1. Abre o projeto.
2. Escreve código novo e genial. Salva o arquivo.
3. Roda `git add .`
4. Roda `git commit -m "feat: fiz x e y"`
5. Roda `git push`
6. Descansa em paz.

### Próximos Passos

Existem conceitos para equipes maiores que ficam para uma trilha avançada, mas vale saber
que eles existem:

- **Branches:** Criar caminhos paralelos para você e seu colega trabalharem ao mesmo tempo sem quebrar o projeto do outro.
- **Merge Conflicts:** Como arrumar a bagunça quando dois devs tentam alterar exatamente a mesma linha no mesmo arquivo.
- **Pull Requests (PR):** Como pedir pro tech lead revisar e aprovar seu código antes dele virar oficial.

E na **[Aula 03](./03-padronizando-o-codigo.md)**, que vem a seguir, vamos resolver um
problema que só aparece quando mais de uma pessoa mexe no mesmo projeto: cada um escrevendo
com um estilo diferente. Vamos instalar ferramentas que padronizam o código sozinhas.

---

## ✅ Como saber que deu certo

| Verificação                               | O que esperar                                                                              |
| :---------------------------------------- | :----------------------------------------------------------------------------------------- |
| `git status`                              | Diz `nothing to commit, working tree clean`                                                |
| `git log --oneline`                       | Mostra os seus commits, do mais novo para o mais antigo                                    |
| `git remote -v`                           | Mostra o endereço do seu repositório no GitHub                                             |
| Abrir o repositório no navegador          | Os arquivos do projeto estão lá                                                            |
| Procurar a pasta `node_modules` no GitHub | **Ela não deve estar lá.** Se estiver, o `.gitignore` foi criado depois do primeiro commit |
| Procurar `.gitattributes` no GitHub       | Está lá, com uma linha só de regra: `* text=auto eol=lf`                                   |
| Abrir o repositório no navegador          | O texto do `README.md` aparece formatado abaixo da lista de arquivos                       |

> [!CAUTION]
> A última linha é a mais importante. Se o `node_modules` (ou um arquivo `.env`) subiu para o
> GitHub, o `.gitignore` sozinho **não** resolve — ele só impede arquivos novos, e o que já
> está versionado continua lá. Chame alguém do time antes de tentar consertar sozinho.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/02-gabarito.md`](./exercicios/02-gabarito.md).

**1. Leia o seu próprio histórico**
Rode `git log --oneline`. Quantos commits você fez? Olhando só as mensagens, você conseguiria
explicar para outra pessoa o que foi feito em cada um? Se não, o que faltou na mensagem?

**2. O ciclo completo, sozinho**
Crie na raiz do projeto um arquivo chamado `ANOTACOES.md` e escreva dentro dele uma frase
qualquer sobre o que você aprendeu hoje. Depois faça o ciclo inteiro: `git status`, `git add`,
`git commit`, `git push`. Confira no navegador que o arquivo chegou. Use uma mensagem no
padrão `docs:`.

**3. Teste o guarda-costas**
Crie um arquivo chamado `.env` na raiz com o conteúdo `SENHA_SECRETA=123`. Rode `git status`.
O Git mostrou esse arquivo? Explique o motivo. Depois apague o arquivo.

**4. Investigue com o GitLens**
Com o GitLens instalado, abra `src/server.ts` e clique em uma linha qualquer. Que informação
aparece no fim da linha? De onde ela veio?

**5. Pergunta para responder por escrito**
Qual a diferença entre `git add` e `git commit`? Use a analogia do palco e da foto, com suas
palavras. E por que existem dois passos separados, em vez de um só?

---

## Glossário e FAQ do Estagiário 📚

Para não deixar nenhuma lacuna, aqui vai uma sessão de perguntas frequentes para revisar tudo.

**P: Eu esqueci de configurar meu e-mail e tentei fazer o commit. E agora?**

R: O terminal vai mostrar um erro dizendo `Please tell me who you are`. Basta rodar os comandos `git config --global user.name` e `git config --global user.email`, conforme ensinamos no Capítulo 3, e tentar o commit novamente.

**P: O que acontece se eu esquecer o `.gitignore` e subir arquivos confidenciais pro GitHub?**

R: Este é um cenário assustador. Se o repositório for público, qualquer bot na internet raspará suas senhas em segundos. Se isso ocorrer, você NÃO DEVE apenas tentar deletar o arquivo no próximo commit. Você deve considerar que as senhas estão comprometidas e ir em todos os serviços afetados (banco de dados, servidores de e-mail, painéis na nuvem) e **trocar ou invalidar** todas as senhas imediatamente.

**P: Eu posso usar a interface visual do VS Code e ignorar o terminal para sempre?**

R: Poder você pode, mas não deve. Em processos seletivos e entrevistas técnicas, é muito comum pedirem que você explique verbalmente a sequência correta de comandos no terminal (init, add, commit, push). Conhecer os fundamentos é o que diferencia um dev júnior mediano de um futuro desenvolvedor sênior respeitado.

**P: Existe alguma maneira de desfazer um commit logo depois que eu fiz ele?**

R: Sim! Uma das grandes vantagens do Git é justamente o "Desfazer". Se você fez o commit, percebeu que esqueceu um arquivo ou escreveu a mensagem errada (mas **AINDA NÃO** deu o push para a nuvem), você pode usar um comando mais avançado chamado `git reset --soft HEAD~1`. Isso vai desmanchar a "foto", mas vai preservar todos os arquivos modificados na sua tela para que você possa arrumá-los e comitar novamente com segurança.

**P: Eu tenho que fazer commit a cada linha escrita?**

R: Não. O ideal é que cada foto/commit englobe uma **mudança lógica completa e coesa**. Por exemplo, terminei de estilizar o botão de carrinho de compras e testei? Faça um commit. Terminei de criar a rota da API inteira no backend? Faça um commit. Evite comitar partes quebradas, comite pequenas vitórias que já funcionam.

**P: Quando devo usar o Git Pull?**

R: Nós não usamos muito nesta aula inicial, mas o `git pull` é o comando inverso do `push`. Se você trabalha em equipe, antes de começar o seu dia, você deve rodar `git pull` para **baixar** todas as atualizações que seus colegas mandaram para a nuvem durante a noite.

**P: O que é o tal de "clone" que todo mundo fala?**

R: O `git clone` é o comando que você roda quando entra em uma empresa nova. Você pega o link de um repositório que já existe na nuvem, roda `git clone [link]` e ele baixa a pasta do projeto inteirinha para o seu HD.

**P: E se eu comitar na branch errada?**

R: Acontece com os melhores devs! Se você comitou na branch `main` mas deveria estar em outra, você pode criar a branch nova agora com `git branch nova-branch`, depois remover o commit da main com `git reset --hard HEAD~1` (cuidado, isso desfaz a mudança na main local) e então ir para a branch correta com `git checkout nova-branch`. Mas para iniciantes, o mais seguro é pedir ajuda ao sênior antes de rodar comandos com `--hard`.

---

## Galeria do Estagiário: Commits Ruins vs Commits Bons 🖼️

Para garantir que você nunca passe vergonha na hora de mandar o seu código para o tech lead aprovar, preparamos esta pequena tabela comparativa. O segredo é sempre usar o modo imperativo ("adiciona", "cria", "remove") e ser direto ao ponto.

| ❌ Commit Ruim (Não use!) | ✅ Commit Bom (Padrão Sênior)                             | 💡 Por que é melhor?                                                                 |
| :------------------------ | :-------------------------------------------------------- | :----------------------------------------------------------------------------------- |
| `arrumei o bug da tela`   | `fix: resolve erro no carregamento da tela inicial`       | Usa a tag `fix` e especifica **qual** erro foi resolvido e **onde**.                 |
| `mais coisas`             | `feat: adiciona componente de carrinho de compras`        | Usa `feat` para novidades e deixa claro o que foi adicionado.                        |
| `ajeitando p/ o deploy`   | `chore: atualiza versao das dependencias no package.json` | Mostra que é apenas manutenção (`chore`) e diz exatamente o que atualizou.           |
| `apaguei uns arquivos ai` | `refactor: remove arquivos de teste desnecessarios`       | Usa `refactor` para limpeza e diz o motivo da remoção.                               |
| `teste 123`               | `docs: corrige erro de digitacao nas anotacoes`           | Documenta que foi apenas uma mudança de texto. Commits de "teste" sujam o histórico! |

---

## Dicas de Sobrevivência para o Estagiário 🛠️

Para fechar com chave de ouro, aqui vão dicas que ninguém te ensina na faculdade, mas que salvam o seu dia no estágio:

**1. A Regra de Ouro do Commit Atômico**
Não passe 3 semanas programando para fazer um único commit com a mensagem "fiz o sistema inteiro". Faça commits pequenos (atômicos) e frequentes. Criou a tela de login? Commit. Conectou o banco de dados? Commit. Arrumou a cor do botão? Commit. Isso facilita muito caso precise reverter apenas uma das coisas.

**2. O Terminal não morde!**
Se o terminal ficar travado ou aparecer uma tela estranha com um til `~` (provavelmente o editor Vim abriu porque você esqueceu o `-m` no commit), não precisa desligar o computador na tomada! Basta apertar `Esc`, digitar `:q!` e apertar `Enter`. Isso fecha o editor sem salvar e devolve seu terminal.

**3. Nunca faça `push -f` (Force Push)**
A menos que um desenvolvedor sênior esteja ao seu lado e mande você fazer, evite usar a flag `-f` (force) no seu `git push`. Esse comando força a nuvem a aceitar o seu código e pode apagar silenciosamente o trabalho dos seus colegas que estava lá.

**4. Inglês salva vidas**
Acostume-se a ler as mensagens de erro que o Git joga na tela. 99% das vezes, o próprio Git diz exatamente qual é o problema e como resolver, mas como a mensagem está em inglês, muitos iniciantes ignoram e vão direto pro Google. Tente ler o erro primeiro!

**5. O poder do `.gitkeep`**
O Git tem uma peculiaridade: ele não rastreia pastas vazias. Se você criar uma pasta chamada `arquivos_importantes` e ela estiver vazia, o Git vai ignorá-la. Para forçar o Git a versionar uma pasta vazia, os devs costumam criar um arquivo vazio dentro dela chamado `.gitkeep`.

**6. Deixe o código melhor do que encontrou**
O versionamento mostra quem foi a última pessoa a mexer no arquivo (comando `git blame`). Se você abrir um arquivo e ver que o código está confuso ou sem indentação, arrume rapidamente antes de fazer a sua feature. Seus colegas de equipe vão notar e agradecer.

**7. Mantenha a calma nos conflitos**
Merge conflicts (conflitos de junção) vão acontecer. O VS Code vai pintar a tela de verde, azul e roxo e perguntar "Accept Current Change" ou "Accept Incoming Change". Não clique desesperado. Leia as duas partes do código, entenda o que você fez e o que seu colega fez, e escolha com calma, ou chame ele no chat para decidirem juntos.

**8. O GitHub não é apenas um cofre**
Explore o GitHub da comunidade! Você pode buscar projetos parecidos com o seu, ver como devs mais experientes estruturam os arquivos, quais bibliotecas eles usam e ler o código fonte de ferramentas famosas (como o próprio React ou o VS Code, que são open-source).

**9. Leia o README**
Todo projeto de qualidade tem um arquivo chamado `README.md` na pasta principal — o seu já tem, você criou no Passo 4 do Capítulo 7. Ele serve como o manual de instruções do projeto. Sempre que entrar em um projeto novo no estágio, a primeira coisa que você deve fazer é ler o README de cima a baixo.

**10. Faça pausas e beba água**
Acredite, 80% dos bugs cabeludos que você cria (ou erros de Git que você comete) acontecem quando você está programando há mais de 2 horas seguidas sem levantar da cadeira. O cérebro cansa, você esquece de dar o `git add`, comita na branch errada, etc. Levante, pegue um café e volte.

Parabéns pela dedicação e por chegar até o fim deste longo guia, estagiários!

Vocês estão no caminho certíssimo para se tornarem desenvolvedores incríveis. Estão prontos para brilhar na nossa equipe! 🚀
