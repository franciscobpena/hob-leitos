# HOB Export PDF — Paridade Visual + Ordenação TMP · Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir implementação parcial de 11/mai (não commitada, +192 linhas) por export PDF com paridade visual aos bed-cards do dashboard + ordenação dupla por TMP (especialidades desc + cards desc dentro).

**Architecture:** jsPDF nativo (cdnjs lazy load) com paleta RGB replicada do CSS `--f0..--f4`. 10 componentes em `index.html` (single-file SPA). Sem novo arquivo. Sem framework de teste — "TDD adaptado" = smoke step em browser console ou visual check após cada task.

**Tech Stack:** vanilla JS + jsPDF 2.5.1 (cdnjs imutável) + Supabase REST (já em uso) + paleta hardcoded espelhando CSS.

**Spec base:** `docs/superpowers/specs/2026-05-12-hob-pdf-paridade-tmp-design.md` (commit 99f058e)

**Deploy:** `git push main` → auto via Git integration Vercel HOB (memória `vercel-deploy-hob-tem-git-integration`).

---

## File Structure

Tudo em `index.html`. Regiões afetadas:

| Região | Linha aproximada | Mudança |
|---|---|---|
| `#export-bar` | ~7723 | Add `<button id="btn-pdf">` |
| `WARD_COLORS` | 8652 | Add entradas HOB (Neurovascular, Cirurgia, Sem especialidade) |
| Helpers PDF (novo bloco) | ~12120 | `_pdfPaleta`, `_hexToRgb`, `_pdfCorEspecialidade`, `_tmpMedio`, `_ordenarEspecialidadesPorTMP`, `_ordenarCardsPorDias` |
| Funções PDF existentes | 12121-12308 | Refactor: `_pdfDrawCapa` (extract), `_pdfDrawWardHeader` (novo), `_pdfDrawBedCard` (refactor), `_pdfPageEspecialidade` (refactor), `_pdfPageComanejo` (novo), `exportarPDFLeitos` (refactor orquestrador) |

---

## Task 0: Baseline cleanup — reset working tree

**Files:**
- Reset: `index.html` (descarta +192 linhas locais não commitadas de 11/mai)

- [ ] **Step 1: Confirmar estado atual**

Run:
```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git status --short
```
Expected: ` M index.html` + `?? .gitignore` + (commit 99f058e da spec já feito)

- [ ] **Step 2: Reset implementação parcial 11/mai**

Run:
```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git checkout -- index.html && git status --short
```
Expected: working tree limpo pra `index.html`, só `?? .gitignore` permanece. Linha count volta ao baseline.

- [ ] **Step 3: Confirmar baseline ausência do código antigo**

Run grep:
```
Grep pattern="exportarPDFLeitos|carregarJsPDF|_pdfDrawBedCard" path="index.html"
```
Expected: 0 matches (impl 11/mai removida).

- [ ] **Step 4: Sem commit nesta task** — reset não deixa rastro, próxima task constrói do zero.

---

## Task 1: Botão PDF na toolbar `#export-bar`

**Files:**
- Modify: `index.html:~7723` (após `btn-ppt`)

- [ ] **Step 1: Add botão entre PPT e divider**

Edit `index.html` na seção `#export-bar`. Inserir após o `</button>` do `btn-ppt` e antes do `<span style="width:1px..."`:

```html
  <button id="btn-pdf" onclick="exportarPDFLeitos()" style="background:var(--card);color:var(--teal);border:1px solid var(--teal);margin-left:4px" title="Exportar PDF de leitos (dashboard + Comanejo)">
    <i aria-hidden="true" data-lucide="file-text"></i> PDF
  </button>
```

- [ ] **Step 2: Smoke visual (botão aparece)**

Sobe http-server local:
```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && python -m http.server 8765
```

Abre `http://localhost:8765` no browser. Aguarda load. Confere visualmente:
- Botão "PDF" aparece em `#export-bar` entre "PPT" e "Sincronizar"
- Ícone Lucide `file-text` renderiza (precisa ter `lucide.createIcons()` rodar — já roda no init)
- Tooltip "Exportar PDF de leitos..." aparece no hover

