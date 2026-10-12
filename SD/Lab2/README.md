# Laboratorul 2: Cuantifică citirile Dashboard-ului

> Ordinea de lucru: Secțiunile 2 → 3 → 1 → 4 → 5. Cerințele de calitate din Secțiunea 1 au nevoie de valorile RPS drept condiție de operare, deci traficul se calculează primul.

**Citiri analizate:** Overview, Filter, Stock price, History, Watchlist, Search.

**Presupunere comună:** fiecare acțiune a Utilizatorului creează o cerere către Dashboard. O cerere către Dashboard nu creează neapărat o cerere către furnizor; traficul furnizorului nu este modelat aici.

---

## 1. Cerințe de calitate

**Punct de dimensionare:** nivelul de 3,000 Utilizatori concurenți. Regim stabil: 792 RPS. La deschiderea pieței: 1,030 RPS (Secțiunile 2 și 3).

Un rezultat „indisponibil" sau o eroare nu este o citire corectă. Un rezultat indisponibil returnat rapid poate respecta ținta de latență și poate încălca totuși disponibilitatea sau consistența, de aceea cele trei sunt măsurate separat.

### Latența citirilor

**Începutul și sfârșitul măsurării:** de la momentul în care Dashboard-ul primește cererea până la momentul în care a trimis ultimul octet al răspunsului. Timpul de rețea și de afișare de pe dispozitivul Utilizatorului nu este inclus; ținta de 2 secunde a clientului lasă marja pentru el.

> În orele de tranzacționare, cu o încărcare totală de până la 1,030 RPS (nivelul de 3,000 Utilizatori, inclusiv deschiderea pieței), latența citirilor corecte trebuie să respecte:

| Citire | p50 | p95 | Limita la care citirea este oprită | Motiv |
| ------ | --- | --- | ---------------------------------- | ----- |
| Stock price | ≤ 150 ms | ≤ 500 ms | 2 s | Prețul trebuie să „pară imediat" și este cea mai frecventă citire (600 RPS). |
| Overview | ≤ 500 ms | ≤ 2 s | 4 s | Adună mai multe instrumente într-un clasament; folosește pragul de 2 s al clientului. |
| Filter | ≤ 300 ms | ≤ 1 s | 3 s | Este repetat de 3 ori pe minut, deci Utilizatorul așteaptă după fiecare schimbare. |
| History | ≤ 800 ms | ≤ 2 s | 5 s | Seria de 5 ani are cele mai multe puncte, deci cea mai lungă coadă. |
| Watchlist | ≤ 400 ms | ≤ 1.5 s | 4 s | Adună prețurile mai multor instrumente ale aceluiași Utilizator. |
| Search | ≤ 300 ms | ≤ 1 s | 3 s | Este repetat de 3 ori pe minut; căutarea trebuie să răspundă repede. |

**Ce se întâmplă la limita de 2 secunde:** 2 s este ținta p95 pentru majoritatea citirilor, nu un prag absolut. O citire corectă între 2 s și limita ei de oprire intră în cei 5% permiși. La limita de oprire, citirea este întreruptă și Utilizatorul vede o eroare, nu o încărcare fără sfârșit. Citirile oprite nu contează ca reușite la latență; ele se numără la Debit.

### Disponibilitate bazată pe timpul de funcționare

**Când este Dashboard-ul utilizabil.** Un minut este utilizabil dacă sunt îndeplinite ambele condiții:
1. mai puțin de 5% din cererile din minutul respectiv se termină cu eroare sau sunt oprite la limita lor;
2. cel puțin 95% din instrumentele acceptate au un preț acceptat cu vechime de cel mult 25 de minute (regula A, mai jos).

Un minut care nu îndeplinește ambele condiții este indisponibilitate. O aplicație care se încarcă, dar nu poate arăta prețuri, nu este utilizabilă.

**Orele de tranzacționare:** 10:00–18:00, ora României, pe Piața Reglementată a BVL (tranzacționare continuă până la 17:45, închidere la 17:50, tranzacționare la prețul de închidere până la 18:00) = 8 ore pe zi. Restul zilei = 16 ore.

