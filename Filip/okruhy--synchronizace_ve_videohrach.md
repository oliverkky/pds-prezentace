# Synchronizace v online videohrách — plán prezentace

**Kurz:** Paralelní a distribuované systémy
**Délka:** 40 min (cca 36 min výklad + 4 min rezerva / dotazy)
**Ústřední teze:** Herní netcode (hráčský pojem označující síťovou komunikaci a synchronizaci ve videohrách) z velké části znovu objevil (nebo přímo převzal) klasické výsledky distribuovaných systémů — replikaci stavového automatu, konzervativní a optimistickou synchronizaci a kompromis konzistence × latence. Každý blok proto má dvě vrstvy: *herní technika* → *teoretický ekvivalent*.

---

## Přehled časování

| # | Blok | Min |
|---|------|-----|
| 1 | Motivace: latence jako fyzikální limit | 3 |
| 2 | Architektury a model autority | 4 |
| 3 | Teoretický rámec | 5 |
| 4 | Deterministický lockstep = replikace stavového automatu | 6 |
| 5 | Server-autoritativní model (vč. časové osy klienta) | 9 |
| 6 | Optimistická synchronizace: rollback netcode = Time Warp | 7 |
| 7 | Závěr a srovnání | 2 |
| | **Celkem výklad** | **36** |
| | Rezerva / dotazy | 4 |

---

## 1. Motivace: latence jako fyzikální limit (3 min)

**Cíl:** Ukázat, že problém není „špatný kód“, ale nenulová a nestejná latence mezi replikami sdíleného stavu.

**Slidy:**
1. **Titulní slide**
2. **Problém v jedné větě** — N hráčů, každý má lokální repliku herního světa, zprávy mezi nimi putují desítky až stovky ms. Přitom hráč vnímá zpoždění vstupu už v řádu desítek ms.
3. **Fyzikální dolní mez** — rychlost světla ve vlákně ≈ 2·10⁸ m/s → Bratislava–Frankfurt (~600 km vzdušnou čarou) ≥ 3 ms jedním směrem jen šířením, reálně výrazně víc (trasa, routery, fronty). Citát Riotu: dokud se lidstvo nenaučí pohybovat rychleji než světlo, peeker's advantage nezmizí.
4. **Tři konfliktní požadavky** — konzistence (všichni vidí totéž), odezva (vstup se projeví hned), férovost (výsledek nezávisí na pingu). Nelze mít vše najednou → zbytek prezentace = různé kompromisy.

**Poznámka pro řečníka:** Konkrétní výpočet šíření signálu je lepší než abstraktní „latence existuje“ — dá posluchačům číslo, ke kterému se dá vracet.

---

## 2. Architektury a model autority (4 min)

**Cíl:** Zavést otázku *kdo je zdrojem pravdy* a hranici důvěry. Všechny další techniky na tom závisí.

**Slidy:**
1. **Topologie**
   - *Dedikovaný server* — klienti komunikují jen se serverem (CS, Valorant, Overwatch).
   - *Listen server / host* — jeden z klientů je zároveň server (častý u co-op a indie her).
   - *Peer-to-peer* — každý s každým (klasické RTS, fighting games s rollbackem).
   - *Asynchronní* — hráči nemusí být online současně (korespondenční šachy, aukční dům).
2. **Model autority**
   - *Server-authoritative* — klient posílá **vstupy**, server počítá stav a rozesílá výsledek.
   - *Client-authoritative* — klient posílá **výsledný stav** svých objektů, server jen přeposílá. Nulový input lag, ale server se vzdává kontroly.
   - Kompetitivní hry: server-authoritative + client-side prediction (viz blok 5). Client-authoritative je přijatelné jen tam, kde nevadí podvádění (co-op).
3. **Hranice důvěry**
   - Server nemůže věřit klientům: upravený klient může posílat nemožné pozice (speedhack, teleport).
   - **Terminologická poznámka:** nejde o byzantskou odolnost (BFT) — server je jediná důvěryhodná autorita, mezi rovnocennými uzly neprobíhá konsenzus. Klienti jsou prostě mimo hranici důvěry.
