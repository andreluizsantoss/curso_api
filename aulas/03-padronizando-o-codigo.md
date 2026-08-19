# 🧑‍🏫 Aula 03: O "Xerife" do Código e a Padronização Automática

E aí, equipe! Bem-vindos e bem-vindas à **Aula 03**. 🎉

Na Aula 01 construímos a API e na Aula 02 aprendemos a versionar e enviar o código para o
GitHub. Agora que somos **mais de uma pessoa mexendo nos mesmos arquivos**, aparece um
problema novo.

Imagine: você, na sua casa, gosta de usar aspas simples (`'texto'`). Seu colega prefere aspas
duplas (`"texto"`). Você não coloca ponto e vírgula no fim da linha; ele coloca. Você indenta
com 2 espaços; ele com 4.

Se cada um escrever do seu jeito, o projeto vira uma salada de frutas visual. E tem um efeito
colateral pior, que só aparece no Git: quando ele salvar um arquivo seu, o editor dele
reformata tudo, e o Git vai mostrar **200 linhas alteradas** quando na verdade só uma linha
mudou de verdade. Fica impossível revisar.

A solução não é brigar nem combinar regras de cabeça. É instalar **ferramentas automáticas**
que fazem isso por nós. Vamos lá! ☕

---

## 🎯 O que você vai conseguir fazer ao final desta aula

- Explicar a diferença entre um _linter_ e um _formatter_, e por que precisamos dos dois.
- Configurar ESLint e Prettier em um projeto TypeScript.
- Fazer o VS Code arrumar o código sozinho toda vez que você salvar.
- Entender o que são `peerDependencies` e por que às vezes precisamos fixar uma versão.
- Manter o `README.md` em dia com o que o projeto realmente faz.

## 📋 Pré-requisitos

- Aulas 01 e 02 concluídas.
- O comando `npm run build` funcionando no seu projeto.

---

## 📖 Capítulo 1: O que é o "Xerife" do Código?

Sabe quando você escreve um texto no editor de textos e ele sublinha uma palavra de
**vermelho** porque está escrita errada? Ou de **azul** porque faltou uma vírgula?

Isso é um corretor ortográfico e gramatical. Ele mantém o texto correto, padronizado e fácil
de ler para qualquer pessoa.

Na programação temos a mesma coisa, e ela se divide em duas categorias:

1. **Linters** — checam **qualidade e lógica**. Pegam erros de raciocínio e más práticas.
2. **Formatters** — cuidam da **aparência**: espaços, quebras de linha, aspas.

> [!NOTE]
> **Por que isso importa tanto?**
>
> Quando um time trabalha junto, o código precisa parecer que foi escrito por **uma única
> pessoa**, mesmo que dez tenham colocado a mão nele. Se cada arquivo tiver um formato
> diferente, o cérebro gasta energia decifrando a "letra" de cada um, em vez de focar no que
> importa: a lógica e as regras de negócio da API do Curso.

---

## 🦸 Capítulo 2: O Trio de Ferramentas

Vamos usar três ferramentas que trabalham juntas, cada uma com um papel bem definido.

### 1. EditorConfig 📝

Pense nele como "as regras do caderno da escola". É um arquivo simples que garante que — não
importa se você usa VS Code, WebStorm, Vim ou qualquer outro editor — a tecla `Tab` tenha
sempre o mesmo tamanho e os arquivos sejam salvos do mesmo jeito.

Ele é o mais básico dos três, mas evita que a configuração pessoal de cada um bagunce o
projeto dos outros.

### 2. Prettier ✨

"Prettier" significa "mais bonito" em inglês. Ele é o nosso **formatador automático**. A única
função dele é pegar o código que você escreveu, não importa o quão desalinhado esteja, e
deixá-lo com formatação impecável.

E a melhor parte: ele faz isso **no instante em que você salva o arquivo** (`Ctrl + S`).

### 3. ESLint 🕵️

Este é o xerife de verdade. Enquanto o Prettier cuida da _aparência_ (ele não liga se o
código funciona), o ESLint cuida da _qualidade_. Ele lê o seu TypeScript com uma lupa e avisa
sobre problemas reais, como:

- "Você criou a variável `nomeDoUsuario` e nunca usou ela em lugar nenhum."
- "Você importou este arquivo e não usou nada dele."
- "Você deixou um `console.log` esquecido no meio do código."
- "Você usou o tipo `any` aqui, o que desliga toda a proteção do TypeScript neste ponto."

