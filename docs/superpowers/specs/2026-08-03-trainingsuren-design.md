# Trainingsuren (onbetaald) — ontwerp

Datum: 2026-08-03

## Doel

Per medewerker onbetaalde trainingsuren kunnen invoeren die van het loon worden afgetrokken.

## Invoer

- In de kaart "Sales, incentives & bonussen per medewerker", direct onder "Incentive uren", komt een blok **"Trainingsuren (onbetaald)"** met één urenveld (type number, stap 0,25).
- Opslag als `trainingHours` in `agentMonthData[agentId]`, naast `incentiveHours`. Default 0 in `getAgentMD`.
- Setter `setTrainingHours(agentId, hours)` naar analogie van `setIncentiveHours`.
- In de dichtgeklapte agentregel wordt naast "+X inc. uur" ook "−X training" getoond zodra `trainingHours > 0`.

## Berekening (`computeAgentPayroll`)

- `trainingHours = Number(md.trainingHours) || 0`
- `trainingDeduction = trainingHours × uurloon`
- `totalGross` wordt verminderd met `trainingDeduction`.
- De kwalificerende uren voor de aanwezigheidsbonus (`qualifyingHours`) blijven **ongewijzigd** — trainingsuren tellen daar niet in mee (niet erbij, niet eraf).
- Retourwaarde bevat `trainingHours` en `trainingDeduction`.

## Loonstrook & PDF

- Als `trainingHours > 0`: sectie **"Trainingsuren (onbetaald)"** met negatieve regel `−X uur × uurloon = −€Y`, vóór het totaal.
- Zowel in de loonstrook op scherm als in de gegenereerde PDF.

## Randgevallen

- Aftrek groter dan totaal: totaal mag negatief worden, geen afkapping — zichtbaar zodat de gebruiker kan corrigeren.

## Buiten scope

- Geen aparte regels/omschrijvingen per training.
- Geen effect op bonusdrempels, sales of incentive bonussen.
