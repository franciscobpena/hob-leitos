---
type: spec
title: HOB Export PDF — paridade visual com bed-cards + ordenação TMP decrescente
created: 2026-05-12
status: aprovado_design
serves_okr: C (replicação hospitalar) + D (autoridade — caso de uso operacional)
related:
  - index.html:7726 (botão #btn-pdf)
  - index.html:8743-8794 (bed-card HTML real — fonte da paridade)
  - index.html:12121-12308 (implementação PDF atual)
  - second_brain_md_package/.claude/projects/.../memory/padrao-lgpd-abreviar-nome-paciente.md
  - second_brain_md_package/.claude/projects/.../memory/zpd-em-feature-nova-checar-defaults-obvios.md
---

# HOB Export PDF — Paridade Visual + Ordenação TMP

## Contexto

Feature parcialmente implementada em 2026-05-11 noite (commits locais não pushed, +192 linhas em `index.html`, tag `[HOB 2026-05-11]`). Maestro retomou em 2026-05-12 04:59 declarando que **não estava finalizada**: PDF atual mostra info incompleta, ordem alfabética arbitrária, e não tem paridade com cards visuais do dashboard.

Pedido literal do Maestro: "visão exata dos cards conforme aparecem em https://hob-leitos.vercel.app/#/gestao-leitos/dashboard, ordem dos TMPs maior pro menor".

## Objetivo

Exportar PDF que substitua quadro físico de huddle no HOB:
- Equipe imprime ou abre no celular
- Vê especialidades ordenadas por urgência (TMP médio decrescente)
- Identifica pior caso (maior dias permanência) em segundos dentro de cada página
- Reconhece os bed-cards do app sem precisar reaprender layout

## Decisões travadas no brainstorming (2026-05-12)

| Eixo | Decisão | Por quê |
|---|---|---|
| Ordenação | Dupla: páginas por TMP médio desc + cards por dias permanência desc | Ranking de urgência só funciona se consistente entre páginas E dentro delas |
| Paridade visual | Réplica de campos do bed-card HTML + nome paciente abreviado LGPD | PDF impresso vive fora do app; nome resolve "qual paciente é esse leito?" sem abrir sistema |
| Comanejo | Isolado em página final + threshold LP próprio (3d, lê de localStorage) | Retaguarda PS é semântica clínica distinta (boarding); threshold OMS 72h ≠ threshold dashboard 7d |
| Header de página | Completo: cor especialidade + "#N/Total · Esp · X/Y leitos · Z livres · TMP Wd · SP M (P%)" | Página existe por causa do ranking; mostrar posição #N torna explícito qual é a mais crítica |
| Abordagem técnica | jsPDF nativo + paleta replicada (não html2canvas) | Texto selecionável, arquivo leve (~250KB vs 4-8MB), determinístico, acessível |

## Limites assumidos

- **Animação `lp-pulse`** (borda pulsante) do dashboard **não é replicável** em PDF estático. Substituto: borda vermelha mais grossa (1mm) + badge `LP` no canto sup direito do card.
- **Hover overlays** (botões de alta, edit, cancel) **não vão pro PDF** — são interação, não estado.
- **Paleta CSS↔PDF** fica sincronizada manualmente. Comentário no helper aponta `index.html:444-448` (linha do `--f0..--f4`). Se paleta CSS mudar, atualizar `_pdfPaleta` é manual.

## Componentes

Todos em `index.html`, mesma região 12121-12308. Sem novo arquivo.

### 1. `_pdfPaleta` (novo objeto módulo)

```js
const _pdfPaleta = {
  // Replicado de index.html:444-448 (--f0..--f4 CSS custom properties).
  // Sync manual: se mudar CSS, atualizar aqui.
  f0: { fill: [232, 252, 234], border: [40, 167, 69], text: [20, 80, 35] },   // ≤1d verde
  f1: { fill: [240, 252, 232], border: [132, 204, 22], text: [40, 90, 25] },  // ≤2d verde-amarelo
  f2: { fill: [253, 246, 209], border: [251, 191, 36], text: [120, 90, 15] }, // ≤4d amarelo
  f3: { fill: [253, 224, 192], border: [245, 158, 11], text: [140, 80, 20] }, // ≤7d laranja
  f4: { fill: [254, 226, 226], border: [239, 68, 68], text: [140, 30, 30] },  // >7d vermelho
};
```

### 2. `_tmpMedio(pacientes)` (novo)

```js
function _tmpMedio(pacientes) {
  if (!pacientes || !pacientes.length) return 0;
  const soma = pacientes.reduce((s, p) => s + (calcDiasHoje(p) || 0), 0);
  return soma / pacientes.length;
}
```

### 3. `_ordenarEspecialidadesPorTMP(map)` (novo)

```js
function _ordenarEspecialidadesPorTMP(especsMap) {
  return Object.entries(especsMap)
    .map(([esp, pacs]) => ({ esp, pacs, tmp: _tmpMedio(pacs) }))
    .sort((a, b) => b.tmp - a.tmp);
}
```

### 4. `_ordenarCardsPorDias(pacientes)` (novo)

```js
function _ordenarCardsPorDias(pacientes) {
  return [...pacientes].sort((a, b) => {
    const dB = calcDiasHoje(b) || 0;
    const dA = calcDiasHoje(a) || 0;
    if (dB !== dA) return dB - dA;
    // Tie-break: leito asc (estável, ordenação natural numérica quando possível)
    const lA = parseInt(a.numLeito || a.leito || 0, 10);
    const lB = parseInt(b.numLeito || b.leito || 0, 10);
    return lA - lB;
  });
}
```

### 5. `_pdfDrawWardHeader(doc, esp, rank, total, pacientes, capacidade, color)` (novo)

Substitui `drawHeader` interno de `_pdfPageEspecialidade`. Renderiza:
- Linha 1: `#{rank}/{total}` em pequeno cinza + `{esp}` em bold tamanho 14 com cor da especialidade
- Linha 2: stats `{ocupados}/{cap} leitos · {livres} livres · TMP {tmp.toFixed(1)}d · SP {spCnt} ({spPct}%)`

Cor da especialidade: reusar lógica `_corEspecialidade(esp)` se existir; senão fallback teal `[20, 184, 166]`.

### 6. `_pdfDrawBedCard(doc, x, y, w, h, p, opts)` refactor

`opts = { isComanejo: bool, lpThreshold: number }`.

Layout interno (44×28mm):
```
y+5    [Leito {numLeito}]                      [BADGE]
y+10   {esp.substring(0,18)}
y+18   [{dias}] dias                {nome abreviado}
y+24   {paIndicator}{prazoText}     {tagBarreiras}
```

- `BADGE` no canto sup direito: prioridade `LP > OV > AD/AP > SP`. Mostra só o de maior prioridade + texto mini `+N` em cinza se houver outras.
- `dias` grande à esq usando `_pdfPaleta[tier].text` como cor.
- `nome` via `abreviarNomePaciente(p.nomePaciente || p.nome_paciente)` (memória LGPD obrigatória).
- `prazoText`: se SP → "SP" em vermelho bold; senão "até DD/MM".
- `paIndicator`: bullet azul `•` se `getAnn(p)` retorna plano de ação.
- `tagBarreiras`: usar `_barreirasResumo(getAnn(p))` se disponível; renderiza `Barr:N` em cinza claro.
- Borda: usar `_pdfPaleta[tier].border` com largura 0.4mm; se LP, aumentar pra 1.0mm.

Cálculo tier:
- F0: dias ≤ 1
- F1: dias ≤ 2
- F2: dias ≤ 4 (≤ 3 em Comanejo)
- F3: dias ≤ 7 (≤ 5 em Comanejo)
- F4: dias > 7 (> 5 em Comanejo, equivale a `getColorClassRetag` da linha 14634)

### 7. `_pdfPageComanejo(doc, pacientes)` (novo)

Página própria, separa de `_pdfPageEspecialidade` porque:
- Threshold LP é 3d (não 7d)
- Header diz "Comanejo · Retaguarda PS · LP >3d" (não traz #ranking, fica fora do ranking principal)
- `getColorClassRetag` (linha 14634) tem boundaries diferentes que devem ser respeitados

```js
function _pdfPageComanejo(doc, pacientes) {
  const lpThr = (() => {
    const stored = parseInt(localStorage.getItem(`retaguarda_lp_threshold_${HOSPITAL_ID}`), 10);
    return (stored >= 1 && stored <= 7) ? stored : 3;
  })();
  // ... renderiza header próprio + grid de cards com _pdfDrawBedCard(opts={isComanejo:true, lpThreshold:lpThr})
}
```

### 8. `_pdfDrawCapa(doc, dash, comanejo)` (novo — extract da capa atual)

Extrai a renderização da capa (atualmente inline em `exportarPDFLeitos` linhas 12156-12178) pra função própria. Mantém conteúdo idêntico ao atual (título, subtítulo, data, resumo geral com totais + LP + SP). Motivo: orquestrador fica enxuto + testável.

### 9. `_corEspecialidade(esp)` (novo OU verificar se já existe)

Risco #3: header da página usa cor da especialidade. Antes da implementação, **grep `_corEspecialidade|corEsp|colorByEsp` no `index.html`** pra confirmar se já existe. Se existir: reusar. Se não existir, criar com mapping mínimo:

```js
const _PDF_COR_ESPEC = {
  'Neurovascular':   [88, 28, 135],   // roxo
  'Cirurgia':        [37, 99, 235],   // azul
  'Ortopedia':       [13, 148, 136],  // teal
  'Clínica Médica':  [217, 119, 6],   // âmbar
  'Comanejo':        [100, 116, 139], // slate
};
function _corEspecialidade(esp) {
  return _PDF_COR_ESPEC[esp] || [20, 184, 166]; // fallback teal
}
```

Mapping inicial é palpite — Maestro confirma na 1ª pass do smoke se as cores batem com dashboard. Se não, ajustar `_PDF_COR_ESPEC`.

### 10. `exportarPDFLeitos()` refactor orquestrador

```js
async function exportarPDFLeitos() {
  let pdfLib;
  try { pdfLib = await carregarJsPDF(); }
  catch (e) { alert('Erro carregando jsPDF: ' + e.message); return; }
  const { jsPDF } = pdfLib;
  const doc = new jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });

  // Fetch Comanejo (não vem em allPatients)
  let comanejo = [];
  try {
    const resp = await fetch(`${SB_URL}/rest/v1/internacoes_hmsa?hospital=eq.${HOSPITAL_ID}&setor=eq.Comanejo&status_internacao=eq.ativa&select=*&order=leito.asc`, { headers: SB_H });
    if (resp.ok) comanejo = await resp.json();
  } catch (e) { console.warn('[PDF] Comanejo fetch fail:', e); }

  const dash = allPatients || [];

  // Capa (mantém estrutura atual)
  _pdfDrawCapa(doc, dash, comanejo);

  // Agrupa especialidades dashboard, exclui Comanejo (já filtrado), ordena por TMP médio desc
  const especsMap = {};
  dash.forEach(p => {
    const k = (p.especialidade || 'Sem especialidade').toString().trim();
    if (!especsMap[k]) especsMap[k] = [];
    especsMap[k].push(p);
  });
  const ranking = _ordenarEspecialidadesPorTMP(especsMap);
  const totalEspecs = ranking.length;

  ranking.forEach(({ esp, pacs, tmp }, idx) => {
    doc.addPage();
    const pacsOrd = _ordenarCardsPorDias(pacs);
    _pdfPageEspecialidade(doc, esp, pacsOrd, idx + 1, totalEspecs, tmp);
  });

  // Comanejo isolado no fim
  if (comanejo.length) {
    doc.addPage();
    const comanejoOrd = _ordenarCardsPorDias(comanejo);
    _pdfPageComanejo(doc, comanejoOrd);
  }

  const hoje = new Date().toLocaleDateString('pt-BR');
  const safeDate = hoje.replace(/\//g, '-');
  doc.save(`HOB_leitos_${safeDate}.pdf`);
}
```

## Estrutura de saída final

```
P1     Capa
       · Hospital Metropolitano Odilon Behrens
       · Gestão de Leitos — Kanban Altas Oportunas
       · Data: DD/MM/AAAA
       · Total ocupados dashboard: {N}
       · Total Comanejo: {M}
       · Longa permanência (>7d): {K}
       · Sem previsão de alta: {S}

P2     #1/4 · Neurovascular · 23/26 leitos · 3 livres · TMP 8.2d · SP 6 (26%)
       [grid 4 cols, cards dias desc]

P3     #2/4 · Cirurgia · 32/36 leitos · 4 livres · TMP 6.8d · SP 4 (12%)
       [grid 4 cols, cards dias desc]

...

P5     #4/4 · Clínica Médica · 48/60 leitos · 12 livres · TMP 4.1d · SP 11 (23%)
       [grid 4 cols, cards dias desc]

P6     Comanejo · Retaguarda PS · LP >3d · 10/13 leitos · 1 bloqueado · TMP 4.5d
       [grid 4 cols, cards dias desc]
```

## Error handling

| Situação | Comportamento |
|---|---|
| jsPDF falha ao carregar | `alert('Erro carregando jsPDF: ' + e.message)` (já existe) |
| Fetch Comanejo falha | `console.warn` + segue sem página Comanejo |
| `allPatients` vazio | Exporta só capa + Comanejo se houver |
| Especialidade null/empty | Agrupa em "Sem especialidade", entra no ranking com TMP próprio |
| `localStorage` LP_THRESHOLD inválido | Default 3d (pattern já usado em `renderRetaguardaCards` linha 14560) |
| `calcDiasHoje` retorna NaN/undefined | Trata como 0 (não quebra ordenação) |
| Cards estouram página A4 | Paginação automática "(cont.)" como já existe |
| Especialidade sem nome em ranking | Mostra como "Sem especialidade" no header |

## Teste de aceitação (smoke)

Maestro executa em ordem:

1. Sobe http-server local em `C:/Users/Francisco/Desktop/claude-cowork/hob-leitos/` na porta 8765 (`python -m http.server 8765`)
2. Abre `http://localhost:8765` no browser, navega pra `/#/gestao-leitos/dashboard`
3. Aguarda load dos 134 pacientes HOB (banner sync conclui)
4. Clica botão `PDF` na barra `#export-bar` (toolbar global, à esquerda do "Sincronizar")
5. Arquivo `HOB_leitos_DD-MM-AAAA.pdf` baixa
6. Abre PDF e verifica:
   - [ ] Capa com totais corretos (bater com counters do dashboard)
   - [ ] N páginas (uma por especialidade do dashboard) em ordem TMP médio desc — conferir que TMP médio do header da página 1 ≥ página 2 ≥ ... ≥ página N
   - [ ] Dentro de cada página, cards em ordem dias permanência desc — conferir que primeiro card tem mais dias que o último
   - [ ] Header de cada página tem `#N/Total · Esp · X/Y leitos · TMP Wd · SP M (P%)`
   - [ ] Bed-cards mostram: leito grande, especialidade, dias grande à esq, nome abreviado, prazo ou SP, badge LP/OV/AD/AP se aplicável, bullet PA se tem plano, tag barreiras se tem
   - [ ] Cores F0-F4 batem visualmente com dashboard (verde→amarelo→laranja→vermelho por dias)
   - [ ] Última página é Comanejo, header próprio "LP >3d"
   - [ ] Cards Comanejo usam threshold 3d (vermelho/F4 entra mais cedo)

## Critério de done

- 8/8 checkboxes do smoke passam
- Maestro confirma "ok pra commitar"
- Commit + push pra `main` (deploy auto via Git integration HOB conforme memória `vercel-deploy-hob-tem-git-integration`)
- Smoke prod adicional em `https://hob-leitos.vercel.app/#/gestao-leitos/dashboard`

## Riscos auto-declarados

1. **Paleta CSS drift** — se alguém mudar `--f0..--f4` no CSS sem atualizar `_pdfPaleta`, PDF e dashboard divergem visualmente. Mitigação: comentário explícito + grep teste mensal.
2. **TMP médio é frágil com N pequeno** — especialidade com 1 paciente de 30d vira TMP=30 e disputa #1 com sentido operacional baixo. Decisão: aceitar drift (raro no HOB, 134 pacientes em 4-5 especialidades grandes). Memória `agregacao-estatistica-exige-n-minimo` se aplica se virar problema — então adicionar `if (n<3) tmp_visible_but_flagged`.
3. **Cor da especialidade no header** depende de `_corEspecialidade()` existir. Componente 9 cobre os 2 caminhos (reusar se existe, criar com mapping mínimo + ajuste pós-smoke se não existe).
4. **Comanejo page** assume schema `setor='Comanejo'` da carga 11/mai. Se Maestro recarregar com schema diferente, quebra. Mitigação: fetch usa `setor=eq.Comanejo`, então mudança força ajuste consciente.

## Não escopo desta entrega

- Filtros (período, andar, especialidade específica): ficam pra v2 se houver demanda
- Configurações de PDF (orientação, formato, paleta custom): hardcoded por agora
- Export em batch (múltiplos hospitais): só HOB
- Versão "compacta" com 2 cards por linha pra impressão A5: não pedido
- Histórico de PDFs gerados: download direto, sem persistência
