## Dramaturgický návrh

- **Začátek:** situace, kterou zná každý hráč. Schováš se za roh, ale zásah dostaneš, až když už jsi v bezpečí (tzv. _peeker's advantage_). Nebo střílíš přesně na cíl a ve hře to nevyjde.
- **Jedna otázka pro každou architekturu:** (1) Kdo je zdroj pravdy? (2) Jak se maskuje latence? (3) Co se stane, když něco selže nebo někdo podvádí?
- U každé architektury stejné tři otázky a stejný „verdikt" (hra, výhody, nevýhody). Publikum se tak snadno orientuje.

## 1. Úvod

- Háček: „proč mě zasáhl střelec, kterého jsem už neviděl?"
- Proč je to těžké: rychlost světla (Praha–Sydney má v optice nejméně ~100 ms jen na jednu cestu), žádné globální hodiny, ztrátové pakety.
- ⟶ **Reference:** přednáška 1 a 6 (žádné společné hodiny, relativita „teď"), Lamport.
	- Možné řešení: nenechat hrát všechny společně, ale rozdělit je po regionech.
	- U stříleček změnit mechaniky zbraní (Hitscan na Projectile)

## 2. Hlavní pojmy

- **Tick/snapshot:** hra jako sekvence diskrétních kroků, tedy logický čas.
- **UDP vs. TCP:** proč hry často posílají „zastaralá" data raději než čekat na doručení.
- **Konzistence vs. odezva:** kompromis.
- **Model chyb:** ztráta paketu, pád klienta, cheater. Byzantská chyba.
- ⟶ **Reference:** přednáška 7 (klasifikace chyb a modelů), přednáška 5 (komunikace).

# Přístupy
## 3. Deterministický lockstep a P2P

- **Hry:** Age of Empires, StarCraft, Warcraft III.
- **Princip:** neposílá se stav, ale jen vstupy hráčů. Každý klient simuluje stejný svět deterministicky.
- **Nevýhoda:** všichni čekají na nejpomalejšího hráče. Vyžaduje to přesnou deterministiku (floating point, seed náhodných čísel) a desynchronizace jsou tvrdá chyba („Out of Sync").
- **Výhoda:** minimální provoz (tisíce jednotek se synchronizují pár bajty).
- ⟶ **Reference:** **bariéra** (každý tick je bariérou, kterou musí dosáhnout všichni). Zmínit i Lamportovo uspořádání událostí.

## 4. Rollback netcode

- **Hry:** Street Fighter 6, Skullgirls, Mortal Kombat, knihovna GGPO.
- **Myšlenka:** místo čekání na soupeřův vstup ho odhadnu. Když se mýlím, vrátím stav a přepočítám.
- **Porovnání:** _delay-based_ (čekání) vs. _rollback_ (optimismus), tedy pesimistický vs. optimistický přístup.
- ⟶ **Reference:** optimistické vs. pesimistické řízení souběžnosti. Pokud se mýlím, je to podobné spekulativnímu provedení a odvolání (rollback transakce).

# Architektury
## 5. Autoritativní klient-server

- **Hry:** Counter-Strike, Valorant, Overwatch, Fortnite, Quake (historicky).
- **Server je pravda:** klient jen posílá vstupy a zobrazuje.
- **Triky na skrytí latence:**
    - _Client-side prediction_ (ihned se hýbu)
    - _Server reconciliation_ (oprava, když se server liší)
    - _Entity interpolation_ (ostatní hráče zobrazuji mírně v minulosti)
    - _Lag compensation_ (server „přetočí čas" při vyhodnocení výstřelu)
- Vysvětlení peeker's advantage z úvodu.
- **Tick rate:** 128 tick (Valorant) vs. nižší (hry s menším rozpočtem).
- **Bonus:** výhoda proti cheaterům (klient nemá celý stav; _fog of war_ na serveru).
- ⟶ **Reference:** server = leader/koordinátor, klient = replika. Zmínit slabinu jedné centrální instance (single point of failure). Zmínit logické hodiny jako _tick_ a vztah k Lamportově logickému času.

## 6. Hostitel mezi hráči a migrace

- **Hry:** Left 4 Dead, Among Us, Minecraft (LAN), starší díly Call of Duty, R.E.P.O.
- **Princip:** jeden hráč je zároveň server. Je levné, ale hostitel má výhodu v latenci a může podvádět.
- **Zajímavost:** co se stane, když hostitel odejde? _Host migration_ — musí se zvolit nový hostitel.
- ⟶ **Reference:** **volba leadera** (přednáška 7: Bully, Raft). Dá se srovnat: hry často volí podle nejlepšího spojení nebo hardwaru, ne podle ID. Také poukázat, že to není tolerantní k chybám v tom smyslu jako Raft (konsenzus).

## 7. MMO: shardy, jeden svět a Time Dilation

- **WoW, Guild Wars 2:** _realm shardy_ a megaservery (rozdělení hráčů na nezávislé instance).
- **EVE Online:** jeden svět, ale herní svět rozdělen na uzly. Při obrovských bitvách (např. Battle of B-R5RB) se zapíná **Time Dilation**: čas ve hře se zpomalí, aby server stihl zpracovat všechno. Skvělá ukázka, že _čas lze obětovat místo konzistence_.
- **Star Citizen / server meshing:** dynamické přidělování serverů oblastem světa (ověřit aktuální stav, hra je ve vývoji).
- **Interest management:** každý hráč dostává jen data o svém okolí. Vyhýbá se to posílání všeho všem.
- ⟶ **Reference:** přednáška 5 (architektura DS), škálování, **vzájemné vyloučení** (obchod mezi dvěma hráči, nesmí se duplikovat předměty, tedy transakce a zámky). Zmínit _Chord/DHT_ (přednáška 7) jako možný způsob rozdělení světa.

## 8. Synchronizace uvnitř jedné hry

- **Herní engine** je paralelní systém: vlákno logiky, vlákno vykreslování, síťové vlákno, zvuk.
- **Konkrétně:** _double buffering_ stavu, fronty příkazů (producent–konzument), job systémy a thread pooly, bariéry mezi fázemi snímku.
- **Příklad:** Naughty Dog a jejich fiber-based job system (GDC přednáška).
- ⟶ **Reference:** přednášky 2–4 (kritická sekce, semafory, monitory, ThreadPool, klasické problémy). Tohle je část, kde je „učivo" nejpřímější. Pokud vám bude chybět čas, zkraťte ji.

## 9. Závěr

- Návrat k peeker's advantage: co přesně se stalo za tím rohem.
- Srovnávací tabulka: architektura × zdroj pravdy × latence × odolnost vůči cheatům × škálování × příklady.
- Slide s potenciálními otázkami.