| Interval | Ore pe zi | Timp măsurat în 30 de zile | Țintă | Buget de indisponibilitate |
| -------- | --------- | -------------------------- | ----- | -------------------------- |
| Orele de tranzacționare | 8 | 8 × 30 = 240 h | 99.9%  | 240 × 0.001 = 0.24 h |
| Restul zilei | 16 | 16 × 30 = 480 h | 99%  | 480 × 0.01 = 4.8 h |

> În fiecare fereastră de 30 de zile, Dashboard-ul trebuie să fie utilizabil cel puțin 99.9% din timpul măsurat al orelor de tranzacționare și cel puțin 99% din timpul măsurat al restului zilei.

**De ce ținte diferite.** Timpul de funcționare numără fiecare secundă la fel, dar impactul asupra Utilizatorilor nu este egal: zece minute de indisponibilitate la prânz, când piața se mișcă și oamenii verifică prețuri, afectează mult mai mulți Utilizatori decât zece minute la 03:00. O țintă identică ar fi fie prea strictă (și scumpă) tot timpul, fie prea slabă când contează. Ținta mai relaxată din afara orelor de tranzacționare lasă și loc pentru mentenanță și actualizări.

**De ce nu 99.99%.** 240 × 0.0001 = 0.024 h = 86.4 s pe lună. O singură pauză a furnizorului mai lungă de un minut și jumătate ar epuiza bugetul, iar costul unei redundanțe care să o prevină este prea mare pentru un produs care doar afișează date.

*Simplificare:* se folosesc 30 de zile calendaristice. BVL nu tranzacționează în weekend, deci un model mai strict ar folosi aproximativ 21–22 de zile de tranzacționare.

### Consistență

**Regula A: vechimea prețului unei acțiuni.** Furnizorul deține valoarea sursă; Dashboard-ul decide cum o arată. Vechimea = ora curentă − ora furnizorului. Întârzierea normală a furnizorului este de aproximativ 15 minute, deci este situația normală, nu o defecțiune; tolerăm încă 10 minute.

> Dacă vechimea celui mai recent preț acceptat este de cel mult 25 de minute, Dashboard-ul îl arată cu ora furnizorului și cu marcajul „întârziat". Dacă vechimea depășește 25 de minute sau nu există niciun preț acceptat, rezultatul este „indisponibil". Nu se arată niciodată zero și niciodată un preț vechi prezentat ca actual.

| Vechimea prețului | Ce vede Utilizatorul |
| ----------------- | -------------------- |
| 0–25 min | Preț cu ora furnizorului și marcajul „întârziat" (normal, datele sunt cu ~15 min în urmă chiar când totul funcționează). |
| Peste 25 min | „Indisponibil"; prețul nu se mai afișează. |
| Niciun preț acceptat | „Indisponibil", niciodată zero. |

Peste limită refuzăm să arătăm prețul, în loc să îl etichetăm: la date financiare Utilizatorii citesc numărul, nu eticheta mică de lângă el.

**Regula B: Watchlist după o modificare și acces.**

> Doar Utilizatorul autentificat care deține un Watchlist îl poate citi sau modifica. După ce o modificare (adăugare sau eliminare) a fost confirmată, următoarea citire a Watchlist-ului de către același Utilizator arată conținutul anterior plus instrumentul adăugat sau minus instrumentul eliminat.

| Situație | Ce vede Utilizatorul |
| -------- | -------------------- |
| Proprietarul citește după o modificare confirmată | Conținutul actualizat |
| Alt Utilizator sau un vizitator neautentificat încearcă să citească sau să modifice | „Acces interzis", nu o listă goală |
| Watchlist-ul este gol | Stare goală explicită |
| Watchlist-ul nu poate fi încărcat | Eroare clară, nu o listă goală |

Dacă o modificare nu este confirmată, Watchlist-ul rămâne neschimbat, iar Utilizatorul este informat.