4. **Co autorita serveru neřeší**
   - *Manipulace se stavem* (speedhack, teleport) → řeší autorita serveru.
   - *Únik informací* (wallhack) → autorita nestačí, protože klient pozice nepřátel dostává. Řešení: **Fog of War** (Riot, League of Legends → VALORANT) — server pozice nepřátel zadržuje, dokud je klient nepotřebuje zobrazit; při selhání kontroly přímé viditelnosti přestane o daném aktérovi posílat aktualizace.
   - *Automatizace vstupů* (aimbot) → ze síťové vrstvy prakticky nerozlišitelné od legitimního vstupu; otevřený problém.

**Poznámka pro řečníka:** Fog of War je přirozený most k distribuovaným systémům — server rozhoduje, *komu* jakou část stavu replikuje, tj. replikace není nutně úplná.

---

## 3. Teoretický rámec (5 min)

**Cíl:** Dát posluchačům slovník, kterým se pak popíše každá technika.

**Slidy:**
1. **Dvě strategie replikace**
   - *Replikovat vstupy* (+ deterministický přechodový automat) → málo dat, vysoké nároky na determinismus.
   - *Replikovat stav* (snapshoty) → více dat, tolerance k nedeterminismu.
2. **Logický čas: tick / frame**
   - Simulace postupuje v diskrétních krocích (tick). 128 tick = 7,8125 ms, 64 tick = 15,625 ms.
   - **Terminologická poznámka:** tick *není* Lamportovy hodiny. Lamportovy hodiny zachycují kauzalitu mezi událostmi bez globální shody; tick je sdílený diskrétní *virtuální čas*, na jehož hodnotě se všichni shodují — blíže pojmu *virtual time* (Jefferson).
3. **Kompromis konzistence × latence**
   - CAP se týká chování při síťovém rozdělení; pro hry je relevantnější **PACELC** (Abadi, 2012): i bez rozdělení (*Else*) se volí mezi *Latency* a *Consistency*.
   - Každá herní technika v prezentaci je bod na této ose.
4. **Konzervativní × optimistická synchronizace** (z paralelní diskrétní simulace, PDES)
   - *Konzervativní* — zpracuj událost, až je jisté, že nepřijde žádná starší (Chandy–Misra–Bryant, bariéry).
   - *Optimistická* — zpracuj hned, při chybě se vrať zpět (Time Warp, Jefferson 1985).
   - Ukázat mapu: lockstep → konzervativní, rollback → optimistický, server-autoritativní model → primární kopie + spekulativní klienti.

---

## 4. Deterministický lockstep = replikace stavového automatu (6 min)

**Cíl:** První konkrétní technika; přímá aplikace state machine replication.

**Slidy:**
1. **Princip**
   - Posílají se jen vstupy, každý účastník simuluje celý svět.
   - Krok N se provede, až jsou k dispozici vstupy všech hráčů pro krok N → bariérová synchronizace.
   - Teorie: **state machine replication** (Schneider, 1990) — stejný počáteční stav + stejná posloupnost vstupů + deterministický přechod ⇒ stejný stav.
2. **Případová studie: Age of Empires** („1500 Archers on a 28.8“, GDC 2001)
   - Tisíce jednotek na modemu 28,8 kbit/s → posílat stav nepřipadá v úvahu.
   - Příkazy se plánují s předstihem (vydané v tahu N se vykonají v tahu N+2) → latence se schová za input delay.
   - Délka tahu se adaptivně řídí: každý klient hlásí udržitelný frame rate a nejdelší pozorovaný ping, host podle toho upravuje délku tahu. Doba tahu kolísala až o 200 ms.
3. **Cena: nejpomalejší účastník určuje tempo**
   - Hra může běžet jen tak rychle, jak nejpomalejší stroj zvládne zpracovat komunikaci a vykreslit tah.
   - *Pozor na chybu v úvodním videu:* P2P lockstep **nezmírňuje** dopad hráče se špatným připojením — zpomalí všechny, jen se to řídí adaptivně.