**Expected**: botão visível. Click ainda gera erro `exportarPDFLeitos is not defined` no console — esperado (próximas tasks criam a função).

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T1] botao #btn-pdf no #export-bar"
```

---

## Task 2: Lazy load `carregarJsPDF()`

**Files:**
- Modify: `index.html:~12120` (insert ANTES de qualquer outro código PDF)

- [ ] **Step 1: Localizar ponto de inserção**

Run Grep:
```
pattern="/\* =+\s*$|Fase 60.*EXCEL EXPORT PROJETOS KAIZEN" path="index.html" -n=true
```

Achar a linha do block `/* === FASE 60 — EXCEL EXPORT PROJETOS KAIZEN === */` (era ~12422 no estado pré-Task 0, mas vai mudar conforme tasks). Inserir antes desse bloco.

- [ ] **Step 2: Add função `carregarJsPDF`**

Insert no ponto definido:

```js
/* ============================================================
   HOB 2026-05-12 — PDF EXPORT GESTÃO DE LEITOS
   Spec: docs/superpowers/specs/2026-05-12-hob-pdf-paridade-tmp-design.md
   ============================================================ */

/** Lazy load jsPDF (cdnjs imutável, sem SRI necessário).
 *  Cache global em window._jsPdfLoading pra evitar recarga. */
function carregarJsPDF() {
  if (window.jspdf?.jsPDF) return Promise.resolve(window.jspdf);
  if (window._jsPdfLoading) return window._jsPdfLoading;
  window._jsPdfLoading = new Promise((resolve, reject) => {
    const s = document.createElement('script');
    s.src = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js';
    s.crossOrigin = 'anonymous';
    s.onload = () => resolve(window.jspdf);
    s.onerror = () => reject(new Error('Falha carregando jsPDF'));
    document.head.appendChild(s);
  });
  return window._jsPdfLoading;
}
```

- [ ] **Step 3: Smoke console**

Browser DevTools console:
```js
await carregarJsPDF()
```
**Expected**: retorna `{jsPDF: ƒ}` (sem erro).

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T2] carregarJsPDF lazy load"
```

---

## Task 3: Helpers paleta — `_pdfPaleta` + `_hexToRgb` + `_pdfCorEspecialidade`

**Files:**
- Modify: `index.html:8652` (add entradas HOB em `WARD_COLORS`)
- Modify: `index.html:~12140` (helpers novos após `carregarJsPDF`)

- [ ] **Step 1: Atualizar `WARD_COLORS` com nomes HOB**

Edit linha 8652-8658. Adicionar entradas:

```js
const WARD_COLORS = {
  'Ortopedia':            '#60A5FA',
  'Clinica Medica':       '#34D399',
  'Clínica Médica':       '#34D399',
  'Neuro\Tórax\Vascular': '#A78BFA',
  'CirGeral\Bucomaxilo':  '#FB923C',
  // [HOB 2026-05-12] Entradas HOB
  'Neurovascular':        '#A78BFA',
  'Cirurgia':             '#FB923C',
  'Comanejo':             '#64748B',
  'Sem especialidade':    '#94A3B8',
};
```

- [ ] **Step 2: Add `_pdfPaleta` + `_hexToRgb` + `_pdfCorEspecialidade`**

Insert depois de `carregarJsPDF()`:

```js
/** Paleta F0-F4 replicada de index.html:444-448 (--f0..--f4 CSS).
 *  Sync manual: se CSS mudar, atualizar aqui. */
const _pdfPaleta = {
  f0: { fill: [232, 252, 234], border: [40, 167, 69],   text: [20, 80, 35] },   // ≤1d
  f1: { fill: [240, 252, 232], border: [132, 204, 22],  text: [40, 90, 25] },   // ≤2d
  f2: { fill: [253, 246, 209], border: [251, 191, 36],  text: [120, 90, 15] },  // ≤4d
  f3: { fill: [253, 224, 192], border: [245, 158, 11],  text: [140, 80, 20] },  // ≤7d (≤5d Comanejo)
  f4: { fill: [254, 226, 226], border: [239, 68, 68],   text: [140, 30, 30] },  // >7d (>5d Comanejo)
};

/** Helper: '#RRGGBB' → [r,g,b]. Fallback teal se inválido. */
function _hexToRgb(hex) {
  if (!hex || typeof hex !== 'string') return [20, 184, 166];
  const m = hex.replace('#','').match(/^([0-9a-f]{2})([0-9a-f]{2})([0-9a-f]{2})$/i);
  if (!m) return [20, 184, 166];
  return [parseInt(m[1], 16), parseInt(m[2], 16), parseInt(m[3], 16)];
}

/** Cor da especialidade no PDF, lookup em WARD_COLORS. Fallback teal. */
function _pdfCorEspecialidade(esp) {
  return _hexToRgb(WARD_COLORS[esp]);
}
```

- [ ] **Step 3: Smoke console**