### Debit

> În fereastra de 10 secunde de la deschiderea pieței (10:00), cu până la 1,030 RPS, cel mult 1% din cereri pot eșua din cauza supraîncărcării (respinse sau oprite la limita citirii). Se numără citirile finalizate în limita lor, nu doar răspunsurile trimise.

---

## 2. Estimări RPS în regim stabil

Formula: `RPS = Utilizatori concurenți × cota participantă × acțiuni pe Utilizator / secunde`. Toate intervalele sunt convertite în secunde înainte de împărțire; rezultatele nu sunt rotunjite.

| Citire | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| ------ | --------------- | ----------------- | ------------------ |
| Overview | 300 × 0.70 × 1 / 30 = 7 | 3,000 × 0.70 × 1 / 30 = 70 | 30,000 × 0.70 × 1 / 30 = 700 |
| Filter | 300 × 0.50 × 3 / 60 = 7.5 | 3,000 × 0.50 × 3 / 60 = 75 | 30,000 × 0.50 × 3 / 60 = 750 |
| Stock price | 300 × 0.20 × 1 / 1 = 60 | 3,000 × 0.20 × 1 / 1 = 600 | 30,000 × 0.20 × 1 / 1 = 6,000 |
| History | 300 × 0.20 × 1 / 300 = 0.2 | 3,000 × 0.20 × 1 / 300 = 2 | 30,000 × 0.20 × 1 / 300 = 20 |
| Watchlist | 300 × 0.60 × 1 / 60 = 3 | 3,000 × 0.60 × 1 / 60 = 30 | 30,000 × 0.60 × 1 / 60 = 300 |
| Search | 300 × 0.10 × 3 / 60 = 1.5 | 3,000 × 0.10 × 3 / 60 = 15 | 30,000 × 0.10 × 3 / 60 = 150 |
| **Total în regim stabil** | 7 + 7.5 + 60 + 0.2 + 3 + 1.5 = **79.2** | 70 + 75 + 600 + 2 + 30 + 15 = **792** | 700 + 750 + 6,000 + 20 + 300 + 150 = **7,920** |

Modelul este liniar: coloanele 3,000 și 30,000 sunt coloana de 300 înmulțită cu 10 și cu 100.

**Observație:** Stock price înseamnă 60 din 79.2 RPS (≈ 76%) la 300 de Utilizatori, mai mult decât toate celelalte citiri la un loc. Este o ipoteză de investigat în Secțiunea 5, nu o dovadă de blocaj.

---

## 3. Estimări RPS la deschiderea pieței

La deschidere, 30% dintre Utilizatori reîmprospătează Overview o dată într-un interval de 10 secunde, iar 60% **din acest grup** reîmprospătează și Watchlist. Aceste fluxuri se adaugă traficului stabil, care continuă neschimbat (rândurile stabile Overview și Watchlist nu se numără a doua oară).

| Calcul la deschiderea pieței | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| ---------------------------- | --------------- | ----------------- | ------------------ |
| Trafic stabil de citire | 79.2 | 792 | 7,920 |
| Flux suplimentar de reîmprospătare Overview | 300 × 0.30 × 1 / 10 = 9 | 3,000 × 0.30 × 1 / 10 = 90 | 30,000 × 0.30 × 1 / 10 = 900 |
| Flux suplimentar de reîmprospătare Watchlist | 300 × 0.30 × 0.60 × 1 / 10 = 5.4 | 3,000 × 0.30 × 0.60 × 1 / 10 = 54 | 30,000 × 0.30 × 0.60 × 1 / 10 = 540 |
| **Subtotal la deschiderea pieței** | 79.2 + 9 + 5.4 = **93.6** | 792 + 90 + 54 = **936** | 7,920 + 900 + 540 = **9,360** |
| Marjă de capacitate de 10% | 93.6 × 0.10 = 9.36 | 936 × 0.10 = 93.6 | 9,360 × 0.10 = 936 |
| **Țintă rotunjită în sus la deschiderea pieței** | 93.6 × 1.10 = 102.96 → **103** | 936 × 1.10 = 1,029.6 → **1,030** | 9,360 × 1.10 = 10,296 → **10,296** |

