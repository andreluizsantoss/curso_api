# Gabarito — Aula 02

---

## 1. Leia o seu próprio histórico

Não existe número certo de commits. O que importa é o teste que você fez: **conseguir explicar
cada mudança lendo só a mensagem**.

Se alguma mensagem não passou nesse teste, provavelmente faltou uma destas coisas:

| Faltou                | Exemplo ruim         | Como ficaria bom                                  |
| :-------------------- | :------------------- | :------------------------------------------------ |
| Dizer **o que** mudou | `chore: ajustes`     | `chore: adiciona script de build no package.json` |
| Dizer **onde**        | `fix: corrige erro`  | `fix: corrige porta padrão no server`             |
| Usar o **tipo** certo | `atualiza anotacoes` | `docs: registra o passo a passo do primeiro push` |

**Regra prática:** a mensagem precisa fazer sentido para você daqui a seis meses, quando você
não lembrar mais de nada do contexto.

---

## 2. O ciclo completo, sozinho

```bash
git status
#   Mostra ANOTACOES.md em vermelho, na seção "Untracked files"

git add ANOTACOES.md
#   Nada é impresso. Silêncio, aqui, é sucesso.

git status
#   Agora o ANOTACOES.md aparece em verde, em "Changes to be committed"

git commit -m "docs: registra as anotacoes da aula 02"
#   Mostra o resumo: 1 file changed, 1 insertion(+)

git push
#   Envia para o GitHub
```

**Por que rodar `git status` duas vezes:** para **ver** o arquivo mudar de vermelho para
verde. Essa é a diferença entre "arquivo alterado" e "arquivo selecionado para entrar na
próxima foto". Enquanto isso não estiver claro na sua cabeça, vale repetir.

**Por que dessa vez ele aparece em "Untracked files":** o `ANOTACOES.md` é um arquivo novo,
que o Git nunca viu antes. Um arquivo que já está versionado e foi editado apareceria em
"Changes not staged for commit". Nos dois casos o remédio é o mesmo — `git add` —, mas vale
reparar na diferença, porque as duas mensagens vão aparecer no seu dia a dia.

> [!TIP]
> Note que você não precisou avisar o Git de nada: bastou o arquivo existir dentro da pasta
> do projeto para ele aparecer no `git status`. É exatamente por isso que o `.gitignore` do
> Capítulo 4 é tão importante — sem ele, o `node_modules/` inteiro apareceria aqui também.

---

## 3. Teste o guarda-costas

**O Git não mostra o `.env`.** Ele simplesmente não aparece no `git status`.

**Motivo:** o arquivo `.gitignore` tem uma linha escrita `.env`. Ela manda o Git ignorar
completamente qualquer arquivo com esse nome — ele nunca é oferecido, nunca vai para o palco,
nunca entra numa foto.

**Por que isso é a proteção mais importante do projeto:** o `.env` é onde ficam senhas de
banco de dados, chaves de API e segredos de autenticação. Se ele fosse para um repositório
público, robôs que varrem o GitHub encontrariam essas senhas **em segundos**. Não é exagero:
existem programas rodando 24 horas por dia procurando exatamente isso.

> [!CAUTION]
> **Se um segredo já subiu**, apagar o arquivo no próximo commit **não resolve**. O Git guarda
> histórico: o valor continua acessível em qualquer commit anterior. A única resposta correta
> é considerar aquela senha comprometida e **trocá-la** em todos os serviços afetados,
> imediatamente. Depois disso, sim, limpar o histórico — com ajuda de alguém experiente.

---

## 4. Investigue com o GitLens

No fim da linha aparece, em cinza claro, algo como:

```
Você, 2 horas atrás • feat: cria estrutura inicial da API
```

Essa informação vem do **histórico do Git**, o mesmo que você vê com `git log`. O GitLens
apenas cruza cada linha do arquivo com o commit que a alterou pela última vez, e mostra ali
mesmo, sem você precisar rodar comando nenhum.

**Onde isso salva o seu dia:** você abre um arquivo, encontra uma linha esquisita e pensa
"isso não faz sentido, vou apagar". O GitLens mostra que ela foi escrita há dois anos com a
mensagem `fix: contorna limite de tamanho do gateway`. Pronto — existe um motivo, e você
acabou de evitar reintroduzir um bug antigo.

---

## 5. `git add` × `git commit`

Resposta esperada, com suas palavras:

> O `git add` coloca os arquivos **no palco**: você escolhe quem vai aparecer na foto. O
> `git commit` **tira a foto** e a guarda no álbum, com uma legenda.
>
> São dois passos separados porque nem tudo que você alterou pertence à mesma mudança lógica.
> Se você corrigiu um bug e, no meio do caminho, também mexeu na documentação, dá para fazer
> dois commits separados: `git add` só do arquivo do bug, commit com `fix:`, depois `git add`
> da documentação, commit com `docs:`.
>
> Um único passo obrigaria a fotografar tudo junto, e o histórico perderia o sentido.

**Na prática:** é isso que permite os "commits atômicos" da dica de sobrevivência número 1.
Cada foto conta **uma** história completa — e é o que torna possível desfazer uma mudança
específica sem levar junto tudo o que foi feito no mesmo dia.
