# Trainingsuren (onbetaald) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Per medewerker onbetaalde trainingsuren invoeren die als aftrek (uren × uurloon) van het maandtotaal gaan.

**Architecture:** Alles zit in één bestand, `index.html` (React + in-browser Babel, geen build, geen tests). Trainingsuren volgen exact het patroon van `incentiveHours`: opslag in `agentMonthData`, berekening in `computeAgentPayroll`, invoer in de MonthTab-agentkaart, weergave in PayslipCard en de jsPDF-loonstrook — maar dan als negatief bedrag.

**Tech Stack:** React 18 via CDN, Babel standalone v7 (JSX in `<script type="text/babel">`), Tailwind CDN, jsPDF. Geen testframework — verificatie is handmatig in de browser (`open index.html`).

## Global Constraints

- Alle UI-teksten in het Nederlands; label exact **"Trainingsuren (onbetaald)"**.
- Trainingsuren tellen NIET mee in `qualifyingHours` (aanwezigheidsbonus-drempel blijft ongewijzigd).
- Geen afkapping: `totalGross` mag negatief worden.
- Veldnaam in data: `trainingHours` (getal, default 0), naast `incentiveHours` in `agentMonthData[agentId]`.
- Spec: `docs/superpowers/specs/2026-08-03-trainingsuren-design.md`.
- Regelnummers hieronder zijn indicatief (stand vóór dit werk); zoek op het getoonde codefragment, niet blind op regelnummer.

---

### Task 1: State, setter en berekening

**Files:**
- Modify: `index.html` — `getAgentMD` (~727), setters (~732), `computeAgentPayroll` (~820–882), `MonthTab`-props (~1331)

**Interfaces:**
- Produces: `setTrainingHours(agentId, hours)`; `computeAgentPayroll` retourneert extra velden `trainingHours` (getal) en `trainingDeduction` (getal, positief bedrag dat afgetrokken is). Taken 2–4 gebruiken deze exact zo.

- [ ] **Step 1: Default toevoegen in `getAgentMD`**

Vervang:
```js
return agentMonthData[agentId] || { incentiveHours: 0, sales: {}, manualBonuses: [], systemOutages: [] };
```
door:
```js
return agentMonthData[agentId] || { incentiveHours: 0, trainingHours: 0, sales: {}, manualBonuses: [], systemOutages: [] };
```

- [ ] **Step 2: Setter toevoegen** — direct onder `setIncentiveHours` (~regel 732):

```js
function setTrainingHours(agentId, hours) { updateAgentMD(agentId, md => ({ ...md, trainingHours: hours })); }
```

- [ ] **Step 3: Berekening in `computeAgentPayroll`** — direct onder de twee incentive-regels (`const incHours = ...; const incentivePay = ...;`, ~regel 821-822):

```js
const trainingHours = Number(md.trainingHours) || 0;
const trainingDeduction = trainingHours * wage;
```

- [ ] **Step 4: Aftrekken van totaal** — pas de `totalGross`-regel (~865) aan:

```js
const totalGross = workPay + holidayPay + incentivePay + outagePay + toiletPay
  + totalProjectBonus + totalManualBonus + hourBonusAmount + totalFixedAllowances
  - trainingDeduction;
```

Laat `qualifyingHours` (~857) ONGEWIJZIGD.

- [ ] **Step 5: Retourneren** — voeg in het return-object (~868) toe, direct na `incentiveHours: incHours, incentivePay,`:

```js
trainingHours, trainingDeduction,
```

- [ ] **Step 6: Prop doorgeven** — in de `<MonthTab ...>`-aanroep (~1331) én de destructuring van `function MonthTab({...})` (~1718): voeg `setTrainingHours` toe direct naast `setIncentiveHours`.

```jsx
setIncentiveHours={setIncentiveHours} setTrainingHours={setTrainingHours} setSales={setSales}
```
en in de destructuring:
```js
setIncentiveHours, setTrainingHours, setSales, addManualBonus, updateManualBonus, removeManualBonus,
```

- [ ] **Step 7: Verifieer in de browser dat er geen fouten zijn**

