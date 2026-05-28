# Spec — Fluxo de saída + Histórico (HOB lista de internações)

Data: 2026-05-28 · Repo: `hob-leitos/index.html` · Rota: `#/internacoes/lista`
Auditoria triangular: `second_brain_md_package/08-Agent/health/auditorias/hob_fluxo_saida_internacoes_audit_20260528/` (veredicto REVISAR acatado integral; GO Maestro 2026-05-28).
Escopo: HOB apenas. Tabela `internacoes_hmsa` compartilhada — TODA query escopada `hospital=HOB`.

## Problema
Marcar saída na lista não grava data de saída (só o botão verde grava `data_alta`; óbito/evasão/transferência não gravam nada). Falta input de data com poka-yoke, tab Histórico, e Excel com data/motivo de saída.

## Decisões (pós-auditoria)
- **Bifurcação** do fluxo de saída (não botão único).
- **Campo de data**: grava `data_saida` (universal); `data_alta` sincronizado só quando motivo=alta. Exibição/export por coalesce `data_saida ?? data_alta`. Sem backfill agora (reservado v2, exige grep cross-hospital).
- **Histórico não-editável**; correção via drawer de edição (com auth).
- **Trilha mínima**: `updated_by`+`updated_at`+`motivo_edicao` na correção.
- **F5 race**: aceitar residual + carimbo "atualizado HH:MM"; medir antes de Realtime.

## Design

### 1. Fluxo de saída (bifurcação)
Linha da lista (`intRenderLista`, ~20355) tem hoje: verde `_intHandlerDarAlta` + log-out `intMarcarSaida`. Novo:
- **Botão verde "Dar alta" → `intDarAltaRapida(id)` (1-clique, sem modal)**: PATCH `{data_saida=hoje, data_alta=hoje, status_internacao='alta_medica', updated_at, updated_by}` → vagar leito (`_onda6MarcarLeitoVagado`) → `intFetchAll`+render → toast `✓ Alta de {nome LGPD} · leito {X} livre · [Desfazer]`. **Undo 5s**: guarda `{status, data_saida, data_alta}` anteriores; "Desfazer" faz PATCH de volta. Sem poka-yoke (data=hoje fixa, sempre ≥ admissão salvo admissão futura → validar mesmo assim).
- **Botão "Dar saída / outros motivos" → `intAbrirModalSaida(id)` (generaliza `_intHandlerDarAlta`)**: modal com:
  - **Motivo** select: Alta médica · Óbito · Evasão · Transferência.
  - **Data de saída**: input date, default hoje, `min=data_internacao`, `max=hoje` + validação no submit (data < admissão OU > hoje → bloqueia com mensagem). Defesa em 2 camadas (HTML + JS); client-side assumido (chave anon pública — risco aceito, escopo do app).
  - Confirmação + nome abreviado LGPD + aviso "leito {X} ficará livre".
  - Save → PATCH `{data_saida=data, status_internacao=motivo, data_alta=(motivo==='alta_medica'?data:null), updated_at, updated_by}` → vagar leito → render. Cobre alta retroativa (motivo=alta com data≠hoje).
- **DELETAR `intMarcarSaida`** (def ~20754 + call ~20357). **F1: grep `intMarcarSaida` e confirmar zero call sites remanescentes** antes de commit. Dashboard (`_intHandlerDarAlta` em 8882/15016) passa a chamar `intAbrirModalSaida` com motivo=alta pré-selecionado.
- Botão lixeira (`_intExcluirInline`) e edição (pencil) inalterados.

### 2. Lista (Internados)
- `intFiltrados`/`intRenderLista`: sub-tab Internados filtra `status_internacao==='ativa'`.
- Coluna "Data alta" → renomear "Data saída", lê coalesce `data_saida ?? data_alta`.

### 3. Tab Histórico (nova)
- Sub-tab nav: Fila | Internados | **Histórico** (espelha `tab-sub-int-*`, `switchView('internacoes','historico')`).
- `intRenderHistorico()`: filtra `status_internacao !== 'ativa'` de `INT.internacoes` (já carregado). Colunas: Status/Motivo (badge) · Data saída (coalesce) · Paciente (LGPD) · Atend. · Setor · Esp. · Permanência (dias = data_saida − data_internacao; "—" se falta data) · Ações (pencil → drawer correção).
- Controles próprios: filtro motivo + período + busca + botão Excel (`exportarHistoricoXLS`).
- **Não-editável inline.** Pencil abre `intAbrirEditar` (drawer) → ver §4.
- KPIs (média permanência, total saídas no período, % óbito): OPCIONAL v1.1, definir com NIR. NÃO incluir na v1 pra evitar overengineering.

### 4. Correção via drawer (com trilha)
- `intRenderForm`: hoje só mostra status quando `status==='ativa'` (~20970). Estender: quando `status!=='ativa'`, mostrar select Motivo + input Data saída (min=data_internacao, max=hoje) + campo obrigatório **Motivo da correção** (`motivo_edicao`).
- `intSalvar` edit path (~21040): aplicar regra do PATCH: `data_saida=data`, `status_internacao=motivo`, `data_alta=(motivo==='alta_medica'?data:null)`, `updated_at`, `updated_by`, e registrar `motivo_edicao` (campo/observação). Requer `requireEditAuth` (já existe).

### 5. Excel
- `exportarInternacoesXLS` (~20463): +colunas **Data saída** (coalesce), **Data alta**, **Motivo de saída** (label do status).
- `exportarHistoricoXLS`: mesmo conjunto, dataset = Histórico filtrado.

### 6. Premissa leito livre
- Ambos os caminhos vagam o leito + reload. Dashboard conta ocupado só `status==='ativa'` → leito vira Livre. Carimbo "atualizado HH:MM" no header da lista/dashboard.

## Out of scope (v1)
- Replicar pra HMSA/HJXXIII/HMAGR (follow-up pós-smoke).
- Backfill `data_saida=data_alta` cross-hospital (v2).
- Log de auditoria em tabela separada (mínimo via updated_by/updated_at/motivo_edicao).
- Supabase Realtime / polling (medir antes).
- KPIs do Histórico (v1.1 com NIR).
- Saída em lote (medir frequência antes).

## Verificação (critério de done)
- [ ] Grep `intMarcarSaida` = 0 call sites; `intDarAltaRapida`+`intAbrirModalSaida` cobrem alta/óbito/evasão/transferência.
- [ ] Saída por qualquer motivo grava `data_saida`; coluna "Data saída" e Excel mostram a data.
- [ ] Poka-yoke: data < admissão e data > hoje bloqueadas (modal e drawer).
- [ ] Undo da alta rápida reverte status+datas em 5s.
- [ ] Tab Histórico lista status≠ativa com permanência; não editável inline; pencil → drawer com auth + motivo_edicao.
- [ ] Saída libera leito no dashboard (smoke 1 aba). Smoke 2 abas documenta janela de desatualização.
- [ ] Excel Internados + Histórico com Data saída/Data alta/Motivo.
- [ ] Smoke local + QA visual (render real, não só código) antes de deploy.
- [ ] Tudo escopado `hospital=HOB`; zero toque em dados/lógica dos outros 3 hospitais.