> [!TIP]
> **Resumo para não esquecer:**
>
> - **EditorConfig** ajusta o editor.
> - **Prettier** cuida da aparência.
> - **ESLint** cuida da qualidade e da lógica.

### Por que manter Prettier e ESLint separados

Existe um jeito de fazer o Prettier rodar _por dentro_ do ESLint. Neste projeto **não
fazemos isso**, de propósito, e vale explicar o motivo.

Se juntássemos os dois, um espaço sobrando apareceria na sua tela com **exatamente a mesma
cor e o mesmo peso** de um bug de verdade. Quando tudo é vermelho, você aprende a ignorar o
vermelho — e um dia ignora o erro que importava.

Separados, a divisão fica clara: se o ESLint reclamou, **pare e leia**. Se foi só formatação,
o Prettier já resolveu sozinho e você nem ficou sabendo.

---

## 📦 Capítulo 3: Instalando as ferramentas

Abra o terminal integrado do VS Code, confirme que está dentro da pasta
`curso_api` e rode:

```bash
npm install -D eslint @eslint/js typescript-eslint prettier eslint-config-prettier
```

### O que cada pacote faz

- `eslint` — o xerife principal.
- `@eslint/js` — o conjunto de regras básicas recomendadas para JavaScript.
- `typescript-eslint` — ensina o vocabulário do TypeScript ao ESLint. Sem isso ele não
  entenderia arquivos `.ts`, porque o ESLint nasceu antes do TypeScript existir.
- `prettier` — o formatador.
- `eslint-config-prettier` — o **apaziguador de brigas**. O ESLint tem algumas regras próprias
  sobre aparência que conflitariam com o Prettier. Este pacote desliga essas regras e deixa o
  Prettier cuidar da aparência sozinho.

> [!IMPORTANT]
>
> ### 🔍 Por que o nosso projeto usa TypeScript 6, e não a versão mais nova?
>
> Lembra que na Aula 01 instalamos `typescript@6` de propósito? Chegou a hora de você
> conferir o motivo com as próprias mãos. Rode:
>
> ```bash
> npm view typescript-eslint peerDependencies
> ```
>
> A resposta vai ser parecida com esta:
>
> ```
> {
>   eslint: '^8.57.0 || ^9.0.0 || ^10.0.0',
>   typescript: '>=4.8.4 <6.1.0'
> }
> ```
>
> **Traduzindo:** o `typescript-eslint` está declarando com quais versões dos outros pacotes
> ele sabe trabalhar. Repare em `typescript: '>=4.8.4 <6.1.0'` — ele funciona com TypeScript a
> partir da versão 4.8.4 e **abaixo** da 6.1.
>
> Isso se chama **`peerDependencies`** (dependências de convivência). Se tivéssemos instalado
> o TypeScript 7, o comando `npm run lint` simplesmente não rodaria: ele mostraria um erro
> dizendo que a versão não é suportada.
>
> **Por que isso é uma aula em si:** no mundo real, as ferramentas evoluem em ritmos
> diferentes. A versão mais nova nem sempre é a melhor escolha — a melhor escolha é a
> combinação que **funciona junto**. Saber ler `peerDependencies` te poupa horas de
> frustração, e é o tipo de conhecimento que separa quem trava de quem resolve.
>
> Quando o `typescript-eslint` publicar suporte à versão 7, atualizamos.

---

## 🛠️ Capítulo 4: Criando os Arquivos de Configuração

Os pacotes estão instalados, mas ainda não sabem as regras do nosso time. Vamos criar os
arquivos de configuração na raiz do projeto, junto do `package.json`.

### Passo 1: As regras do editor (`.editorconfig`)

Crie um arquivo chamado **exatamente** `.editorconfig` (começa com ponto e não tem extensão):

```ini
# Informa ao editor que este é o arquivo principal de configuração.
# Ele não deve procurar outro .editorconfig em pastas acima desta.
root = true

# Regras válidas para todos os arquivos (o asterisco significa "tudo")
[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
trim_trailing_whitespace = true
insert_final_newline = true

# Markdown é a exceção: dois espaços no fim da linha significam "quebre a linha aqui".
# Se o editor apagasse esses espaços, a formatação dos nossos textos quebraria.
[*.md]
trim_trailing_whitespace = false
```

**O que ele está mandando o editor fazer:** salvar sempre em UTF-8, usar 2 espaços na
indentação, remover espaços sobrando no fim das linhas e deixar sempre uma linha vazia no fim
do arquivo (o Git gosta disso).

