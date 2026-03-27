# x86-64 regiszterek — részletes magyarázat

> Kiegészítő anyag a nap-01 tananyaghoz

---

## 1. Mi a regiszter, és miért fontos?

Képzeld el a CPU-t mint egy szakácsot. A **RAM** a kamra — nagy, sok fér bele, de odamenni időbe telik. A **regiszterek** pedig a pult a kezed alatt — nagyon kicsi (16 db 64-bites „rekesz"), de azonnal eléred.

Minden művelet, amit a CPU végez (összeadás, összehasonlítás, adatmozgatás), **regisztereken** történik. Ha a RAM-ból akarsz összeadni két számot, először be kell töltened őket regiszterekbe, ott összeadod, majd az eredményt visszaírhatod a RAM-ba.

**Sebesség összehasonlítás:**

| Tárhely | Hozzáférési idő | Méret |
|---|---|---|
| Regiszter | ~0.3 ns (1 CPU ciklus) | 16 × 8 byte = 128 byte |
| L1 cache | ~1 ns | ~64 KB |
| L2 cache | ~4 ns | ~256 KB |
| RAM | ~100 ns | GB-ok |

Szóval a regiszter **300×-szor** gyorsabb, mint a RAM. Ez az oka annak, hogy az egész CPU-design arról szól: hogyan tartsuk a fontos adatot regiszterekben.

---

## 2. A „matrjoska baba" — regiszterrészek

Ez a tananyag legtrükkösebb része. Az x86-64 architektúra **visszafelé kompatibilis** az 1978-as 8086-os processzorral. Ezért minden regiszter tartalmaz régebbi, kisebb „alregisztereket" — mint egy matrjoska baba:

```
 Bitek:  63 .................. 32  31 ............. 16  15 ... 8  7 ... 0
         ┌─────────────────────────────────────────────────────────────┐
   RAX = │  felső 32 bit          │            EAX                     │  ← 64 bit (x86-64, 2003-)
         │  (csak RAX-ból érhető) ├────────────────────────────────────┤
         │                        │   felső 16 bit   │      AX         │  ← 32 bit (386, 1985-)
         │                        │   (csak EAX-ból) ├────────┬────────┤
         │                        │                  │   AH   │   AL   │  ← 16 bit (8086, 1978-)
         └────────────────────────┴──────────────────┴────────┴────────┘
                                                         8 bit   8 bit
```

**Mit jelent ez gyakorlatban?**

Mind ugyanarra a fizikai tárhelyre mutatnak — nem másolatok! Ha `AL`-t módosítod, az `AX`, `EAX` és `RAX` értéke is megváltozik, mert `AL` az `RAX` legalsó 8 bitje.

**Konkrét példa:**

```
RAX = 0xDEADBEEF_CAFEBABE

Ezt így bontjuk szét:
RAX =  DE AD BE EF CA FE BA BE      ← mind a 8 bájt (64 bit)
EAX =              CA FE BA BE      ← alsó 4 bájt (32 bit)
 AX =                    BA BE      ← alsó 2 bájt (16 bit)
 AH =                    BA         ← a 2. bájt alulról (bit 15-8)
 AL =                       BE      ← a legalsó bájt (bit 7-0)
```

**Java analógia:** Nincs igazi párhuzam, de képzeld el mintha egy `long` változóból bármikor ki tudnád olvasni az alsó `int`-et, alsó `short`-ot, vagy alsó `byte`-ot — **cast nélkül, azonnal, zero cost**.

---

## 3. Az elnevezési rendszer

Két generáció regiszter van, és az elnevezésük eltér:

### Régi regiszterek (8086-ból örökölve): `rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`

| Prefix/Suffix | Méret | Példa | Eredet |
|---|---|---|---|
| `r__` | 64-bit | `rax` | **R**egiszter (x86-64 kiterjesztés) |
| `e__` | 32-bit | `eax` | **E**xtended (386-os kiterjesztés) |
| `__` (semmi) | 16-bit | `ax` | Eredeti 8086 név |
| `__l` | 8-bit low | `al` | **L**ow byte |
| `__h` | 8-bit high | `ah` | **H**igh byte (bit 15-8) |

### Új regiszterek (x86-64-ben jöttek): `r8` – `r15`

| Suffix | Méret | Példa |
|---|---|---|
| (semmi) | 64-bit | `r8` |
| `d` | 32-bit | `r8d` (**D**ouble word) |
| `w` | 16-bit | `r8w` (**W**ord) |
| `b` | 8-bit | `r8b` (**B**yte) |

> **⚠️ Figyelem:** Az `r8`–`r15` regisztereknél **nincs "high byte"** (`r8h` nem létezik). Ez egy egyszerűsítés az AMD részéről.

---

## 4. A 32-bites írás csapdája (NAGYON fontos!)

Ez az egyik leggyakoribb hiba Assembly-ben:

```nasm
; Kiinduló állapot
mov rax, 0xDEADBEEF_CAFEBABE    ; RAX = 0xDEADBEEFCAFEBABE

; 32-bites írás: NULLÁZZA a felső 32 bitet!
mov eax, 0x11111111              ; RAX = 0x0000000011111111  (!!!)
                                 ; A felső DEADBEEF eltűnt!

; 16-bites írás: NEM nulláz semmit
mov rax, 0xDEADBEEF_CAFEBABE    ; visszaállítjuk
mov ax, 0x2222                   ; RAX = 0xDEADBEEFCAFE2222
                                 ; Csak az alsó 16 bit változott

; 8-bites írás: szintén NEM nulláz
mov al, 0x33                     ; RAX = 0xDEADBEEFCAFE2233
                                 ; Csak az alsó 8 bit változott
```

**Miért van ez?** Az AMD úgy tervezte az x86-64-et, hogy a 32-bites nullázás segítse a processzor belső optimalizálásait (eliminál egy „partial register dependency"-t). Ez teljesítménynövelő döntés volt, de meg kell szokni.

**Praktikus szabály:** Ha egy regiszter teljes 64 bitjét nullázni akarod, és utána egy kis értéket akarsz beleírni:

```nasm
mov eax, 42    ; RAX = 0x0000000000000002A  — automatikusan tiszta!
; Ez 1 bájttal rövidebb utasítás, mint: mov rax, 42
```

---

## 5. A regiszterek „személyisége" — hagyományos szerepek

Bár elméletben mind „általános célú", a gyakorlatban minden regiszternek megvan a szerepe:

### 🔴 `RAX` — Accumulator (az „eredmény-regiszter")

- Függvények visszatérési értéke IDE kerül (System V ABI)
- `mul`/`div` utasítások implicit operandusa
- Java analógia: `return` értéke

### 🟢 `RCX` — Counter (a „számláló")

- `loop` utasítás ezt használja
- `shl`/`shr` (shift) utasítások `cl`-t használják shift-mennyiségként
- `rep` prefix (string műveletek ismétlése) ezt számolja

### 🔵 `RSI` / `RDI` — Source / Destination Index

- String műveletek (`movsb`, `cmpsb`, `stosb`) ezeket használják
- Linux System V ABI-ban: `rdi` = 1. arg, `rsi` = 2. arg
- Gondolj rá úgy: `memcpy(rdi, rsi, rcx)` — cél, forrás, darabszám

### 🟡 `RSP` — Stack Pointer (SOHA NE PISZKÁLD közvetlenül!)

- Mindig a stack tetejére mutat
- `push`/`pop`/`call`/`ret` automatikusan módosítja
- Ha elrontod, a programod azonnal összeomlik (segfault)

### 🟠 `RBP` — Base Pointer (stack frame alap)

- Függvényhívásoknál a stack frame elejét jelöli
- Lokális változók és paraméterek `rbp`-hez képest érhetők el
- Debugger ebből olvassa ki a call stack-et

### ⚪ `RIP` — Instruction Pointer (nem GPR!)

- Mindig a **következő** utasítás címét tartalmazza
- NEM írható `mov`-val! Csak `jmp`, `call`, `ret` módosítja
- Ha GDB-ben `rip`-et látsz → ott tart a program végrehajtása

---

## 6. Caller-saved vs. Callee-saved — ki menti el?

Ez a System V AMD64 ABI (Linux/macOS hívási konvenció) szabálya:

| Típus | Regiszterek | Mit jelent? |
|---|---|---|
| **Caller-saved** (volatile) | `rax, rcx, rdx, rsi, rdi, r8-r11` | Ha hívó fél használja ezeket, **maga mentse el** hívás előtt, mert a meghívott függvény szabadon felülírhatja |
| **Callee-saved** (non-volatile) | `rbx, rbp, r12-r15` | A meghívott függvény **köteles visszaállítani** ezeket, ha módosítja |

**Java analógia:** Ez olyan, mint egy metódushívás „szerződése". Ha egy Java metódust hívsz, nem kell aggódnod, hogy a lokális változóid megváltoznak — a JVM elrejti ezt. Assembly-ben **te vagy felelős** azért, hogy a megfelelő regisztereket elmentsd a stack-re.

**Gyakorlati példa:**

```nasm
; Te használod rbx-et, és meghívsz egy másik függvényt
push rbx              ; elmentjük rbx-et (callee-saved, kötelező)
mov rbx, 42           ; használjuk
call some_function    ; some_function NEM fogja rbx-et elrontani
                      ; (mert ő is betartja az ABI-t)
pop rbx               ; visszaállítjuk
```

---

## 7. RFLAGS — a „státuszjelző tábla"

Az RFLAGS nem „normál" regiszter — ez egy bitenként értelmezett jelzőtábla. Minden aritmetikai/logikai művelet **automatikusan** beállítja a megfelelő biteket:

```nasm
mov rax, 5
sub rax, 5        ; RAX = 0
                  ; → ZF = 1 (eredmény nulla)
                  ; → SF = 0 (eredmény nem negatív)
                  ; → CF = 0 (nem volt átvitel)

mov rax, 0
sub rax, 1        ; RAX = 0xFFFFFFFFFFFFFFFF  (= -1 signed, vagy max unsigned)
                  ; → ZF = 0 (nem nulla)
                  ; → SF = 1 (MSB = 1, tehát "negatív")
                  ; → CF = 1 (unsigned alulcsordulás: 0-ból vontunk ki)
                  ; → OF = 0 (signed: 0 - 1 = -1, ez valid signed eredmény)
```

**Miért fontos?** A feltételes ugróutasítások (`je`, `jne`, `jg`, `jl`, stb.) ezeket a flag-eket olvassák. Ez az `if` és a `for` loop mechanizmusa Assembly szinten:

```nasm
cmp rax, rbx      ; belül: rax - rbx, eredmény eldobva, DE flag-ek beállítva
je equal_label    ; ha ZF=1 (tehát rax == rbx), ugorj
```

---

## Összefoglaló vizuális áttekintés

```
┌───────────────────────────────────────────────────────────┐
│  ÁLTALÁNOS CÉLÚ REGISZTEREK (GPR) — mind 64 bit           │
│                                                           │
│  Visszatérési érték:  RAX                                 │
│  Függvény arg-ok:     RDI, RSI, RDX, RCX, R8, R9          │
│  Caller-saved:        RAX, RCX, RDX, RSI, RDI, R8-R11     │
│  Callee-saved:        RBX, RBP, R12-R15                   │
│  Stack pointer:       RSP  ⚠️ ne nyúlj hozzá kézzel       │
│  Base pointer:        RBP                                 │
├───────────────────────────────────────────────────────────┤
│  SPECIÁLIS REGISZTEREK                                    │
│  Utasításmutató:      RIP  (csak jmp/call/ret módosítja)  │
│  Jelzőbitek:          RFLAGS  (ZF, CF, SF, OF, PF...)     │
│  Szegmens:            CS, DS, SS, ES, FS, GS (ignoráld)   │
└───────────────────────────────────────────────────────────┘
```