Browser DevTools console:
```js
_pdfCorEspecialidade('Neurovascular')   // [167, 139, 250]
_pdfCorEspecialidade('Cirurgia')        // [251, 146, 60]
_pdfCorEspecialidade('Ortopedia')       // [96, 165, 250]
_pdfCorEspecialidade('Inexistente')     // [20, 184, 166]
_pdfPaleta.f4.border                    // [239, 68, 68]
```

**Expected**: cada um retorna o RGB triplet correto.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T3] helpers paleta _pdfPaleta + _hexToRgb + _pdfCorEspecialidade + entradas HOB em WARD_COLORS"
```

---

## Task 4: Helpers ordenação — `_tmpMedio` + `_ordenarEspecialidadesPorTMP` + `_ordenarCardsPorDias`

**Files:**
- Modify: `index.html:~12170` (após helpers de paleta)

- [ ] **Step 1: Add 3 helpers**

Insert:

```js
/** TMP médio = média de calcDiasHoje. 0 se vazio. */
function _tmpMedio(pacientes) {
  if (!pacientes || !pacientes.length) return 0;
  const soma = pacientes.reduce((s, p) => s + (calcDiasHoje(p) || 0), 0);
  return soma / pacientes.length;
}

/** Ordena Object.entries(especsMap) por TMP médio desc.
 *  Retorna [{ esp, pacs, tmp }]. */
function _ordenarEspecialidadesPorTMP(especsMap) {
  return Object.entries(especsMap)
    .map(([esp, pacs]) => ({ esp, pacs, tmp: _tmpMedio(pacs) }))
    .sort((a, b) => b.tmp - a.tmp);
}

/** Ordena cards por dias permanência desc. Tie-break: leito asc (numérico). */
function _ordenarCardsPorDias(pacientes) {
  return [...pacientes].sort((a, b) => {
    const dA = calcDiasHoje(a) || 0;
    const dB = calcDiasHoje(b) || 0;
    if (dB !== dA) return dB - dA;
    const lA = parseInt(a.numLeito || a.leito || 0, 10);
    const lB = parseInt(b.numLeito || b.leito || 0, 10);
    return lA - lB;
  });
}
```

- [ ] **Step 2: Smoke console com dados reais**

Browser DevTools console (com app carregado, `allPatients` populado):
```js
_tmpMedio(allPatients)
// Expected: número razoável (TMP HOB ~5-8d)

const map = {};
allPatients.forEach(p => {
  const k = (p.especialidade || 'Sem').trim();
  (map[k] = map[k] || []).push(p);
});
_ordenarEspecialidadesPorTMP(map).map(r => `${r.esp}: TMP ${r.tmp.toFixed(1)}d (${r.pacs.length} pacs)`)
// Expected: array ordenado decrescente por TMP

_ordenarCardsPorDias(allPatients.slice(0, 10)).map(p => `${p.numLeito} → ${calcDiasHoje(p)}d`)
// Expected: dias decrescentes
```

**Expected**: todos retornam dados consistentes. TMPs decrescentes confirmados visualmente.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T4] helpers ordenacao _tmpMedio + _ordenarEspecialidadesPorTMP + _ordenarCardsPorDias"
```

---

## Task 5: `_pdfDrawCapa(doc, dash, comanejo)` — extract da capa

**Files:**
- Modify: `index.html:~12200` (add função capa)

- [ ] **Step 1: Add função capa**

Insert:

```js
/** Desenha a capa do PDF: hospital + data + resumo geral. */
function _pdfDrawCapa(doc, dash, comanejo) {
  const hoje = new Date().toLocaleDateString('pt-BR');

  doc.setFont('helvetica', 'bold');
  doc.setFontSize(18);
  doc.text('Hospital Metropolitano Odilon Behrens', 105, 40, { align: 'center' });
  doc.setFontSize(13);
  doc.text('Gestão de Leitos — Kanban Altas Oportunas', 105, 50, { align: 'center' });
  doc.setFont('helvetica', 'normal');
  doc.setFontSize(11);
  doc.text(`Data de emissão: ${hoje}`, 105, 60, { align: 'center' });

  const lpThr = (typeof getT === 'function') ? getT() : 7;
  const lpCount = dash.filter(p => (typeof calcDiasHoje === 'function' ? calcDiasHoje(p) : 0) > lpThr).length;
  const spCount = dash.filter(p => typeof isSP === 'function' && isSP(p)).length;

  doc.setFontSize(11);
  doc.setFont('helvetica', 'bold');
  doc.text('Resumo geral', 105, 80, { align: 'center' });
  doc.setFont('helvetica', 'normal');
  doc.setFontSize(10);
  doc.text(`Total ocupados (dashboard): ${dash.length}`, 105, 88, { align: 'center' });
  doc.text(`Total Comanejo (retaguarda): ${comanejo.length}`, 105, 94, { align: 'center' });
  doc.text(`Longa permanência (>${lpThr} dias): ${lpCount}`, 105, 100, { align: 'center' });
  doc.text(`Sem previsão de alta: ${spCount}`, 105, 106, { align: 'center' });
}
```