Repare no bloco `[*.md]`. No Markdown — a linguagem em que estas aulas são escritas — dois
espaços no fim da linha têm significado: são uma quebra de linha. Se apagássemos, a formatação
do texto quebraria. **Regra geral com exceção justificada** é algo que você vai ver muito.

### Passo 2: As regras de aparência (`.prettierrc.json`)

Crie o arquivo `.prettierrc.json`:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "embeddedLanguageFormatting": "off"
}
```

**Cada escolha:**

- `"semi": false` — sem ponto e vírgula no fim das linhas. O JavaScript moderno não precisa
  na esmagadora maioria dos casos, e o código fica mais limpo.
- `"singleQuote": true` — aspas simples (`'texto'`). É o padrão mais comum na comunidade Node.
- `"trailingComma": "all"` — coloca vírgula depois do último item de listas e objetos que
  quebram linha. Parece bobagem, mas ajuda muito na hora de ler as mudanças no Git: adicionar
  um item novo passa a alterar **uma** linha em vez de duas.
- `"printWidth": 100` — quebra linhas com mais de 100 caracteres, para ninguém precisar rolar
  a tela para o lado.
- `"tabWidth": 2` — reforça a indentação de 2 espaços.
- `"embeddedLanguageFormatting": "off"` — impede que o Prettier formate o código escrito
  **dentro** de blocos em arquivos Markdown.

> [!NOTE]
> Essa última opção merece atenção. Um arquivo Markdown — como o `README.md` — pode conter
> blocos de código dentro dele. Sem essa opção, o Prettier entraria nesses blocos e os
> reformataria.
>
> Na maior parte do tempo isso seria até bem-vindo. Mas quando o bloco existe justamente para
> mostrar um exemplo **propositalmente mal escrito**, a formatação automática destrói a
> demonstração. É um caso real de "a ferramenta está certa, mas o nosso contexto pede uma
> exceção" — e o jeito correto de tratar isso é registrar a exceção na configuração, com o
> motivo escrito, e não desligar a ferramenta inteira.

> [!TIP]
> **Por que `.prettierrc.json` e não só `.prettierrc`?**
>
> Você vai encontrar muitos tutoriais usando `.prettierrc`, sem extensão. Funciona, mas tem
> uma armadilha divertida: sem a extensão, o Prettier não tem certeza se o arquivo é JSON ou
> YAML — e, ao formatar a si mesmo, ele aplica as regras que você acabou de definir (aspas
> simples, vírgula no final) e transforma o arquivo em algo que **não é mais JSON válido**.
>
> Com `.json` no nome, não há ambiguidade. Uma letra a mais no nome do arquivo, um problema
> a menos para depurar depois.

### Passo 3: O que o Prettier NÃO deve tocar (`.prettierignore`)

Crie o arquivo `.prettierignore`:

```
# =============================================
# Código gerado pelo compilador
# =============================================
dist/

# =============================================
# Dependências
# =============================================
node_modules/

# =============================================
# Arquivos gerados automaticamente pelo npm
# =============================================
package-lock.json
```

**Por que isso é necessário:** sem este arquivo, o comando de formatação tentaria reformatar
os milhares de arquivos dentro de `node_modules` e o código gerado em `dist`. Além de demorar
uma eternidade, não faz sentido nenhum — ninguém lê esses arquivos.

### Passo 4: As regras de qualidade (`eslint.config.js`)

Crie o arquivo `eslint.config.js`:

```javascript
// @ts-check

/**
 * Configuração do ESLint (formato "flat config").
 *
 * O ESLint é o nosso analisador de qualidade: ele procura problemas de lógica e
 * más práticas. Quem cuida da aparência do código (espaços, aspas, quebras de
 * linha) é o Prettier, executado separadamente pelo comando `npm run format`.
 *
 * Manter as duas ferramentas separadas é uma decisão consciente: assim um espaço
 * sobrando nunca aparece com a mesma gravidade visual de um bug de verdade.
 */

import { defineConfig, globalIgnores } from 'eslint/config'
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import prettier from 'eslint-config-prettier'