4. **Nejtěžší část: determinismus floating-point výpočtů**
   - Zdroje odchylek: x87 (80bitová mezipřesnost) vs. SSE2; kontrakce `a*b+c` do FMA (`-ffp-contract`); přeuspořádání při `-ffast-math`; pořadí paralelních redukcí; rozdílné implementace transcendentních funkcí v libm.
   - Gaffer on Games: očekávat stejný výsledek z libovolného FP kódu napříč kompilátory, architekturami, či dokonce debug/release buildy je naivní.
   - Řešení: fixed-point aritmetika, nebo striktní IEEE 754 režim + stejný binární build + vlastní implementace transcendentních funkcí.
   - Detekce desynchronizace: kontrolní součty stavu po tazích.
5. **Měřené limity**
   - Mieschke (HdM Stuttgart, 2024): kvantitativní vyhodnocení vertikálního (počet entit) a horizontálního (počet hráčů) škálování lockstepu vs. snapshot interpolation.

**Poznámka pro řečníka:** Slide o FP determinismu je ideální místo pro konkrétní ukázku — např. stejný výpočet zkompilovaný s `-ffp-contract=off` a `=fast` a rozdíl v posledních bitech výsledku.

---

## 5. Server-autoritativní model (9 min) — jádro pro FPS

**Cíl:** Vysvětlit, jak se spojuje autorita serveru s okamžitou odezvou, a proč pak každý hráč vidí jinou „realitu“.

**Slidy:**
1. **Základní smyčka**
   - Klient posílá vstupy, server simuluje s pevným tickrate a rozesílá snapshoty (Source: tickrate 66 v TF2/CSS; klient standardně přijímá ~20 snapshotů/s).
2. **Snapshot interpolation**
   - Klient bufferuje snapshoty a vykresluje vzdálené entity interpolací mezi dvěma známými stavy → plynulý pohyb i při ztrátách a přeuspořádání paketů.
   - Cena: umělá latence. Source: `lerp = cl_interp_ratio / cl_updaterate` (2/20 = 100 ms); ratio 3 chrání proti jednomu ztracenému paketu, 4 proti dvěma po sobě.
   - *Pozor na chybu v úvodním videu:* velikost bufferu se neodvozuje od RTT („ping 100 ms → buffer 200 ms“), ale od intervalu mezi snapshoty, jitteru a tolerované ztrátovosti. RTT určuje stáří stavu, ne velikost bufferu.
3. **Interpolace × extrapolace**
   - *Interpolace* — mezi dvěma známými stavy (bezpečné, ale zpožděné).
   - *Extrapolace / dead reckoning* — odhad z poslední pozice a rychlosti (bez zpoždění, ale chybuje při změně směru → „rubber banding“). Původ: distribuované vojenské simulace (DIS).
   - *Pozor na chybu v úvodním videu:* extrapolace pohybu míčku z rychlosti **není** client-side prediction.
4. **Client-side prediction + server reconciliation**
   - Klient okamžitě aplikuje *vlastní* vstupy (spekulativní provedení), čísluje je a uchovává.
   - Server vrátí autoritativní stav + číslo posledního zpracovaného vstupu → klient převezme stav serveru a znovu aplikuje nepotvrzené vstupy.
   - Teorie: lokální spekulace nad primární kopií; korekce = malý lokální rollback.
   - Overwatch (Tim Ford, GDC 2017): architektura ECS a determinismus simulace jako předpoklad přesné predikce.
5. **⭐ Časová osa klienta — „každý vidí jinou realitu“** (klíčový slide)
   - Jedna vodorovná osa serverového času `t`:
     - **Vlastní postava** je *v budoucnosti*: predikce je o ≈ jednosměrnou latenci před serverem, aby vstupy dorazily včas.
     - **Vzdálené entity** jsou *v minulosti*: ≈ jednosměrná latence + interpolační buffer za serverem.
   - Důsledek: dva hráči nikdy nesdílejí tentýž okamžik herního světa. Otázka „čí realitu použít pro vyhodnocení zásahu?“ nemá řešení bez kompromisu — přímý důsledek PACELC (blok 3).
   - *Volitelně:* odhad serverového času klientem ≈ Cristianův algoritmus (předpoklad symetrické cesty, RTT/2).
