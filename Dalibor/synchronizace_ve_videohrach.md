# Synchronizace ve videohrách (Paralelní a distribuované systémy)

Videohry představují jeden z nejnáročnějších reálných případů nasazení distribuovaných systémů. V reálném čase (často 60+ snímků za sekundu) musí řešit fundamentální konflikt známý z **CAP teorému**: obětují okamžitou globální konzistenci ve prospěch nízké latence a dostupnosti (**Eventual Consistency**).

## 1. Základní pojmy a síťové architektury

### Tick Rate vs. Frame Rate

* **Frame Rate (FPS):** Frekvence, s jakou klient vykresluje obraz na monitor.

* **Tick Rate (Hz):** Frekvence, s jakou server (nebo simulace) vyhodnocuje herní logiku, fyziku a síťové pakety (typicky v rozmezí 20 Hz až 128 Hz).

* **Zneužití (Tick Manipulation):** Útočník uměle pozdrží odchozí pakety (*choke*) a odešle je najednou. Server pak v jediném ticku vyhodnotí více akcí najednou (např. tzv. *Double Tap* exploit, kdy hráč vystřelí více kulek z automatické zbraně v jednom okamžiku).

### Peer-to-Peer (P2P) / Listen Server

* **Princip:** Klienti komunikují napřímo bez centrálního dedikovaného serveru, nebo jeden z hráčů plní roli autoritativního hostitele (*Listen Server*).

* **Zneužití:**

  * **Host Advantage & Force Host:** Hráč, který je hostitelem, má nulový ping a plnou kontrolu nad stavem hry v RAM. Cheateři pomocí nástrojů vynutí, aby hra zvolila jako hosta vždy je.

  * **DDoS útoky (IP Leaking):** Klienti musí znát své IP adresy pro přímou komunikaci. Útočník snadno vyčte IP protihráče ze sítě a zahltí mu připojení.

### Client-Server (Dedikovaný autoritativní server)

* **Princip:** Server drží jedinou „pravdu“ o stavu světa. Klienti jsou „hloupé terminály“ – posílají pouze vstupy (stisky kláves, pohyb myši) a server jim vrací výsledný stav.

* **Zneužití:** Odolné proti přímé manipulaci se stavem (nelze si přepsat HP v paměti), ale otevírá prostor pro **automatizaci vstupů** (*Aimbot, Triggerbot*) a čtení dat, která server klientovi poslal navíc (*Wallhack / ESP*).

## 2. Modely synchronizace stavu

### A) Deterministický Lockstep

V teorii distribuovaných systémů jde o **State Machine Replication**. Místo výsledného stavu světa se po síti posílají pouze události (vstupy hráčů).

* **Technický princip:**

  1. Čas hry je rozdělen na diskrétní kroky (*Lockstep Turns*, např. po 100 ms).

  2. Klienti sbírají vstupy v kroku $T$, zabalí je a pošlou všem ostatním s časovým razítkem $T + 2$ (rezerva na síťovou latenci).

  3. **Bariérová synchronizace:** Simulace kroku $T + 2$ se nesmí spustit, dokud nedorazí potvrzení (nebo prázdný paket) od **všech** účastníků.

  4. Každý klient spustí identický simulační kód. Výpočty musí být bitově identické napříč různými CPU a OS – používá se aritmetika s pevnou řádovou čárkou (*Fixed-Point Math*) namísto standardního `float`/`double`.

* **Zneužití:**

  * **Map Hack (Information Leak):** Protože každý klient simuluje celý svět u sebe, má v RAM pozice všech jednotek i ve *Fog of War*. Cheat pouze čte vlastní operační paměť.

  * **Drop Hack / Desync Exploit:** Pokud hráč prohrává, úmyslně změní jednu proměnnou v paměti (např. HP své jednotky). Tím poruší determinismus, hra vyhlásí chybu synchronizace (*Out of Sync*) a zápas zruší bez započítání prohry.

### B) Rollback Netcode (Optimistická replikace)

Evoluce Lockstepu navržená pro rychlé akční hry, kde je nepřípustné čekat na pakety protihráče (nulový *input lag*).

* **Technický princip:**

  1. **Lokální predikce:** Hráč A zmáčkne úder ve snímku 10 a klient jej okamžitě vykoná. Protože ještě nemá vstup Hráče B pro snímek 10, **predikuje**, že Hráč B opakuje svůj poslední známý vstup (např. stále drží blok).

  2. **State Serialization:** Před každým snímkem si engine uloží kompletní stav hry do kruhového bufferu v RAM (typicky posledních 7–15 snímků).

  3. **Rollback a Resimulace:** Ve snímku 13 dorazí paket od Hráče B, že ve snímku 10 skočil. Klient Hráče A zjistí neshodu, načte uložený stav ze snímku 9, bez vykreslování bleskově přepočítá snímky 10 až 13 se správným vstupem a ve snímku 14 vykreslí opravený stav.