Se rotunjește în sus doar ținta finală; la 30,000 de Utilizatori rezultatul este exact 10,296. Marja de 10% nu acoperă operații lipsă sau ipoteze greșite despre comportamentul Utilizatorilor.

---

## 4. Estimări de stocare

### Ce este un „Stock" în acest Dashboard

Un „Stock" este un instrument financiar tranzacționat pe piața aleasă, identificabil printr-un simbol (ticker), care are preț și istoric. Acțiunile ordinare și ETF-urile se tranzacționează pe bursă, iar prețul lor se schimbă în timpul zilei la fel ca al acțiunilor.

| Decizie | Alegere | Justificare |
| ------- | ------- | ----------- |
| Piețe / burse | Bursa de Valori din Londra, doar Piața Reglementată | O singură piață, într-un singur fus orar, cu program clar (10:00–18:00), același cu fereastra din Secțiunea 1. Restul de peste 200 de companii listate (sistemul alternativ) rămân în afară . |
| Tipuri incluse | Acțiuni ordinare și ETF-uri | Acoperă ce ar urmări realist un investitor individual și au ambele tip și sector pentru Filter. |
| Tipuri excluse | Obligațiuni, unități de fond non-ETF, produse structurate, certificate, drepturi de preferință | Fiecare adaugă alt model de preț și de identitate, fără nevoie în prima versiune. |
| Instrumente inactive sau delistate | Nu mai rămân disponibile în Search, Filter și Overview; în Watchlist apar marcate „delistat", fără preț | Un preț oprit nu poate fi prezentat ca actual. Istoricul lor nu se afișează în v1 și nu este în estimare. |
| Număr de Stocks | 88 acțiuni + 10 ETF-uri = **98** | Vezi sursele. Simplificare: un instrument de tip acțiune pe companie. |

| Informație | Valoare | Sursă | Observat |
| ---------- | ------- | ----- | -------- |
| Companii pe Piața Reglementată BVL | 88 | comunicatul BVL din 13 august 2026, preluat de Business24 | 12 octombrie 2026 |
| ETF-uri tranzacționabile la BVL | 10 | declarația președintelui ASF din 18 august 2026, preluată de Business24 | 12 octombrie 2026 |
| Programul de tranzacționare | 10:00–18:00 | programul publicat de BVL | 12 octombrie 2026 |

### Datele sincronizate

Fiecare câmp stocat se leagă de o citire; câmpurile furnizorului fără nevoie de produs nu se copiază.

| Citire | Câmpuri stocate | De ce |
| ------ | --------------- | ----- |
| Search | `instrument_id`, `ticker`, `name` | Potrivirea cu textul introdus. |
| Filter | `type`, `sector` | Toate instrumentele sunt de pe L, deci tipul (acțiune/ETF) și sectorul sunt criteriile utile. |
| Overview | `current_price`, `previous_close`, `provider_timestamp` | Clasamentul necesită variația procentuală (preț curent față de închiderea anterioară) și ora furnizorului. |
| Stock price | `current_price`, `provider_timestamp`, `status` | `status` marchează lipsa prețului ca „indisponibil", nu zero (regula A). |
| History | `instrument_id`, `timestamp`, `price` | Seria din spatele evoluției pe perioade. |
| Watchlist | — | Este starea Utilizatorului, nu date sincronizate de furnizor. |

**Frecvența de sincronizare (presupunere):**

| Date | Frecvență | Motiv |
| ---- | --------- | ----- |
| Prețuri curente | La 5 minute, în 10:00–18:00 | Vechimea maximă = 15 min întârziere + 5 min = 20 min, sub limita de 25 min. Un ciclu ratat duce la exact 25 min; două cicluri ratate fac prețul indisponibil. |
| Istoric zilnic | O dată pe zi, după închidere | Valoarea zilei este finală după sesiune. |
| Date despre instrumente | O dată pe zi, înainte de deschidere | Numele, tipul și sectorul se schimbă rar. |