Run: `open "/Users/haruntekin/Loon Dashboard/index.html"` en check de browserconsole: geen Babel/JS-fouten, app rendert zoals voorheen.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Trainingsuren: state, setter en loonaftrek in berekening"
```

---

### Task 2: Invoerveld in agentkaart

**Files:**
- Modify: `index.html` — MonthTab agentkaart: dichtgeklapte regel (~1877–1880) en uitgeklapt blok "Incentive uren" (~1918–1925)

**Interfaces:**
- Consumes: `setTrainingHours(agentId, hours)` en `md.trainingHours` uit Task 1.

- [ ] **Step 1: Invoerblok toevoegen** — direct ONDER het bestaande "Incentive uren"-blok (het `<div>` dat eindigt na de incentive-`Input`, ~regel 1925), binnen dezelfde `space-y-4`-kolom:

```jsx
<div>
  <div className="text-xs text-stone-500 uppercase tracking-wider font-medium mb-2">Trainingsuren (onbetaald)</div>
  <div className="grid grid-cols-[1fr_90px] gap-2 items-center">
    <div className="text-sm text-stone-600">Aantal uren (wordt afgetrokken)</div>
    <Input type="number" step="0.25" value={md.trainingHours || ''} onChange={(e) => setTrainingHours(a.id, parseFloat(e.target.value) || 0)} placeholder="0" />
  </div>
</div>
```

- [ ] **Step 2: Indicator in dichtgeklapte regel** — in het `<div className="text-xs text-stone-500">`-blok (~1877), na de incentive-span:

```jsx
{(md.trainingHours || 0) > 0 && <span className="mr-3 text-rose-600">−{fmtHours(md.trainingHours)} training</span>}
```

- [ ] **Step 3: Verifieer in de browser**

Open de app, tab met maandinvoer → klap een medewerker uit → vul bijv. `2` in bij Trainingsuren. Verwacht: waarde blijft staan na dichtklappen/openen, en de dichtgeklapte regel toont "−2 training" in rood.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Trainingsuren: invoerveld per medewerker in maandtab"
```

---

### Task 3: Loonstrook op scherm

**Files:**
- Modify: `index.html` — `PayslipCard` (~2302–2325)

**Interfaces:**
- Consumes: `calc.trainingHours`, `calc.trainingDeduction` uit Task 1.

- [ ] **Step 1: Negatieve regel toevoegen** — in de rows-grid van `PayslipCard`, direct ná de `manualBonuses.map(...)`-regels (~2324), vóór het sluiten van de grid-`div`:

```jsx
{calc.trainingHours > 0 && (
  <PayslipRow label={`Trainingsuren onbetaald (${fmtHours(calc.trainingHours)} uur)`} value={`− ${fmtEuro(calc.trainingDeduction)}`} />
)}
```

- [ ] **Step 2: Verifieer in de browser**

Loonstroken-tab: medewerker met 2 trainingsuren en uurloon €10 toont de regel "Trainingsuren onbetaald (2,00 uur)" met waarde `− € 20,00`, en het Netto-bedrag bovenaan is €20,00 lager dan zonder trainingsuren.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Trainingsuren: aftrekregel op loonstrook (scherm)"
```

---

### Task 4: PDF-loonstrook

**Files:**
- Modify: `index.html` — `buildPdfDoc`, tussen het "Incentive bonussen"-blok (~1037–1048) en het TOTAAL NETTO-blok (~1050)

**Interfaces:**
- Consumes: `c.trainingHours`, `c.trainingDeduction`, bestaande helpers `sectionHeader(title)`, `row(label, qty, rate, amount, bold)`, `fmtHours`, `fmtEuro`, `c.wage`.

- [ ] **Step 1: PDF-sectie toevoegen** — direct ná het `if (c.manualBonuses.length > 0) {...}`-blok en vóór `ensureSpace(22);`:

```js
if (c.trainingHours > 0) {
  sectionHeader('Trainingsuren (onbetaald)');
  row('Trainingsuren', fmtHours(c.trainingHours) + ' uur', fmtEuro(c.wage), '− ' + fmtEuro(c.trainingDeduction));
  y += 3;
}
```

- [ ] **Step 2: Verifieer in de browser**

Download een PDF van een medewerker met trainingsuren. Verwacht: sectie "Trainingsuren (onbetaald)" met `− € ...`-bedrag, en TOTAAL NETTO gelijk aan het schermbedrag (dus inclusief aftrek). Ook een PDF van iemand zónder trainingsuren: geen sectie zichtbaar.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Trainingsuren: aftreksectie in PDF-loonstrook"
```