- [ ] **Step 2: Smoke isolado (não há orquestrador ainda)**

Console:
```js
const pdfLib = await carregarJsPDF();
const doc = new pdfLib.jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
_pdfDrawCapa(doc, allPatients || [], []);
doc.save('test-capa.pdf');
```

**Expected**: download `test-capa.pdf` com 1 página, capa renderizada. Abre e confere visualmente: título, data, totais corretos.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T5] _pdfDrawCapa extract"
```

---

## Task 6: `_pdfDrawBedCard(doc, x, y, w, h, p, opts)` — refactor paridade visual

**Files:**
- Modify: `index.html:~12240` (add função; substitui versão antiga que ainda não existe pós-Task 0)

- [ ] **Step 1: Add função bed-card com paridade**

Insert:

```js
/** Desenha 1 bed-card no PDF (44×28mm). Paridade visual com bed-card HTML.
 *  opts = { isComanejo: bool, lpThreshold: number } */
function _pdfDrawBedCard(doc, x, y, w, h, p, opts) {
  const isComanejo = !!(opts && opts.isComanejo);
  const lpThr = (opts && opts.lpThreshold) || (typeof getT === 'function' ? getT() : 7);
  const dias = (typeof calcDiasHoje === 'function' ? calcDiasHoje(p) : 0) || 0;

  // Tier por dias (Comanejo usa thresholds diferentes)
  let tierKey;
  if (isComanejo) {
    tierKey = dias <= 1 ? 'f0' : dias <= 2 ? 'f1' : dias <= 3 ? 'f2' : dias <= 5 ? 'f3' : 'f4';
  } else {
    tierKey = dias <= 1 ? 'f0' : dias <= 2 ? 'f1' : dias <= 4 ? 'f2' : dias <= 7 ? 'f3' : 'f4';
  }
  const tier = _pdfPaleta[tierKey];
  const isLP = dias > lpThr;

  // Borda + fundo
  doc.setFillColor(tier.fill[0], tier.fill[1], tier.fill[2]);
  doc.setDrawColor(tier.border[0], tier.border[1], tier.border[2]);
  doc.setLineWidth(isLP ? 1.0 : 0.4);
  doc.roundedRect(x, y, w, h, 1.2, 1.2, 'FD');
  doc.setLineWidth(0.2);

  // Campos
  const leitoN = p.numLeito || p.leito || '—';
  const rawNome = p.nomePaciente || p.nome_paciente || '';
  const nome = (typeof abreviarNomePaciente === 'function')
    ? (abreviarNomePaciente(rawNome) || '—')
    : (rawNome.substring(0, 16) || '—');
  const esp = p.especialidade || '—';
  const prevText = p.previsaoAlta || p.previsao_kanban || '';
  const isSP_ = (typeof isSP === 'function') ? isSP(p) : (!prevText || prevText === 'SP');
  const statusAlta = p.status_alta_pre_marcada;
  const hasOv = !!p._hasLeitoOverride;
  const ann = (typeof getAnn === 'function') ? getAnn(p) : null;
  const temPA = !!ann;
  const barr = (typeof _barreirasResumo === 'function' && ann) ? _barreirasResumo(ann) : null;

  // Linha 1: Leito grande + badge canto sup direito
  doc.setTextColor(20, 20, 30);
  doc.setFontSize(10);
  doc.setFont('helvetica', 'bold');
  doc.text(`Leito ${leitoN}`, x + 2, y + 5);

  // Badge canto sup direito — prioridade LP > OV > AD/AP > SP
  let badgeText = null, badgeColor = null;
  if (isLP)                  { badgeText = 'LP';        badgeColor = [239, 68, 68]; }
  else if (hasOv)            { badgeText = 'OV';        badgeColor = [167, 139, 250]; }
  else if (statusAlta)       { badgeText = statusAlta;  badgeColor = [40, 130, 60]; }
  else if (isSP_)            { badgeText = 'SP';        badgeColor = [239, 68, 68]; }

  if (badgeText) {
    doc.setFillColor(badgeColor[0], badgeColor[1], badgeColor[2]);
    doc.setTextColor(255, 255, 255);
    doc.setFontSize(6);
    doc.setFont('helvetica', 'bold');
    const bw = badgeText.length <= 2 ? 8 : 10;
    doc.roundedRect(x + w - bw - 1, y + 1, bw, 4.5, 0.5, 0.5, 'F');
    doc.text(badgeText, x + w - bw/2 - 1, y + 4, { align: 'center' });
  }

  // Linha 2: especialidade (Comanejo destaca; outras cor cinza)
  doc.setFont('helvetica', 'normal');
  doc.setFontSize(7);
  doc.setTextColor(100, 100, 110);
  doc.text(esp.substring(0, 22), x + 2, y + 10);

  // Linha 3: dias GRANDE à esq + nome à dir
  doc.setFontSize(13);
  doc.setFont('helvetica', 'bold');
  doc.setTextColor(tier.text[0], tier.text[1], tier.text[2]);
  doc.text(String(dias), x + 2, y + 18);

  doc.setFontSize(6);
  doc.setFont('helvetica', 'normal');
  doc.setTextColor(120, 120, 130);
  doc.text('dias', x + 2, y + 21);

  doc.setFontSize(8);
  doc.setFont('helvetica', 'normal');
  doc.setTextColor(20, 20, 30);
  doc.text(nome.substring(0, 18), x + 11, y + 18);

  // Linha 4 (rodapé): bullet PA + prazo + tag barreiras
  doc.setFontSize(7);

  let footX = x + 2;
  if (temPA) {
    doc.setTextColor(59, 130, 246);
    doc.setFont('helvetica', 'bold');
    doc.text('•', footX, y + 25);
    footX += 2;
    doc.setFont('helvetica', 'normal');
  }

  if (isSP_) {
    doc.setTextColor(239, 68, 68);
    doc.setFont('helvetica', 'bold');
    doc.text('SP', footX, y + 25);
  } else if (prevText) {
    doc.setTextColor(40, 130, 60);
    const m = prevText.match(/^(\d{4})-(\d{2})-(\d{2})/);
    const display = m ? `${m[3]}/${m[2]}` : prevText.slice(0, 8);
    doc.text(`até ${display}`, footX, y + 25);
  } else if (temPA) {
    doc.setTextColor(100, 100, 110);
    doc.text('c/ ação', footX, y + 25);
  }

  // Tag barreiras à direita do rodapé
  if (barr && barr.count) {
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(6);
    doc.setTextColor(140, 80, 20);
    const tagText = `Barr:${barr.count}`;
    doc.text(tagText, x + w - 2, y + 25, { align: 'right' });
  }

  // Reset cores/fonte
  doc.setFont('helvetica', 'normal');
  doc.setTextColor(0, 0, 0);
}
```

- [ ] **Step 2: Smoke isolado**

Console:
```js
const pdfLib = await carregarJsPDF();
const doc = new pdfLib.jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
// Renderiza 4 cards lado a lado pra ver paleta
[allPatients[0], allPatients[1], allPatients[2], allPatients[3]].forEach((p, i) => {
  _pdfDrawBedCard(doc, 12 + i * 47, 30, 44, 28, p, { isComanejo: false });
});
doc.save('test-cards.pdf');
```

**Expected**: download com 4 cards mostrando: leito, especialidade, dias grande à esq, nome abreviado, prazo ou SP, badge canto sup direito se aplicável, bullet azul se tem PA, tag barreiras se tem. Cores F0-F4 conforme dias.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T6] _pdfDrawBedCard paridade visual completa"
```