* **Zneužití:**

  * **Lag Switching (One-sided Rollback):** Útočník krátkodobě zablokuje odchozí pakety. Protihráčův klient neustále predikuje, že útočník stojí na místě. Jakmile útočník spojení obnoví, hra se protihráči zpětně přepočítá a útočník se na něj „teleportuje“ uprostřed úderu bez animace nápřahu.

### C) State Synchronization (Snapshot Interpolation)

Simulace běží výhradně na autoritativním serveru, který klientům pravidelně posílá snímky (*Snapshots*) stavu světa.

* **Technický princip:**

  1. **Tvorba snímků:** Na konci každého serverového ticku server serializuje pozice, rotace a stavy dynamických entit.

  2. **Delta komprese & Kvantizace:** Server posílá pouze rozdíly (*delty*) oproti poslednímu snapshotu, který klient potvrdil přes ACK paket. Čísla se komprimují na úroveň bitů (*bit-packing*).

  3. **Entity Interpolation:** Klient drží **interpolační buffer** (zpoždění typicky o 2 ticky, např. 30–100 ms). Čeká, až má v bufferu snímek $S_1$ a $S_2$, a vykresluje ostatní hráče v minulosti mezi těmito dvěma body pomocí lineární/sférické interpolace (LERP/SLERP). Tím vyhlazuje síťový *jitter*.

* **Zneužití:**

  * **Obcházení Anti-Wallhacku:** Aby server zabránil wallhackům, posílá data o nepřátelích jen tehdy, když jsou blízko rohu (*Interest Management / Network Culling*). Cheaty tyto pakety odchytí o zlomek sekundy dříve, než se nepřítel skutečně vykloní.

## 3. Sub-tick architektura

Hybridní přístup, který řeší problém **kvantizace vstupů v čase** bez nutnosti navyšovat výpočetní frekvenci serveru (tickrate).

* **Problém klasického pevného tickratu:** Při pevném tickratu (např. 64 Hz = 1 tick každých 15,625 ms) se všechny akce hráčů provedené uvnitř tohoto okna sesbírají až na konci ticku. Z pohledu serveru se tak dva výstřely na začátku a na konci téhož ticku stanou ve stejný diskrétní okamžik, čímž se ztrácí přesná kauzalita událostí.

* **Technický princip:**

  1. **Sub-tick časová razítka:** Server sice stále simuluje svět v pevných krocích, ale klient každý vstup (výstřel, pohyb, skok) opatří přesným relativním časovým razítkem uvnitř daného ticku (hodnota `fraction` od `0.0` do `1.0`, vázaná na lokální snímek klienta).

  2. **Rekonstrukce kauzality:** Při vyhodnocování ticku server seřadí události podle jejich sub-tick razítek (např. výstřel v `100.23` proběhl dříve než výstřel v `100.81`).

  3. **Interpolace stavu v momentě akce:** Server při výpočtu zásahu (raycastu) interpoluje pozice, rychlosti a úhly pohledu hráčů přesně do daného zlomku ticku, nikoliv na konec celého ticku.

* **Výhody:**

  * **Přesné pořadí událostí a hit-registrace:** Střelba a míření jsou nezávislé na tom, v jaké fázi ticku hráč stiskl tlačítko.

  * **Nižší výpočetní nároky na server:** Odpadá nutnost provozovat drahé vysokofrekvenční servery (128 Hz a více), protože základní simulační smyčka zůstává na nižší frekvenci.

* **Nevýhody:**

  * **Větší síťový provoz (Bandwidth):** Klient musí posílat detailní historii událostí včetně jejich přesných sub-tick časů, což výrazně zvětšuje velikost paketů a zvyšuje citlivost hry na ztrátu paketů (*packet loss*).

  * **Zpožděná vizuální odezva (Feedback Latency):** I když se zásah vypočítá v přesném sub-tick čase, potvrzení ze serveru dorazí až po dokončení celého ticku, což bez dodatečné klientské predikce animací působí opožděně.

  * **Nekonzistence fyziky pohybu:** Přesné vyhodnocování výskoků v libovolném zlomku ticku vede k mírně odlišným trajektoriím skoků, což často vyžaduje uzamčení pohybu zpět na pevné ticky.

