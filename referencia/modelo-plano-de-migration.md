# Modelo — Plano de migration em produção

> Copie este arquivo, preencha **antes** de rodar qualquer migration em um banco com dado de
> gente, e deixe-o junto do Pull Request da alteração.
>
> Ele não existe por burocracia. Existe porque decidir com calma agora custa dez minutos, e
> improvisar com o sistema fora do ar custa o dia — e às vezes o dado.

---

## 1. O que muda

_Uma frase, em português, que qualquer pessoa da equipe entenda._

Exemplo: "a coluna `nome` passa a se chamar `nomeCompleto`".

**Migrations envolvidas:**

- `20260818202130_expande_nome_completo`
- `20260818202405_contrai_remove_nome`

---

## 2. É retrocompatível?

Marque uma:

- [ ] **Sim.** O código que está no ar hoje continua funcionando com o schema novo.
- [ ] **Não.** Vai haver janela em que o sistema fica indisponível ou responde errado.

**Se não for**, responda também:

- Por que não dá para fazer em etapas (expande/contrai)?
- Qual a janela estimada, e em que horário ela machuca menos?
- Quem precisa ser avisado antes?

> Uma migration não retrocompatível não é proibida — é uma decisão que precisa ser **tomada**,
> e não descoberta durante o deploy.

---

## 3. Backup

- Comando exato usado:

  ```bash
  docker compose exec mysql mysqldump -uroot -p<senha> --databases <banco> > backups/<arquivo>.sql
  ```

- Feito às: `__:__` de `__/__/____`
- Arquivo: `backups/____________________.sql` — tamanho: `____`
- **A restauração foi testada?** [ ] sim [ ] não

> Backup que nunca foi restaurado é esperança, não backup. Se a resposta for "não", teste antes
> de rodar a migration — não depois de precisar dele.

---

## 4. Como conferir que deu certo

_A consulta exata que vai ser rodada depois, e o resultado esperado._

```sql
-- exemplo
SELECT COUNT(*) AS total, COUNT(nomeCompleto) AS preenchidos FROM cidadaos;
-- esperado: os dois números iguais
```

---

## 5. Como voltar

_Comando a comando. Escrito antes, para não ser inventado sob pressão._

- [ ] Se a migration **falhar no meio**:
      `prisma migrate resolve --rolled-back "<nome_da_migration>"` e então corrigir a causa.
      Lembre: esse comando conserta o **registro**, não desfaz o efeito.
- [ ] Se o efeito precisar ser desfeito: qual migration compensatória será aplicada?
      (Não se apaga migration já aplicada — acrescenta-se a que a desfaz.)
- [ ] Se o dado precisar voltar: restauração a partir do arquivo da seção 3.
      Tempo estimado: `____`

---

## 6. Quem acompanha

- Quem executa: `____________________`
- Quem confere o resultado: `____________________`
- Quem decide voltar atrás, se for o caso: `____________________`

> As três podem ser a mesma pessoa em uma equipe pequena. O que não pode é ninguém.

---

## 7. Ordem do deploy

A ordem padrão deste projeto, para não haver dúvida no dia:

```
1. backup                     ← imediatamente antes
2. prisma migrate deploy      ← schema novo, com o código velho ainda no ar
3. sobe a versão nova         ← código novo, sobre schema já pronto
4. confere (seção 4)
```

**Entre 2 e 3 o código velho roda sobre o schema novo.** É por isso que a seção 2 existe.