---

## Task 7: `_pdfDrawWardHeader(doc, esp, rank, total, pacientes, capacidade)` — header com ranking

**Files:**
- Modify: `index.html:~12380` (add antes do `_pdfPageEspecialidade`)

- [ ] **Step 1: Add função header**

Insert:

```js
/** Header de página de especialidade: cor + ranking + stats completas. */
function _pdfDrawWardHeader(doc, esp, rank, total, pacientes, capacidade, cont) {
  const marginX = 12;
  const color = _pdfCorEspecialidade(esp);
  const tmp = _tmpMedio(pacientes);
  const ocupados = pacientes.length;
  const livres = (typeof capacidade === 'number' && capacidade > 0) ? Math.max(0, capacidade - ocupados) : 0;
  const spCnt = pacientes.filter(p => typeof isSP === 'function' && isSP(p)).length;
  const spPct = ocupados ? Math.round(spCnt / ocupados * 100) : 0;

  // Linha 1: #rank pequeno cinza + ESP grande colorida
  doc.setFont('helvetica', 'normal');
  doc.setFontSize(8);
  doc.setTextColor(120, 120, 130);
  doc.text(`#${rank}/${total}`, marginX, 12);

  doc.setFont('helvetica', 'bold');
  doc.setFontSize(14);
  doc.setTextColor(color[0], color[1], color[2]);
  doc.text(esp + (cont ? ' (cont.)' : ''), marginX + 14, 12);

  // Linha 2: stats
  doc.setFont('helvetica', 'normal');
  doc.setFontSize(8);
  doc.setTextColor(80, 80, 90);
  const capTxt = (typeof capacidade === 'number' && capacidade > 0) ? `${ocupados}/${capacidade} leitos` : `${ocupados} leitos ocupados`;
  const livresTxt = livres > 0 ? ` · ${livres} livre${livres === 1 ? '' : 's'}` : '';
  const tmpTxt = ocupados ? ` · TMP ${tmp.toFixed(1)}d` : '';
  const spTxt = ocupados ? ` · SP ${spCnt} (${spPct}%)` : '';
  doc.text(`${capTxt}${livresTxt}${tmpTxt}${spTxt}`, marginX, 18);

  // Reset
  doc.setTextColor(0, 0, 0);
}
```

- [ ] **Step 2: Smoke isolado**

Console:
```js
const pdfLib = await carregarJsPDF();
const doc = new pdfLib.jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
const espGrp = {};
allPatients.forEach(p => {
  const k = (p.especialidade || 'Sem').trim();
  (espGrp[k] = espGrp[k] || []).push(p);
});
const ranking = _ordenarEspecialidadesPorTMP(espGrp);
ranking.forEach((r, i) => {
  if (i > 0) doc.addPage();
  _pdfDrawWardHeader(doc, r.esp, i + 1, ranking.length, r.pacs, null, false);
});
doc.save('test-headers.pdf');
```

**Expected**: download com N páginas, cada uma só com header (sem cards). Confere: `#1/N` + esp colorido + stats batem com dashboard ward block.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T7] _pdfDrawWardHeader com ranking #N/Total"
```

---

## Task 8: `_pdfPageEspecialidade(doc, esp, pacientes, rank, total)` — orquestra página

**Files:**
- Modify: `index.html:~12430`

- [ ] **Step 1: Add função página**

Insert:

```js
/** Renderiza 1 página de especialidade: header + grid bed-cards. Paginação automática (cont.). */
function _pdfPageEspecialidade(doc, esp, pacientes, rank, total) {
  const pageH = 297, marginX = 12;
  const cardW = 44, cardH = 28, gapX = 3, gapY = 3, cardsPerRow = 4;
  const topY = 24; // header ocupa 0-22

  _pdfDrawWardHeader(doc, esp, rank, total, pacientes, null, false);

  let row = 0, col = 0;
  pacientes.forEach(p => {
    const x = marginX + col * (cardW + gapX);
    let y = topY + row * (cardH + gapY);
    if (y + cardH > pageH - 12) {
      doc.addPage();
      _pdfDrawWardHeader(doc, esp, rank, total, pacientes, null, true);
      row = 0; col = 0;
      y = topY;
    }
    _pdfDrawBedCard(doc, x, y, cardW, cardH, p, { isComanejo: false });
    col++;
    if (col >= cardsPerRow) { col = 0; row++; }
  });
}
```

- [ ] **Step 2: Smoke isolado**

Console:
```js
const pdfLib = await carregarJsPDF();
const doc = new pdfLib.jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
const espGrp = {};
allPatients.forEach(p => {
  const k = (p.especialidade || 'Sem').trim();
  (espGrp[k] = espGrp[k] || []).push(p);
});
const ranking = _ordenarEspecialidadesPorTMP(espGrp);
ranking.forEach((r, i) => {
  if (i > 0) doc.addPage();
  const pacsOrd = _ordenarCardsPorDias(r.pacs);
  _pdfPageEspecialidade(doc, r.esp, pacsOrd, i + 1, ranking.length);
});
doc.save('test-pages.pdf');
```

**Expected**: download com N páginas completas. Header + grid 4 cols. Cards em ordem dias desc. Paginação "(cont.)" funcionando pra especialidades que estouram A4.

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T8] _pdfPageEspecialidade orquestra header + grid + paginacao"
```