export default defineConfig([
  // Pastas que o ESLint nunca deve analisar: código gerado e bibliotecas de terceiros.
  globalIgnores(['dist/**', 'node_modules/**']),

  {
    files: ['**/*.ts'],
    extends: [
      js.configs.recommended,
      tseslint.configs.recommended,
      // Precisa ser o ÚLTIMO da lista: desliga as regras do ESLint que brigariam
      // com a formatação do Prettier.
      prettier,
    ],
    rules: {
      // O tipo `any` desliga a checagem do TypeScript naquele ponto. Às vezes é
      // inevitável, por isso é aviso e não erro — mas precisa ser uma decisão
      // consciente, nunca um descuido.
      '@typescript-eslint/no-explicit-any': 'warn',

      // Em produção usamos o logger do Fastify (`app.log`), que gera JSON
      // estruturado e pode ser filtrado por nível. `console.log` escreve texto
      // solto, que as ferramentas de monitoramento não conseguem indexar.
      //
      // É `error`, e não `warn`: aviso que não reprova nada é aviso que o time
      // aprende a ignorar. As exceções legítimas continuam possíveis, mas exigem
      // um `eslint-disable` com o motivo escrito ao lado.
      'no-console': 'error',

      // Variável declarada e não usada quase sempre indica código morto ou um
      // erro de digitação. O prefixo `_` marca o caso em que ignorar é proposital.
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_', varsIgnorePattern: '^_' },
      ],
    },
  },
])
```

**Entendendo a estrutura:**

- **`defineConfig([...])`** — este é o formato atual de configuração do ESLint, chamado
  _flat config_. Ele substituiu um formato antigo baseado em `.eslintrc.json`. Se você
  encontrar tutoriais na internet mostrando `.eslintrc`, saiba que estão desatualizados.
- **`globalIgnores([...])`** — pastas que o ESLint nunca deve olhar.
- **`extends: [...]`** — a parte mais poderosa. Aqui dizemos: "use as regras recomendadas do
  JavaScript, as recomendadas do TypeScript e, por último, desligue o que brigaria com o
  Prettier". É como entregar ao xerife os melhores manuais já escritos.
- **A ordem importa:** o `prettier` fica por **último** porque a função dele é _desligar_
  regras que os anteriores ligaram. Se viesse antes, os outros religariam tudo depois dele.

  > [!NOTE]
  > Sendo honesto com você: **hoje, neste projeto, mudar essa ordem não causa diferença
  > nenhuma.** As versões atuais do ESLint tiraram as regras de formatação dos conjuntos
  > recomendados, então o `eslint-config-prettier` quase não tem o que desligar.
  >
  > Ele continua ali por segurança: no dia em que alguém adicionar um conjunto de regras que
  > se meta com formatação, ele já está no lugar certo para evitar o conflito. Você vai
  > comprovar isso sozinho no exercício 4.
  >
  > Guarde a ideia: em programação, muita coisa que "todo mundo faz" continua sendo repetida
  > depois que o motivo original deixou de existir. Vale sempre conferir em vez de aceitar.

- **`rules: {...}`** — aqui ajustamos regras específicas do nosso time. Repare que cada uma
  tem um comentário explicando **por que** ela existe. Configuração sem justificativa vira
  mistério para quem chegar depois.

  > [!NOTE]
  > Repare que duas regras têm gravidades diferentes, e isso é decisão, não acaso.
  >
  > O `no-explicit-any` é **aviso**: usar `any` às vezes é inevitável, e o aviso serve para
  > que seja uma escolha consciente. Já o `no-console` é **erro**, porque `console.log` em
  > código de servidor nunca é a resposta certa — existe `app.log` para isso.
  >
  > A diferença prática: erro faz o `npm run check` inteiro falhar; aviso não. E aviso que
  > não falha nada é aviso que, em pouco tempo, todo mundo aprende a ignorar. O exercício 2
  > desta aula faz você sentir isso na prática.

> [!NOTE]
> **`'warn'` ou `'error'`, qual a diferença?**
>
> - `'warn'` mostra um aviso amarelo, mas o comando termina com sucesso.
> - `'error'` mostra um erro vermelho e o comando **falha**.
>
> Isso importa toda vez que você roda o `npm run check`, que vamos criar daqui a pouco: ele
> encadeia as verificações com `&&`, e um `error` **interrompe a sequência ali mesmo** — o
> build nem chega a rodar. Um `warn` deixa passar. É a diferença entre "isto não entra no
> projeto" e "olhe para isto quando puder".

---

## 🏃 Capítulo 5: Criando os Atalhos no package.json

A configuração está pronta, mas seria chato digitar comandos longos toda vez. Vamos criar
atalhos.

Abra o `package.json` e deixe **exatamente** assim (as novidades estão em `scripts`):

```json
{
  "name": "curso_api",
  "version": "1.0.0",
  "description": "API RESTful backend do curso",
  "main": "dist/server.js",
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "prebuild": "node --eval \"require('node:fs').rmSync('dist', { recursive: true, force: true })\"",
    "build": "tsc",
    "start": "node dist/server.js",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "check": "npm run lint && npm run format:check && npm run build"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "fastify": "^5.12.0"
  },
  "devDependencies": {
    "@eslint/js": "^10.0.1",
    "@types/node": "^26.2.0",
    "eslint": "^10.8.1",
    "eslint-config-prettier": "^10.1.8",
    "prettier": "^3.9.6",
    "tsx": "^4.23.12",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.67.0"
  }
}
```

**Os cinco comandos novos:**

| Comando                | O que faz                                                                                            |
| :--------------------- | :--------------------------------------------------------------------------------------------------- |
| `npm run lint`         | **Fofoca, mas não encosta.** Varre a pasta `src` e lista os problemas encontrados, sem alterar nada. |
| `npm run lint:fix`     | **Faz a faxina.** Encontra e corrige tudo o que for corrigível automaticamente.                      |
| `npm run format`       | Formata todos os arquivos do projeto com o Prettier.                                                 |
| `npm run format:check` | Confere a formatação e **avisa** o que está fora do padrão, sem alterar nada.                        |
| `npm run check`        | Roda os três: lint, verificação de formatação e build. **É este que você roda antes de commitar.**   |

> [!TIP]
> Repare no `&&` do script `check`. Ele significa "só execute o próximo se o anterior deu
> certo". Se o lint falhar, o build nem chega a rodar — e o comando inteiro falha. É o que
> transforma três verificações separadas em uma resposta única: passou ou não passou.

---

## 🧪 Capítulo 6: A Hora da Verdade!

Não adianta configurar tudo isso e não ver funcionando. Vamos fazer um experimento.

### Passo 1: Confirme que o projeto está limpo

Antes de bagunçar, vamos ver o estado atual:

```bash
npm run lint
```

Você vai ver só o cabeçalho que o npm sempre imprime, com o nome do comando:

```
> curso_api@1.0.0 lint
> eslint src
```

**E mais nada depois dele.** Esse silêncio do ESLint é o sinal de sucesso: nenhum problema
encontrado.

### Passo 2: Criando um arquivo bagunçado de propósito

> [!CAUTION]
> **Nunca** bagunce um arquivo que já funciona só para testar ferramenta. Vamos criar um
> arquivo descartável, e apagá-lo no fim.

Crie o arquivo `src/exemplo-bagunca.ts` e cole este código horroroso:

```typescript
import   Fastify   from "fastify";