6. **Lag compensation a „favor the shooter“**
   - Server při zpracování výstřelu přetočí čas o latenci střelce + jeho interpolační zpoždění a vyhodnotí zásah ve stavu, který střelec viděl. Source standardně uchovává 1 s historie pozic/animací.
   - Důsledek: oběť může dostat zásah, když už je na své obrazovce za zdí. Shora omezené oknem historie (1 s v Source).
   - Teorie: server vědomě vyhodnocuje událost v minulém stavu → férovost pro střelce na úkor oběti.
7. **Peeker's advantage a tickrate (VALORANT)**
   - Hráč, který vykoukne zpoza rohu, vidí oběť dřív, než oběť vidí jeho.
   - Nečekané zjištění Riotu: velikost výhody závisí na rychlosti připojení *oběti*, ne útočníka.
   - Riot zvolil 128-tick servery, privátní páteřní síť (Riot Direct) a minimální bufferování; jako férovou hranici uvádí < 80 ms.

**Poznámka pro řečníka:** Slide 5 (časová osa) je nejlepší kandidát na vlastní nakreslený diagram — všechny následující slidy (lag compensation, peeker's advantage) se z něj dají odvodit graficky.

---

## 6. Optimistická synchronizace: rollback netcode = Time Warp (7 min)

**Cíl:** Ukázat nejsilnější paralelu prezentace — herní technika z roku 2009 je v podstatě algoritmus z PDES z roku 1985.

**Slidy:**
1. **Problém fighting games**
   - Snímková přesnost (60 FPS → 16,67 ms/snímek), P2P, dva hráči. Delay-based netcode (čekej na vstup soupeře) → proměnlivý input lag.
2. **Rollback (GGPO, Tony Cannon, 2009)**
   - Lokální vstupy se zpracují hned; vstupy soupeře se **predikují** (typicky „opakuje poslední vstup“).
   - Když dorazí skutečný vstup a liší se → obnov stav z checkpointu, resimuluj až do aktuálního snímku.
   - GGPO od roku 2019 open source (MIT); dnes standard žánru (Guilty Gear Strive, Skullgirls…).
3. **Kvantitativní požadavek**
   - Resimulace *k* snímků se musí vejít do jednoho snímku: `k · t_sim + t_render ≤ 16,67 ms`.
   - → simulace musí být řádově rychlejší než realtime a stav musí jít levně ukládat/obnovovat (serializace celého stavu každý snímek).
   - Stejný požadavek determinismu jako lockstep (blok 4).
4. **⭐ Mapování na Time Warp (Jefferson 1985)**

   | Time Warp (PDES) | Rollback netcode |
   |---|---|
   | logický proces (LP) | instance hry u hráče |
   | optimistické zpracování událostí | simulace s predikovanými vstupy |
   | straggler (zpráva „z minulosti“) | opožděný vstup soupeře |
   | state queue / checkpointy | uložené stavy posledních snímků |
   | rollback | obnova stavu + resimulace |
   | GVT (dolní mez budoucích rollbacků) | poslední potvrzený snímek |
   | fossil collection | zahození starých checkpointů |
   | anti-messages | **nejsou potřeba** — posílají se jen vstupy, výstup je deterministický |

   - Time Warp OS bere rollback jako *normální* mechanismus synchronizace, ne jako zotavení z chyby — stejně jako rollback netcode.
5. **Akademický most: Trailing State Synchronization**
   - Cronin et al. (2002): několik kopií stavu s různým zpožděním; při nekonzistenci se opraví z „trailing“ kopie. Navrženo pro FPS.
   - Konstatují, že dosavadní synchronizační metody byly převzaty z vojenských distribuovaných simulací a nebyly optimalizované pro rychlé hry.

---

## 7. Závěr a srovnání (2 min)

**Slidy:**
1. **Souhrnná tabulka**

   | Technika | Teoretický ekvivalent | Co obětuje |
   |---|---|---|
   | Deterministický lockstep | state machine replication, konzervativní synchronizace | odezvu (input delay), tempo nejpomalejšího |
   | Snapshot interpolation | replikace stavu z primární kopie | aktuálnost (umělá latence) |
   | Client-side prediction | lokální spekulace + korekce | krátkodobou konzistenci (rubber banding) |
   | Lag compensation | vyhodnocení v minulém stavu | férovost vůči oběti |
   | Rollback netcode | optimistická synchronizace (Time Warp) | výpočetní výkon, vizuální „skoky“ |

2. **Pointa:** Jefferson & Barnes (2017, *Virtual Time III*) sjednotili konzervativní a optimistickou synchronizaci — hybridní herní netcode (rollback pro blízké entity, interpolace pro ostatní) dělá prakticky totéž.

---

## Poznámky ke zdrojům

- **CS2 „sub-tick“** — nanejvýš okrajová zmínka. Veřejně dostupné je hlavně marketingové video Valve a komunitní spekulace; bez vlastního měření (pcap, analýza dem) nejde o podložené tvrzení.
- Valve Developer Wiki, technický blog Riotu a GDC přednášky jsou primární, ale nerecenzované zdroje. Recenzovanou oporu tvoří Jefferson, Cronin et al., Schneider, Abadi.

## Zdroje

**Distribuované systémy / PDES**
- D. Jefferson et al.: *Distributed Simulation and the Time Warp Operating System* — https://www.ics.uci.edu/~cs237/reading/files/Distributed%20Simulation%20and%20the%20Time%20Wrap%20OS.pdf
- R. Fujimoto: slidy *Time Warp: Local Control* — https://sigsim.acm.org/conf/mskr/Courseware/Fujimoto/Slides/FujimotoSlides-11-TimeWarpLocalControl.pdf
- D. Jefferson, P. Barnes: *Virtual Time III: Unification of Conservative and Optimistic Synchronization in PDES* (WSC 2017) — https://www.informs-sim.org/wsc17papers/includes/files/058.pdf
- F. B. Schneider: *Implementing Fault-Tolerant Services Using the State Machine Approach* (ACM Computing Surveys, 1990)
- D. Abadi: *Consistency Tradeoffs in Modern Distributed Database System Design* (IEEE Computer, 2012) — PACELC

**Synchronizace ve hrách — akademické**
- E. Cronin et al.: *An Efficient Synchronization Mechanism for Mirrored Game Architectures* (NetGames 2002) — https://www.eecg.utoronto.ca/~ashvin/courses/ece1746/2003/reading/cronin-netgames02.pdf
- P. Mieschke: *Deterministic Lockstep in Networked Games* (diplomová práce, HdM Stuttgart, 2024) — https://hdms.bsz-bw.de/files/7106/Deterministic_Lockstep_in_Networked_Games.pdf

**Průmyslové zdroje**
- M. Terrano, P. Bettner: *1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond* (GDC 2001) — https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond
- Valve: *Source Multiplayer Networking* — https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking
- Valve: *Interpolation* — https://developer.valvesoftware.com/wiki/Interpolation
- Valve: *Lag Compensation* — https://developer.valvesoftware.com/wiki/Lag_Compensation
- Y. Bernier: *Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization* (2001) — https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization
- T. Ford: *'Overwatch' Gameplay Architecture and Netcode* (GDC 2017) — https://gdcvault.com/play/1024001
- Riot: *Peeking into VALORANT's Netcode* — https://technology.riotgames.com/node/111
- Riot: *VALORANT's 128-Tick Servers* — https://technology.riotgames.com/node/112
- Riot: *Demolishing Wallhacks with VALORANT's Fog of War* — https://technology.riotgames.com/node/108
- G. Fiedler: *Deterministic Lockstep*, *Snapshot Interpolation*, *Floating Point Determinism* — https://gafferongames.com/post/deterministic_lockstep/, https://gafferongames.com/post/snapshot_interpolation, https://gafferongames.com/post/floating_point_determinism
- GGPO (open source, MIT) — https://github.com/pond3r/ggpo
- Infil: *Explaining how fighting games use delay-based and rollback netcode* — https://words.infil.net/w02-netcode-p5.html