---

## Task 9: `_pdfPageComanejo(doc, pacientes)` — página dedicada Comanejo

**Files:**
- Modify: `index.html:~12480`

- [ ] **Step 1: Add função Comanejo**

Insert:

```js
/** Renderiza página Comanejo (retaguarda PS). Threshold próprio (3d default, lê localStorage).
 *  Fora do ranking principal: sem #N/Total no header. */
function _pdfPageComanejo(doc, pacientes) {
  const pageH = 297, marginX = 12;
  const cardW = 44, cardH = 28, gapX = 3, gapY = 3, cardsPerRow = 4;
  const topY = 24;

  const lpThr = (() => {
    const stored = parseInt(localStorage.getItem(`retaguarda_lp_threshold_${HOSPITAL_ID}`), 10);
    return (stored >= 1 && stored <= 7) ? stored : 3;
  })();

  const color = _pdfCorEspecialidade('Comanejo');
  const tmp = _tmpMedio(pacientes);
  const lpCount = pacientes.filter(p => (calcDiasHoje(p) || 0) > lpThr).length;

  function drawHeader(cont) {
    doc.setFont('helvetica', 'bold');
    doc.setFontSize(14);
    doc.setTextColor(color[0], color[1], color[2]);
    doc.text(`Comanejo${cont ? ' (cont.)' : ''}`, marginX, 12);

    doc.setFont('helvetica', 'normal');
    doc.setFontSize(8);
    doc.setTextColor(80, 80, 90);
    doc.text(`Retaguarda PS · LP >${lpThr}d · ${pacientes.length} leitos em retaguarda · TMP ${tmp.toFixed(1)}d · LP ${lpCount}`, marginX, 18);
    doc.setTextColor(0, 0, 0);
  }
  drawHeader(false);

  let row = 0, col = 0;
  pacientes.forEach(p => {
    const x = marginX + col * (cardW + gapX);
    let y = topY + row * (cardH + gapY);
    if (y + cardH > pageH - 12) {
      doc.addPage();
      drawHeader(true);
      row = 0; col = 0;
      y = topY;
    }
    _pdfDrawBedCard(doc, x, y, cardW, cardH, p, { isComanejo: true, lpThreshold: lpThr });
    col++;
    if (col >= cardsPerRow) { col = 0; row++; }
  });
}
```