* **Zneužití:**

  * **Sub-tick Timestamp Manipulation / Rapid Fire:** Útočník upraví odchozí pakety tak, aby obsahovaly uměle nahuštěná sub-tick razítka (např. odeslání více výstřelů s minimálním časovým rozestupem `0.01` v rámci jednoho paketu), čímž obejde standardní prodlevu mezi výstřely zbraně.

## 4. Techniky skrývání latence (Latency Hiding)

### A) Client-Side Prediction & Server Reconciliation

Řeší problém, jak ovládat vlastní postavu bez zpoždění o celý *Round-Trip Time* (RTT), aniž bychom obětovali autoritu serveru.

* **Technický princip:**

  1. **Číslování vstupů:** Klient každému vstupu přiřadí pořadové číslo (`input_id`), uloží jej do fronty a pošle serveru.

  2. **Predikce:** Klient okamžitě spustí pohybovou fyziku lokálně a posune kameru.

  3. **Reconciliation (Usmiřování):** Server vstup zpracuje a pošle klientovi autoritativní pozici společně s posledním zpracovaným `input_id`. Klient smaže z fronty starší vstupy a porovná svou historickou predikci s odpovědí serveru. Pokud se liší (např. kolize s jiným hráčem), přepíše svou pozici daty ze serveru a v jednom cyklu znovu aplikuje všechny nepotvrzené vstupy z fronty.

* **Zneužití:**

  * **Speedhack / Teleport (Timer Manipulation):** Cheater zrychlí systémové hodiny svého PC. Klient pak posílá pakety se vstupy rychleji, než by měl. Pokud server striktně nekontroluje časová razítka a maximální možnou rychlost pohybu za sekundu, postava se pohybuje nadlidskou rychlostí.

  * **Flyhack / Noclip:** U her se slabou validací fyziky na straně serveru klient predikuje pohyb vzduchem a server tento stav přijme jako platný.

### B) Lag Compensation (Server Rewind)

Umožňuje hráčům mířit přímo na cíl, i když mají vysoký ping, aniž by museli střílet *před* nepřítele (*leading shots*).

* **Technický princip:**

  1. **Historie na serveru:** Server si ukládá historii hitboxů všech hráčů za poslední např. 1 sekundu do kruhového bufferu.

  2. **Výstřel:** Hráč A vystřelí. Paket obsahuje přesné časové razítko toho, **jaký snímek světa Hráč A zrovna viděl na monitoru** (odvozeno z posledního snapshotu a interpolace).

  3. **Přetočení času (Rewind):** Server po přijetí paketu dočasně posune hitboxy všech ostatních hráčů zpět v čase o ping střelce + jeho interpolační zpoždění. Provede výpočet dráhy střely (*raycast*), zaznamená zásah a okamžitě vrátí hitboxy zpět do přítomnosti.

  4. *(Vysvětluje fenomén „smrti za zdí“: Hráč B už na svém klientu zaběhl za zeď, ale v čase, kdy vystřelil Hráč A, byl ještě na otevřeném prostranství).*

* **Zneužití:**

  * **Fake Lag / Backtrack Cheat:** Cheater uměle zpožďuje své odchozí pakety (např. o 200 ms), čímž nutí server přetáčet čas na maximum povoleného limitu. Cheater pak může vystřelit do prázdného místa, kudy nepřítel běžel před 0,2 sekundami, a server zásah uzná jako trefu.

  * **Peeker's Advantage Abuse:** Hráč s vysokým pingem (nebo umělým lagem) vyběhne zpoza rohu a okamžitě vystřelí. Než se paket o jeho pohybu dostane přes server k nepříteli, nepřítel je mrtvý, aniž by útočníka vůbec viděl vykouknout.

## 5. Souhrnná tabulka kompromisů

| Architektura / Technika | Hlavní výhoda | Hlavní slabina / Riziko cheatu | 
| ----- | ----- | ----- | 
| **Deterministický Lockstep** | Minimální datový tok pro tisíce entit | **Map Hack** (vše je v RAM klienta) a **Desync** | 
| **Rollback Netcode** | Pocit nulové latence v rychlých duelech | **Lag Switch** (teleportační údery) | 
| **Sub-tick architektura** | Přesná kauzalita bez vysokého tickratu | Větší velikost paketů, **Timestamp manipulation** | 
| **Authoritative + Prediction** | Vysoká bezpečnost, server drží pravdu | **Speedhack** (při chybějící validaci času) | 
| **Lag Compensation** | Férové střílení i s vyšším pingem | **Backtrack** (střelba do „duchů“ v minulosti) | 