Punctele din seriile intraday și de 30 de minute se iau din prețurile sincronizate la 5 minute.

### Seriile de istoric

Perioadele 1W și 1M folosesc aceeași serie de 30 de minute, iar 6M, 1Y și 5Y aceeași serie zilnică, la lungimi diferite de felie. Sesiune = 10:00–18:00 = 480 min. Păstrarea se numără în zile de tranzacționare (≈ 250 pe an), pentru că în weekend nu se generează puncte.

| Serie | Interval | Păstrare | Servește | Puncte / instrument / zi | Puncte păstrate / instrument |
| ----- | -------- | -------- | -------- | ------------------------ | ---------------------------- |
| Intraday | 5 min | 5 zile de tranzacționare | 1D | 480 / 5 = 96 | 96 × 5 = 480 |
| Medie | 30 min | 25 de zile de tranzacționare | 1W, 1M | 480 / 30 = 16 | 16 × 25 = 400 |
| Zilnică | 1 zi | 5 ani × 250 = 1,250 de zile | 6M, 1Y, 5Y | 1 | 1,250 |
| **Total** | | | | **113** | **2,130** |

### Calculul stocării

```text
raw storage = record count x average bytes per record
history record count = 98 Stocks x 2,130 points per Stock = 208,740
```

Înregistrare reprezentativă de istoric: `instrument_id` 4 B + `timestamp` 8 B + `price` 8 B = 20 B, rotunjit la 24 B cu aliniere.

| Set de date | Decizia despre produs și păstrare | Calculul numărului de înregistrări | Octeți per înregistrare | Stocare brută |
| ----------- | --------------------------------- | ---------------------------------- | ----------------------- | ------------- |
| Date de referință Stock | Toate cele 98, cât timp sunt listate | 98 | 120 B (`id` 4 + `ticker` 12 + `name` 64 + `type` 1 + `sector` 32 = 113, rotunjit) | 11,760 B ≈ 11.5 KiB |
| Cele mai recente prețuri | Una pe instrument, suprascrisă la fiecare sincronizare | 98 | 32 B (`id` 4 + `price` 8 + `prev_close` 8 + `provider_ts` 8 + `status` 1 = 29, rotunjit) | 3,136 B ≈ 3.1 KiB |
| Price history | Trei serii, păstrare conform tabelului | 98 × 2,130 = 208,740 | 24 B | 5,009,760 B ≈ 4.78 MiB |
| Alte date selectate | Niciuna; Watchlist este starea Utilizatorului | — | — | — |
| **Total** | | | | **5,024,656 B ≈ 4.79 MiB** |

| Indicator | Calcul | Rezultat |
| --------- | ------ | -------- |
| Stocare inițială | cu istoric complet încărcat la lansare, ca să existe 5Y din prima zi | ≈ 4.79 MiB |
| Înregistrări noi pe zi | 98 × 113 = 11,074 | 11,074 × 24 B = 265,776 B ≈ **0.25 MiB / zi** scrise |
| Scris într-un an | 11,074 × 250 zile = 2,768,500 înregistrări | 2,768,500 × 24 B = 66,444,000 B ≈ **63.4 MiB / an** |
| Stocat după un an | păstrarea limitează fiecare serie | ≈ 4.79 MiB (rămâne constant) |

Volumul scris și volumul păstrat răspund la întrebări diferite: primul arată presiunea de scriere, al doilea capacitatea necesară. Rămân în afara estimării indecșii, replicile, jurnalele, copiile de siguranță, copiile temporare și istoricul instrumentelor delistate. Cu doar aproximativ 5 MiB, stocarea nu este o presiune pe calitate în acest domeniu de aplicare; presiunea vine din numărul de cereri (Secțiunea 5).

---

## 5. Posibile blocaje