const   naoUsada   =   "esta variável nunca é usada";

export function exemploRuim( dados : any ) {
      const app = Fastify({ logger:true })

  app.get( "/exemplo" ,async (request,reply)=> {
return reply.send({  caminho: request.url,  mensagem:  dados})
  })

    console.log( "servidor de exemplo" ) ;

  return app
}
```

Se o seu olho doeu, parabéns: você está desenvolvendo senso crítico de programador. 😄

Salve o arquivo **sem** formatar. (Se o seu VS Code formatar sozinho ao salvar, é uma
configuração pessoal sua de antes desta aula — pode seguir assim mesmo, o experimento funciona
igual. A configuração deste projeto, que faz isso valer para o time todo, é a do Capítulo 7.)

### Passo 3: Chamando o Xerife

```bash
npm run lint
```

Você vai ver exatamente isto:

```
src/exemplo-bagunca.ts
   3:9   error    'naoUsada' is assigned a value but never used. Allowed unused vars must match /^_/u  @typescript-eslint/no-unused-vars
   5:38  warning  Unexpected any. Specify a different type                                             @typescript-eslint/no-explicit-any
  12:5   error    Unexpected console statement                                                         no-console

✖ 3 problems (2 errors, 1 warning)
```

**Leia com atenção**, porque aqui está a lição do dia:

- Os dois **erros** (`error`) são a variável `naoUsada` e o `console.log`. O primeiro é
  código morto; o segundo escreve texto solto onde o projeto usa `app.log`, que gera log
  estruturado — como você aprendeu na Aula 01. Erro faz o comando **falhar**.
- O **aviso** (`warning`) é o `any`. Ele não derruba o comando, porque usar `any` às vezes é
  inevitável — mas fica registrado, para ser uma decisão consciente e não um descuido.
- **Repare no que o ESLint NÃO reclamou:** dos espaços tortos, das aspas duplas, do ponto e
  vírgula. Isso não é trabalho dele. É do Prettier.

### Passo 4: Chamando o Formatador

```bash
npm run format
```

Volte ao arquivo `src/exemplo-bagunca.ts`. **Está irreconhecível — no bom sentido:**

```typescript
import Fastify from 'fastify'