- [ ] **Step 2: Smoke isolado**

Console:
```js
const pdfLib = await carregarJsPDF();
const doc = new pdfLib.jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
const resp = await fetch(`${SB_URL}/rest/v1/internacoes_hmsa?hospital=eq.${HOSPITAL_ID}&setor=eq.Comanejo&status_internacao=eq.ativa&select=*&order=leito.asc`, { headers: SB_H });
const comanejo = await resp.json();
const ord = _ordenarCardsPorDias(comanejo);
_pdfPageComanejo(doc, ord);
doc.save('test-comanejo.pdf');
```

**Expected**: download com 1 página Comanejo: header próprio "Comanejo · Retaguarda PS · LP >3d · ...". Cards Comanejo com threshold 3d (cards >3d ficam laranja/vermelho mais cedo que dashboard).

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T9] _pdfPageComanejo threshold 3d proprio + header dedicado"
```

---

## Task 10: `exportarPDFLeitos()` — orquestrador final

**Files:**
- Modify: `index.html:~12550`

- [ ] **Step 1: Add função orquestradora**

Insert:

```js
/** Exporta PDF consolidado: capa + N pags especialidades (TMP desc) + Comanejo (isolado fim). */
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

  // Capa
  _pdfDrawCapa(doc, dash, comanejo);

  // Agrupa dashboard por especialidade, ordena por TMP médio desc
  const especsMap = {};
  dash.forEach(p => {
    const k = (p.especialidade || 'Sem especialidade').toString().trim();
    if (!especsMap[k]) especsMap[k] = [];
    especsMap[k].push(p);
  });
  const ranking = _ordenarEspecialidadesPorTMP(especsMap);
  const totalEspecs = ranking.length;

  // 1 página por especialidade, em ordem TMP desc, cards ordenados por dias desc
  ranking.forEach(({ esp, pacs }, idx) => {
    doc.addPage();
    const pacsOrd = _ordenarCardsPorDias(pacs);
    _pdfPageEspecialidade(doc, esp, pacsOrd, idx + 1, totalEspecs);
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

- [ ] **Step 2: Smoke pelo botão real**

Reload `http://localhost:8765` no browser. Aguarda load dos 134 pacientes. Clica botão `PDF` na toolbar.

**Expected**: download `HOB_leitos_DD-MM-AAAA.pdf` com:
- P1: Capa
- P2-PN: Especialidades em ordem TMP médio desc
- P(N+1): Comanejo

- [ ] **Step 3: Commit**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git add index.html && git commit -m "[HOB PDF T10] exportarPDFLeitos orquestrador final"
```

---

## Task 11: Smoke completo — 8 checkboxes do spec

**Files:** nenhum — só verificação.

- [ ] **Step 1: Reload local + clica PDF**

Reload `http://localhost:8765/#/gestao-leitos/dashboard`. Aguarda banner sync conclui. Clica `PDF`.

- [ ] **Step 2: Verificar 8 checkboxes**

Abre `HOB_leitos_DD-MM-AAAA.pdf` e confere:
- [ ] Capa com totais corretos (bater com counters do dashboard)
- [ ] N páginas em ordem TMP médio desc — TMP do header P1 ≥ P2 ≥ ... ≥ PN
- [ ] Cards em ordem dias desc dentro de cada página — primeiro card tem mais dias que o último
- [ ] Header de cada página tem `#N/Total · Esp · X leitos · TMP Wd · SP M (P%)`
- [ ] Bed-cards mostram: leito grande, esp, dias grande, nome abreviado, prazo ou SP, badge se aplicável, bullet PA, tag barreiras
- [ ] Cores F0-F4 batem visualmente com dashboard
- [ ] Última página é Comanejo, header próprio "LP >3d"
- [ ] Cards Comanejo usam threshold 3d (vermelho/F4 entra mais cedo que dashboard)

- [ ] **Step 3: Se ALGUM falhar — pause + report ao Maestro**

Stop-the-line conforme memória `controles-anti-erro-recorrente` C7. Não comitar push até resolver.

- [ ] **Step 4: Se todos passarem — confirmar visualmente com Maestro**

Mostra checkbox completo. Aguarda Maestro confirmar "ok pra push".

---

## Task 12: Push pra main → deploy auto + smoke prod

**Files:** nenhum — push.

- [ ] **Step 1: Push pra main**

```bash
cd "C:/Users/Francisco/Desktop/claude-cowork/hob-leitos" && git push origin main
```

**Expected**: push aceito. Git integration Vercel dispara deploy auto (memória `vercel-deploy-hob-tem-git-integration`).

- [ ] **Step 2: Aguardar deploy READY**

Vercel MCP:
```
mcp__claude_ai_Vercel__list_deployments project=hob-leitos limit=3
```

Aguarda último deployment com `state: READY` e `target: production`.

- [ ] **Step 3: Smoke prod**

Maestro abre `https://hob-leitos.vercel.app/#/gestao-leitos/dashboard`. Confere botão PDF aparece. Clica. Confere PDF baixa e tem mesma qualidade do smoke local.

- [ ] **Step 4: Update wip_atual.md + daily 12/mai**

Em `second_brain_md_package/`:
- Daily 12/mai com bloco "HOB PDF Paridade TMP — entregue"
- `wip_atual.md` linha de status

- [ ] **Step 5: Não criar commit final separado** — Task 10 já tem `exportarPDFLeitos` orquestrador. Tasks 0-10 cobrem 11 commits incrementais.

---

## Self-Review

Verificação rápida do plano vs spec:

1. **Spec coverage:**
   - Q1 ordenação dupla → Task 4 (helpers) + Task 10 (orquestrador) ✓
   - Q2 paridade + nome → Task 6 (`_pdfDrawBedCard` com `abreviarNomePaciente`) ✓
   - Q3 Comanejo isolado + threshold próprio → Task 9 (`_pdfPageComanejo`) ✓
   - Q4 header com ranking → Task 7 (`_pdfDrawWardHeader`) ✓
   - Abordagem A jsPDF nativo + paleta → Task 2 (`carregarJsPDF`) + Task 3 (`_pdfPaleta`) ✓
   - 10 componentes da spec → 12 tasks ✓ (T3 cobre 3 helpers de paleta numa task, T4 cobre 3 helpers de ordenação numa task)
   - Bed-card anatomia → Task 6 ✓
   - Estrutura saída → Task 10 ✓
   - Error handling → coberto em cada função relevante ✓
   - 8 checkboxes smoke → Task 11 ✓
   - Push deploy auto → Task 12 ✓

2. **Placeholder scan:** Zero "TBD", zero "implement later", zero "add appropriate error handling" sem mostrar como. Cada step tem código ou comando exato. ✓

3. **Type consistency:**
   - `_tmpMedio(pacientes)` retorna número, usado em `_ordenarEspecialidadesPorTMP` e `_pdfDrawWardHeader` consistente ✓
   - `_pdfPaleta[tierKey]` shape `{fill, border, text}` usado consistentemente ✓
   - `_pdfDrawBedCard(doc, x, y, w, h, p, opts)` signature mesma em Task 6 e Task 8/9 ✓
   - `_pdfCorEspecialidade(esp)` retorna `[r,g,b]` usado consistente em headers ✓

Sem gaps. Plano completo.

---

## Histórico

- 2026-05-12 Plano criado via `superpowers:writing-plans` após spec aprovada (commit 99f058e). Maestro autorizou execução direta sem checkpoints intermediários (memória `executar-recomendacoes-sem-perguntar`).
