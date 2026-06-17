# Quiz do Dia — guia de instalação

Quiz diário: **10 perguntas** de múltipla escolha (4 opções), misturando **assuntos
em alta**, **lógica** e **conhecimentos gerais**. Você responde as 10, pode mudar de
ideia, e só vê o resultado ao **enviar**. Tem **login por nome + PIN** e **placar
compartilhado**, no mesmo estilo do Termo / Top 10.

```
quiz/
├─ index.html            → o jogo (não precisa editar)
├─ quiz-dados.js         → as perguntas (as respostas!). NÃO abra se for jogar 🤫
├─ config.js             → URL e chave do Supabase (já preenchido)
├─ manifest.webmanifest  → deixa instalar como app no celular
├─ icon-192.png / icon-512.png / apple-touch-icon.png → ícones do app
└─ LEIAME.md             → este guia
```

> **Pontuação:** começa em **1000**. Resposta **certa −0**, **em branco −40**,
> **errada −100**. A **💡 dica** elimina uma opção errada por **−20** (no máximo
> **2 por pergunta** — sempre sobram 2 opções). Nunca fica negativo (trava no 0).
> **Um quiz por dia** por pessoa — depois de enviar, não dá pra refazer.

> **Instalar no celular:** abra no navegador e use "Adicionar à tela de início".

---

## Parte 1 — Banco de dados (Supabase)

O placar e os logins do Quiz ficam numa tabela **própria** (`quiz_jogadores`),
separada do Termo e do Top 10. Abra o **SQL Editor** no seu projeto Supabase, cole
**todo** o bloco abaixo e clique em **Run**.

```sql
-- Jogadores do Quiz: nome + PIN + pontos acumulados + dias jogados + histórico
create table if not exists quiz_jogadores (
  nome       text primary key,
  pin        text not null,
  total      int  not null default 0,
  dias       int  not null default 0,
  jogadas    jsonb not null default '{}',
  created_at timestamptz default now()
);
alter table quiz_jogadores enable row level security;

-- Placar público: só nome e totais (nunca o PIN)
create or replace view placar_quiz as
  select nome, total, dias from quiz_jogadores;
grant select on placar_quiz to anon;

-- Entrar: cria o jogador (1ª vez) ou confere o PIN
create or replace function entrar_quiz(p_nome text, p_pin text)
returns table(ok boolean, total int, dias int, jogadas jsonb)
language plpgsql security definer set search_path = public as $$
declare j quiz_jogadores;
begin
  select * into j from quiz_jogadores where nome = p_nome;
  if not found then
    insert into quiz_jogadores(nome, pin) values (p_nome, p_pin) returning * into j;
  elsif j.pin <> p_pin then
    return query select false, 0, 0, '{}'::jsonb;
    return;
  end if;
  return query select true, j.total, j.dias, j.jogadas;
end; $$;
grant execute on function entrar_quiz(text, text) to anon;

-- Registrar o resultado de um dia (só uma vez por dia por pessoa)
create or replace function registrar_dia_quiz(p_nome text, p_pin text, p_data text, p_pontos int)
returns table(total int, dias int, ja_jogou boolean, jogadas jsonb)
language plpgsql security definer set search_path = public as $$
declare j quiz_jogadores;
begin
  select * into j from quiz_jogadores where nome = p_nome;
  if j.nome is null or j.pin <> p_pin then
    raise exception 'login invalido';
  end if;
  if j.jogadas ? p_data then
    return query select j.total, j.dias, true, j.jogadas;
    return;
  end if;
  update quiz_jogadores
     set total   = total + greatest(p_pontos, 0),
         dias    = dias + 1,
         jogadas = jogadas || jsonb_build_object(p_data, p_pontos)
   where nome = p_nome
   returning quiz_jogadores.total, quiz_jogadores.dias, quiz_jogadores.jogadas
   into j.total, j.dias, j.jogadas;
  return query select j.total, j.dias, false, j.jogadas;
end; $$;
grant execute on function registrar_dia_quiz(text, text, text, int) to anon;

-- Excluir jogador: SÓ o perfil "LEANDRO VIDA" (com o PIN dele) pode apagar alguém
create or replace function excluir_quiz(p_alvo text, p_admin_nome text, p_admin_pin text)
returns void
language plpgsql security definer set search_path = public as $$
begin
  if p_admin_nome <> 'LEANDRO VIDA' then
    raise exception 'sem permissao';
  end if;
  if not exists (select 1 from quiz_jogadores where nome = p_admin_nome and pin = p_admin_pin) then
    raise exception 'pin invalido';
  end if;
  delete from quiz_jogadores where nome = p_alvo;
end; $$;
grant execute on function excluir_quiz(text, text, text) to anon;
```

Deve aparecer **"Success. No rows returned"**. A URL e a chave já estão no `config.js`.

> Mesmo sem o SQL o jogo funciona — mas o placar fica só **neste aparelho**.

---

## Parte 2 — Publicar no GitHub (GitHub Pages)

Crie um repositório novo (ex.: `quiz`) e suba **todos** os arquivos da pasta `quiz/`.
Em **Settings → Pages**, aponte para a branch `main`. Em ~1 min fica no ar
(ex.: `https://SEU-USUARIO.github.io/quiz/`). Entre com **LEANDRO VIDA** + seu PIN
(único perfil que pode excluir jogadores).

---

## Adicionando o quiz de amanhã

Abra `quiz-dados.js` e adicione uma nova data (`AAAA-MM-DD`) com 10 perguntas. O jogo
mostra automaticamente os dias **até hoje**; quem perdeu um dia anterior pode voltar
e jogar (uma vez só por dia).

```js
window.QUIZ_PUZZLES = {
  "2026-06-16": { /* ... o de hoje ... */ },
  "2026-06-17": {
    titulo: "Quiz do dia",
    perguntas: [
      {
        categoria: "Conhecimentos gerais",
        pergunta: "...?",
        opcoes: ["...", "...", "...", "..."],
        correta: 0,                  // índice (0-3) da correta
        explicacao: "...",
        fonte: "...", fonteUrl: "https://..."
      }
      // ... 10 perguntas
    ]
  }
};
```

---

## Dúvidas comuns

**Quero zerar tudo.** No Supabase, **Table Editor → quiz_jogadores**, apague as linhas
(ou `delete from quiz_jogadores;`).

**Esqueci meu PIN.** No **Table Editor → quiz_jogadores**, edite a coluna `pin`.

**O placar não aparece.** Confirme que rodou o SQL da Parte 1 inteiro.