const naoUsada = 'esta variável nunca é usada'

export function exemploRuim(dados: any) {
  const app = Fastify({ logger: true })

  app.get('/exemplo', async (request, reply) => {
    return reply.send({ caminho: request.url, mensagem: dados })
  })

  console.log('servidor de exemplo')

  return app
}
```

Espaços alinhados, aspas simples, sem ponto e vírgula, indentação certinha. E você não
apertou a tecla de apagar uma única vez.

### Passo 5: Corrigindo o que o Xerife apontou

```bash
npm run lint:fix
```

Rode `npm run lint` de novo. Os dois erros e o aviso continuam lá.

**E isso é proposital.** O `--fix` corrige o que dá para corrigir **sem risco**. Apagar uma
variável, remover um `console.log` ou escolher um tipo no lugar do `any` são decisões suas:
talvez você fosse usar aquilo e esqueceu, talvez seja lixo mesmo. A ferramenta não adivinha
intenção — ela aponta, você decide. Guarde isso.

### Passo 6: Limpando a bagunça

Apague o arquivo `src/exemplo-bagunca.ts`. Ele cumpriu seu papel.

Agora rode o comando definitivo:

```bash
npm run check
```

Os três — lint, formatação e build — devem passar em sequência, sem nenhum erro.

---

## 🚀 Capítulo 7: A Mágica Final no VS Code

Digitar `npm run format` o tempo todo é cansativo. Vamos ensinar o VS Code a fazer isso
**sozinho toda vez que você salvar**.

### Passo 1: Instale as três extensões

Abra a aba de extensões (`Ctrl + Shift + X`) e instale:

| Extensão         | Identificador               | O que faz                                                                                             |
| :--------------- | :-------------------------- | :---------------------------------------------------------------------------------------------------- |
| **ESLint**       | `dbaeumer.vscode-eslint`    | Mostra os avisos do xerife sublinhados no código, enquanto você digita — sem precisar rodar o comando |
| **Prettier**     | `esbenp.prettier-vscode`    | Formata ao salvar                                                                                     |
| **EditorConfig** | `EditorConfig.EditorConfig` | Faz o editor obedecer ao `.editorconfig`                                                              |

### Passo 2: Configure o editor para este projeto

Crie a pasta `.vscode` na raiz e, dentro dela, o arquivo `settings.json`:

```json
{
  // Formata o arquivo toda vez que você salva (Ctrl + S).
  "editor.formatOnSave": true,

  // Quem formata é o Prettier. O ESLint cuida de qualidade, não de aparência.
  "editor.defaultFormatter": "esbenp.prettier-vscode",

  // Ao salvar, o ESLint também corrige sozinho o que for corrigível
  // (variável não usada, import fora de ordem, etc.).
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },

  // Linha vertical marcando os 100 caracteres do `printWidth` do Prettier.
  // Ajuda a perceber quando a linha está ficando longa demais antes de salvar.
  "editor.rulers": [100],

  // Mesmo final de linha do `.editorconfig`. Sem isso, quem usa Windows gera
  // arquivos que aparecem como "totalmente alterados" no Git.
  "files.eol": "\n",

  // Some com as pastas geradas na busca de arquivos (Ctrl + P) e no "localizar
  // em todos os arquivos". Elas têm milhares de arquivos que não interessam.
  "search.exclude": {
    "**/dist": true,
    "**/node_modules": true
  }
}
```

> [!IMPORTANT]
> Repare que este arquivo fica **dentro do projeto**, e não nas configurações pessoais do seu
> VS Code. Isso é de propósito: assim a configuração vai para o Git e **todo o time recebe as
> mesmas regras automaticamente**. Ninguém precisa configurar nada na mão.

### Passo 3: Registre as extensões recomendadas

Ainda na pasta `.vscode`, crie o arquivo `extensions.json`:

```json
{
  // Quando alguém clona este repositório e abre no VS Code, ele sugere instalar
  // estas extensões automaticamente. Ninguém precisa lembrar de avisar.
  //
  // Esta lista cresce junto com o projeto: cada ferramenta nova que entra
  // acrescenta aqui a extensão correspondente.
  "recommendations": [
    // --- TypeScript: ler erro e testar rota sem sair do editor ---
    "YoavBls.pretty-ts-errors",
    "usernamehw.errorlens",
    "humao.rest-client",
    "streetsidesoftware.code-spell-checker",
    "streetsidesoftware.code-spell-checker-portuguese-brazilian",

    // --- Git: ver quem alterou cada linha, e quando ---
    "eamodio.gitlens",

    // --- Padronização do código ---
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "EditorConfig.EditorConfig"
  ]
}
```

> [!NOTE]
> Os comentários agrupando por aula não são enfeite: eles dizem **de onde veio cada
> extensão**. Daqui a seis meses, quando alguém perguntar por que a lista tem uma entrada
> estranha, a resposta já está escrita ao lado. As próximas aulas acrescentam novos blocos
> aqui, seguindo esse mesmo formato.

Agora, quando alguém clonar este repositório, o VS Code **sugere sozinho** todas as extensões
certas. Aquele processo de "manda no chat qual extensão eu instalo?" simplesmente deixa de
existir.

### Passo 4: Testando a mágica

Crie de novo um arquivo bagunçado qualquer, com espaços tortos e aspas duplas. Aperte
`Ctrl + S`.

**Pronto. Ele se arruma sozinho no ato do salvamento.** Depois é só apagar o arquivo de teste.

---

## 📄 Capítulo 8: Atualizando o README

Na Aula 02 você escreveu o `README.md`, o cartão de visitas do projeto. Ele listava três
comandos, porque três era tudo o que existia.

Hoje você criou **cinco comandos novos**. Um colega que abrir o repositório amanhã vai ler o
README, achar que o projeto tem três comandos e nunca descobrir que existe um `npm run check`.

> [!IMPORTANT]
> Documentação desatualizada é pior do que documentação nenhuma. Sem README, a pessoa
> pergunta. Com um README errado, ela **confia** e faz a coisa errada.
>
> Por isso, a regra que vale desta aula em diante: **quem acrescenta um comando, acrescenta a
> linha no README na mesma tarefa.** Não é capricho — é a única forma de o arquivo continuar
> verdadeiro depois de dez aulas.

Abra o seu `README.md` e deixe-o exatamente assim (as linhas novas são as cinco últimas da
tabela de comandos):

````markdown
# API do Curso

API RESTful do curso, construída com **Fastify + TypeScript**.

## Começando

```bash
npm install     # baixa as dependências
npm run dev     # sobe a API em http://localhost:3333
```

**Pré-requisito:** Node.js na versão registrada no `.nvmrc`.

## Comandos

| Comando                | O que faz                                         |
| :--------------------- | :------------------------------------------------ |
| `npm run dev`          | Sobe a API recarregando a cada alteração salva    |
| `npm run build`        | Compila o TypeScript para a pasta `dist`          |
| `npm start`            | Executa a versão compilada, como roda em produção |
| `npm run lint`         | Procura problemas de lógica e qualidade no código |
| `npm run lint:fix`     | Corrige sozinho o que for corrigível              |
| `npm run format`       | Formata todos os arquivos com o Prettier          |
| `npm run format:check` | Confere a formatação sem alterar nada             |
| `npm run check`        | Roda lint, formatação e build em sequência        |

## Rotas

| Método | Rota      | O que devolve                              |
| :----- | :-------- | :----------------------------------------- |
| `GET`  | `/health` | O estado da API: status, uptime e ambiente |
````

> [!NOTE]
> O bloco começa e termina com **quatro** crases porque o conteúdo tem um bloco de três crases
> dentro. Copie apenas o que está **entre** as linhas de quatro crases — o arquivo começa em
> `# API do Curso`.