Fiecare rând este o ipoteză de verificat, nu o cauză dovedită. O valoare RPS mare este un motiv de investigare, nu o dovadă.

| Calitate | Posibil blocaj | Dovezi din acest laborator | Efect posibil | Ce trebuie măsurat în continuare |
| -------- | -------------- | -------------------------- | ------------- | -------------------------------- |
| Latență | Calea de citire Stock price | 600 din 792 RPS (≈ 76%) în regim stabil, cu cea mai strictă țintă (p95 ≤ 500 ms). Sunt doar 98 de instrumente: 600 / 98 ≈ 6.1 RPS pe instrument, iar prețurile curente ocupă ≈ 3 KiB, deci aceleași răspunsuri sunt cerute de foarte multe ori. | Dacă fiecare cerere repetă tot lucrul (citire, calcul de vechime, marcaj), cererile se așteaptă una după alta și p95 depășește 500 ms, deși volumul de date este mic. | p50/p95 pentru Stock price, măsurate separat, la 1,030 RPS total; procentul cererilor repetate pentru același instrument într-un minut; timpul de procesare față de timpul de așteptare în coadă. |
| Consistență | Citirea Watchlist imediat după o modificare | Regula B cere conținut actualizat la următoarea citire. Watchlist are 30 RPS în regim stabil și 30 + 54 = 84 RPS la deschidere (3,000 Utilizatori); modificările nu sunt modelate în RPS. | Dacă o citire se bazează pe o stare mai veche decât modificarea confirmată, Utilizatorul vede lista fără instrumentul adăugat (sau cu cel eliminat): se încalcă regula B, deși răspunsul este rapid. | Rata citirilor care întorc conținut vechi după o modificare confirmată; timpul până când modificarea este vizibilă; frecvența modificărilor (nemodelată aici), cu citiri și modificări simultane la deschidere. |
| Debit | Fereastra de deschidere a pieței, de 10 secunde | Încărcarea urcă de la 792 la 936 RPS (+144 RPS, ≈ +18%) într-un interval scurt; ținta cu marjă este 1,030. Fluxul suplimentar conține doar Overview (90) și Watchlist (54), citiri care acoperă mai multe instrumente. | Cozi și epuizarea resurselor în aceste 10 secunde; citirile se opresc la limita lor și depășesc pragul de 1% eșecuri. | Test de încărcare: salt de la 792 la 1,030 RPS timp de 10 s, cu proporția Overview/Watchlist de la deschidere; fracția citirilor finalizate corect în limita lor, lungimea cozii, timpul de revenire la p95 normal. |
| Disponibilitate | Market Data Provider, singurul sistem extern | Vechimea maximă este 25 min, față de 15 min întârziere normală, deci doar 10 min de toleranță. Bugetul este 14.4 min / 30 de zile în orele de tranzacționare, iar System Context are un singur sistem extern. | O pauză a furnizorului mai lungă de ≈ 5–10 min începe să conteze ca indisponibilitate (vechime > 25 min), iar una de ≈ 20–25 min epuizează tot bugetul lunar; nu există o a doua sursă. | Disponibilitatea și latența furnizorului; rata de succes a sincronizării; vechimea celui mai recent preț acceptat; câte minute pe lună mai puțin de 95% din instrumente au un preț de cel mult 25 min. |

---

## Listă de verificare

- [x] Am scris cerințe măsurabile pentru toate calitățile de bază.
- [x] Am ales și am justificat ținte ale timpului de funcționare pentru ambele părți ale zilei.
- [x] Am arătat calculele RPS în regim stabil și la deschiderea pieței pentru toate cele trei niveluri.
- [x] Am cercetat și am definit domeniul de aplicare pentru Stocks acceptate.
- [x] Am estimat stocarea brută inițială, zilnică și pentru un an a datelor despre piață.
- [x] Am precizat presupunerile, unitățile, intervalele și rotunjirea finală.
- [x] Am analizat un posibil blocaj pentru fiecare calitate de bază.