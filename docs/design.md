# IBBA Daily News — Design Doc

**Data:** 2026-04-21
**Autor:** Ricardo + Claude

## Escopo

Duas peças ligadas ao mesmo projeto:

1. **Pipeline remoto** — migrar clipping diário do Cowork pro Claude Code (remote trigger), mantendo produto funcionando com PC desligado.
2. **Dashboard redesign** — elevar o `index.html` pra estética editorial premium, mobile decente, e utilidades reais.

Este doc é a referência pra implementação. Decisões tomadas em diálogo; pontos abertos sinalizados.

---

## Parte 1 — Arquitetura do pipeline

### Princípios
- Nada depende da máquina do Ricardo ligada às 7:10 AM, exceto a aprovação final.
- Tudo que falha silencioso escreve duplo (Supabase + outbox).
- Runbook self-contained no prompt do remote trigger.
- Scripts de dados (`search.py`, `format_email.py`, `supabase_insert.py`) ficam no repo `ricaribeirosilva-coder/ibba-daily-news` em `scripts/`.
- Obsidian = second brain local (leitura/edição humana); não é fonte de verdade executável.

### Componentes
1. **Schedule remoto** — cron Anthropic mon-fri 7:10 BRT dispara sessão Claude Code com runbook inline.
2. **Runbook embutido** no prompt do trigger. Regras de curadoria, Coverage Universe, Search Queries, formato do e-mail, tratamento de erros conhecidos — tudo inline.
3. **Execução remota** roda `search.py` → curadoria → `format_email.py` → Gmail draft **na conta pessoal do Ricardo** (`ricaribeirosilva@gmail.com`, não mais bot) → `supabase_insert.py` → execute_sql no Supabase → grava logs em `obsidian_outbox`.
4. **Aprovação** = Ricardo abre o draft no Gmail dele, revisa, clica Send. Sem gate via chat.
5. **Sync Obsidian** = comando local `/ibba-sync` drena `obsidian_outbox` pro vault quando o PC liga.

### Regras de curadoria (ênfase explícita do Ricardo)
- **Eixos primários:** data da notícia (dia da edição) e impacto no setor. Acima de todos os outros critérios.
- Formatação do e-mail é **crítica e bloqueante** — `format_email.py` valida antes de criar draft.
- Bot `ribeirosilvaclaudebot@gmail.com` **aposentado**. Credencial removida dos próximos deploys.

### Tratamento de dores conhecidas (Error Log Obsidian)
- **Erro #17 (URL GL não resolve):** URL não resolvida + matéria boa → **Tier B**. Nunca trava aguardando intervenção humana. Se nem Tier B justifica, descarta com log no outbox.
- **Erro #13 (URLs Google News):** `search.py` já resolve via HTTP HEAD redirects; runbook proíbe entregar URL do Google News em produção.
- **Erro #11 (curadoria fraca):** runbook carrega Curation Rules inline e exige checagem explícita antes do draft.
- **Erro #16 (draft invisível no bot):** resolvido por usar conta pessoal.
- **Erros #7, #10, #14 (Gmail compose frágil):** removidos — Chrome Gmail não é mais usado, só Gmail MCP.

### Pontos abertos (pipeline)
- **Schema do `obsidian_outbox`** — colunas mínimas: `id, created_at, kind` (error_log | licao | execucao | curation_note), `payload jsonb`, `applied_at timestamptz nullable`. A confirmar na implementação.
- **Rollback de falha parcial** — se Gmail draft OK mas Supabase falha: trigger registra `execution_status='partial'` numa tabela `executions`, `/ibba-sync` mostra estado ao Ricardo que decide retry ou abort. Se Supabase OK mas draft falha: idem, com dashboard marcando edição como "sent_pending". A detalhar no plano.

---

## Parte 2 — Dashboard redesign

### Direção
**Editorial premium** (FT/Substack-like). Menos SaaS-dashboard, mais jornal matinal sério.

### Stack
Vanilla HTML/CSS/JS num único arquivo. GitHub Pages. Zero build. (Constraint do Ricardo.)

### Tipografia
- **Títulos:** Fraunces (Google Fonts, variable, peso ótico).
- **UI/body:** Inter (substitui Manrope).
- **Dados:** IBM Plex Mono (mantido).

### Paleta
- **Background:** off-black neutro `#0c0c0e` com warm cream sutil no topo (papel noturno).
- **Accent laranja Itaú** preservado mas **restrito** a CTA principal e dot "database online". Sem usar em eyebrow/decor.
- Hierarquia de texto reduzida de 4 níveis pra 3 (title/body/meta).

### Layout
- **Masthead editorial** topo: "IBBA Daily Journal" em Fraunces grande + eyebrow + edição.
- **Hero MP4** 40-50vh — topografia animada abstrata. Dimmer gradient dark pro bottom. Título do top story sobreposto.
- **Meter bar** inline: `TOTAL 14 · BR 8 · GL 6 · EDITION 2026-04-21` — estilo régua de periódico.
- **Dois decks** (Brasil / Global) como antes, mas cards editoriais: menos chrome, serif nos headlines, mais ar.
- **Lead grid** removido — substituído pelo top story no hero.

### MP4
- **Fonte:** abstrato — topografia/linhas de contorno animadas.
- **Placeholder URL:** Coverr/Pexels free stock (swap na produção).
- **Fallback CSS:** gradient animado estático caso MP4 falhe ou `prefers-reduced-motion`.
- **Mobile (<768px):** desativado. Poster estático + gradient.
- **Performance:** `preload=metadata`, `playsinline muted autoplay loop`, pausa via Visibility API.

### Mobile
- Hero collapse: título maior, sem grade de métricas (linha resumo `14 · 8 BR · 6 GL`).
- Scroll-snap vertical nos cards pra leitura "uma por vez".
- PWA mínimo: manifest + SW pra add-to-home-screen.

### Utilidades incluídas
- **Mark-as-read** via localStorage (ícone sutil por card).
- **Keyboard:** J/K próximo/anterior card, / busca local (filtra headline/source/ticker), ← → edição ±1.
- **Print CSS:** `@media print` limpo, botão "imprimir / PDF".

### Utilidades **excluídas** (decisão Ricardo)
- Copy para WhatsApp.
- Filtro por ticker (pill filter).

### Segurança
- Adicionar CSP meta tag (Supabase origin + fonts + self).
- `rel="noopener noreferrer"` em todos os target=_blank.
- Confirmar RLS do Supabase: `anon` role **só SELECT** em `news_items`, escrita só via `service_role`.
- Sem secrets novos no HTML (Supabase anon é público-por-design).

---

## Plano de teste

1. Rewrite do `index.html`.
2. Abrir localmente via Chrome DevTools MCP, viewport desktop + mobile (375px).
3. Verificar: render, fetch Supabase real, keyboard shortcuts, mark-as-read, print preview, MP4 fallback.
4. Commit no branch `redesign-editorial` (não push ainda).
5. Ricardo revê e aprova ou pede ajuste.

## Próximos passos (pós-redesign)

1. Spec detalhada do pipeline remoto (schema outbox, runbook completo, scripts adaptados).
2. Plano de implementação via writing-plans skill.
3. Configuração do schedule remoto.
4. Dry-run do pipeline end-to-end.