---

## 💾 Fechando o ciclo: mande para o GitHub

Você criou seis arquivos novos hoje e atualizou o README. Todos eles precisam ir para o
repositório — é o que faz a configuração valer para o time inteiro, e não só para você. Use o
ciclo que aprendeu na Aula 02:

```bash
git add .
git commit -m "chore: adiciona eslint, prettier e editorconfig"
git push
```

Confira no navegador que o `eslint.config.js` e a pasta `.vscode` aparecem lá.

---

## ✅ Como saber que deu certo

| Comando                        | O que esperar                                    |
| :----------------------------- | :----------------------------------------------- |
| `npm run lint`                 | O ESLint não lista nenhum problema               |
| `npm run format:check`         | Diz `All matched files use Prettier code style!` |
| `npm run build`                | Termina sem erros                                |
| `npm run check`                | Os três acima, em sequência, todos passando      |
| `Ctrl + S` em um arquivo `.ts` | O arquivo se formata sozinho                     |
| Abrir o `README.md`            | A tabela de comandos tem **oito** linhas         |

> [!CAUTION]
> Se algum falhar, resolva antes de seguir. Estes são os comandos que você vai rodar antes de
> cada commit, pelo resto do curso — e projeto que segue adiante quebrado só acumula
> problema: a cada aula nova, descobrir qual foi a causa fica mais caro.

---

## 🚨 Erros Comuns

**`Error: Cannot find module 'eslint/config'`**
Sua versão do ESLint é antiga. Rode `npm ls eslint` para conferir; precisa ser 9 ou superior.

**`typescript-eslint does not support TS 7.0`**
Você instalou o TypeScript 7. Volte para a versão 6 com `npm install -D typescript@6`.
Releia a caixa do Capítulo 3 — é exatamente o caso das `peerDependencies`.

**O VS Code não formata ao salvar**
Três causas possíveis, nesta ordem: a extensão do Prettier não está instalada; o arquivo
`.vscode/settings.json` está com erro de digitação (uma vírgula a mais quebra tudo); ou outra
extensão está definida como formatadora padrão. Abra a paleta com `Ctrl + Shift + P`, digite
"Format Document With..." e confira qual está selecionada.

**O ESLint sublinha o projeto inteiro de vermelho**
Provavelmente ele não está achando o `eslint.config.js`. Confirme que o arquivo está na
**raiz** do projeto, e não dentro de `src`.

**`npm run format` alterou arquivos que eu não queria**
Confira o `.prettierignore`. Ele deve conter `dist/`, `node_modules/` e `package-lock.json`.

---

## 🏋️ Exercícios

Gabarito em [`exercicios/03-gabarito.md`](./exercicios/03-gabarito.md).

**1. Erro ou aviso?**
Em `src/server.ts`, troque `app.log.info(...)` por `console.log('teste')`. Rode
`npm run lint`. Foi erro ou aviso? Agora rode `npm run check` — o comando inteiro falhou ou
passou? Explique por quê. **Não desfaça ainda:** o exercício 2 continua daqui.

**2. Afrouxando a regra**
Com o `console.log` ainda no lugar, mude `'no-console'` de `'error'` para `'warn'` no
`eslint.config.js` e rode `npm run lint` de novo. O que mudou na saída? E no
`npm run check`? Depois **volte para `'error'`** e desfaça o `console.log` do exercício 1.

**3. Investigação de versões**
Rode `npm view eslint version` e compare com a versão que está no seu `package.json`. Depois
rode `npm view typescript-eslint peerDependencies` e responda: se saísse hoje um ESLint 11,
nosso projeto poderia atualizar na hora? Como você descobriu isso?

**4. Desconfie da regra decorada**
Abra o `eslint.config.js` e mova o `prettier` para ser o **primeiro** item do `extends`, em
vez do último. Rode `npm run lint`.

O que aconteceu? Bate com o que você esperava depois de ler o Capítulo 4? Investigue: por que
o resultado foi esse? E, mesmo assim, por que continuamos deixando o `prettier` por último?
Depois volte como estava.

**5. Pergunta para responder por escrito**
Por que este projeto decidiu **não** rodar o Prettier por dentro do ESLint? Responda em duas
ou três frases, com suas palavras.

---

## 🎯 Resumo e Próximos Passos

Hoje vocês configuraram ferramentas de nível profissional. Todo projeto sério usa linter e
formatador — é assim que times grandes mantêm milhares de arquivos consistentes.

O que ficou pronto:

- O código fica sempre padronizado, sem esforço e sem discussão.
- Erros bobos e código morto são pegos **antes** do programa rodar.
- Todo o time compartilha a mesma configuração de editor, versionada no Git.
- Você entendeu `peerDependencies`, que é um dos motivos mais comuns de "não instala e não sei
  por quê".

**E agora?**

Repare em uma coisa: o projeto agora tem uma configuração escrita, e o computador cobra ela
de você. Isso é uma mudança de natureza — antes o padrão morava na cabeça de quem escrevia.

Falta a configuração do **ambiente**: a porta, o endereço, o modo de execução. Hoje esses
valores estão espalhados pelo código, e é assim que nasce o bug que só aparece na máquina de
outra pessoa.

Na **Aula 04** vamos tirar essa configuração de dentro do código e fazer a API recusar subir
quando ela estiver errada — em vez de subir torta e falhar mais tarde.

Até a próxima! 🚀
