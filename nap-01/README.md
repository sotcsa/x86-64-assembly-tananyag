# 1. nap: Alapok – CPU architektúra, regiszterek, számrendszerek

## Mit tanulunk ma?

Ez a fejezet az x86-64 Assembly tanulás fundamentuma. Nem írunk még futó kódot – ehelyett megértjük azt a mentális modellt, amire az összes többi nap épül. Megismerjük, hogyan látja a CPU a világot, hogyan tárol adatot, és miért fontos a hexadecimális számrendszer. Egy senior Java/DevOps mérnök számára ez a nap a „tudatosítás" napja: sok mindent már tudsz, csak más szinten.

**Tervezett idő:** ~60-90 perc olvasás + gyakorlás

---

## Tartalomjegyzék

1. [Miért tanuljunk Assembly-t?](#1-miért-tanuljunk-assembly-t)
2. [A számítógép felépítése Assembly szemszögből](#2-a-számítógép-felépítése-assembly-szemszögből)
3. [Számrendszerek](#3-számrendszerek)
4. [x86-64 regiszterek teljes áttekintése](#4-x86-64-regiszterek-teljes-áttekintése)
   - [Részletes magyarázat →](regiszterek-reszletes.md)
5. [Adattípusok és méretek](#5-adattípusok-és-méretek)
6. [Gyakorlatok](#6-gyakorlatok)
7. [Ellenőrizd magad](#7-ellenőrizd-magad)

---

## 1. Miért tanuljunk Assembly-t?

### Egy senior mérnök perspektívájából

Ha 22 éve Java-ban és DevOps-ban dolgozol, joggal merül fel a kérdés: *miért szükséges ez nekem?* Assembly-t napi szinten valóban nem fogsz írni – de érteni fogod, ami körülötted történik. Ez az a szint, ahol a magas szintű absztrakciók leleplez(őd)nek.

### Mikor találkozol Assembly-vel a gyakorlatban?

**JVM JIT-fordító (Just-In-Time compiler)**
A JVM a bytecode-ot futásidőben natív gépi kódra fordítja. Ha valaha nézted a `-XX:+PrintAssembly` kimenetét, assembly kódot olvastál – csak nem tudtad értelmezni. Ha tudod, akkor megérted, miért gyorsabb egy `int` összehasonlítás, mint egy `Integer` összehasonlítás a JIT által generált kódban.

```bash
# JVM assembly kimenet engedélyezése (hsdis plugin szükséges)
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:+LogCompilation MyApp
```

**Core dump elemzés (core dump analysis)**
Kubernetes podod OOMKilled-del halt meg, és van egy core dumpod. A GDB stack trace-ben assembly sorok és regiszterértékek jelennek meg. Ha érted a `rsp`, `rbp`, `rip` regisztereket, tudod, hol tartott a program a halál pillanatában.

**Biztonsági kutatás (security research)**
CVE-k elemzése, buffer overflow megértése, ROP (Return-Oriented Programming) láncok felismerése – mind assembly szintű tudást igényelnek. A `objdump -d` kimenete assembly.

**Teljesítmény-profilozás (performance profiling)**
A `perf annotate` parancs assembly szinten mutatja, melyik utasítás fogyaszt legtöbb CPU ciklust. SIMD optimalizálások (SSE, AVX) megértése assembly nélkül lehetetlen.

**Fordítóprogramok és linkerek vizsgálata**
Miért inline-olja a GCC ezt a függvényt? Miért generál a JIT ennyi memória-olvasást? Csak assembly szinten látszik.

### Absztrakciós szintek – hol van az Assembly?

```
+-----------------------------------+
|  Java / Python / Go forráskód     |  ← Te itt dolgozol naponta
+-----------------------------------+
|  JVM bytecode / IR reprezentáció  |  ← Fordítóprogram outputja
+-----------------------------------+
|  x86-64 Assembly (mnemonics)      |  ← ITT TARTUNK
+-----------------------------------+
|  Gépi kód (bináris, opcodes)      |  ← CPU ezt "érti"
+-----------------------------------+
|  Mikroarchitektúra (µops)         |  ← CPU belső optimalizálás
+-----------------------------------+
|  Tranzisztorok / elektronika      |  ← Hardver
+-----------------------------------+
```

> **💡 Tipp:** Az Assembly nem a legalacsonyabb szint – de ez az utolsó szint, ahol a kód még emberileg olvasható. Az Assembly mnemonikokat (pl. `mov`, `add`) az assembler fordítja bináris gépi kóddá (opcodes).

---

## 2. A számítógép felépítése Assembly szemszögből

### Egyszerű mentális modell

Assembly programozáshoz elég egy leegyszerűsített modell. Nem kell mikroarchitektúrális részletek – csak a következő komponensek:

```
┌─────────────────────────────────────────────────────┐
│                        CPU                          │
│  ┌──────────────┐    ┌──────────────────────────┐   │
│  │  Regiszterek │    │       ALU / FPU          │   │
│  │  (registers) │    │  (aritmetika, logika)    │   │
│  │  ~16 × 64bit │    └──────────────────────────┘   │
│  └──────────────┘    ┌──────────────────────────┐   │
│  ┌──────────────┐    │   Vezérlőegység (CU)     │   │
│  │  Cache       │    │   (Fetch-Decode-Execute) │   │
│  │  L1/L2/L3    │    └──────────────────────────┘   │
│  └──────────────┘                                   │
└────────────────────────┬────────────────────────────┘
                         │  Busz (bus)
         ┌───────────────┼───────────────┐
         │               │               │
   ┌─────┴─────┐   ┌─────┴─────┐   ┌────┴──────┐
   │  Memória  │   │    I/O    │   │ Merevlemez│
   │   (RAM)   │   │(billentyű,│   │ (storage) │
   │  byte-ok  │   │ képernyő) │   │           │
   └───────────┘   └───────────┘   └───────────┘
```

**CPU (Central Processing Unit):** Utasításokat hajt végre. Tartalmaz regisztereket (kis, ultra-gyors tárolók), az ALU-t (aritmetikai-logikai egység), és a vezérlőegységet.

**Memória (RAM):** Bájtok lineáris sorozata. Minden bájtnak van egy **memóriacíme** (address), 0-tól kezdve. Egy tipikus folyamat gigabájtokat láthat.

**Busz (bus):** Az adatok "útja" a komponensek között. Az Assembly programozó általában nem foglalkozik a busszal közvetlenül.

### Von Neumann architektúra – 2 mondatban

Az adat és a program ugyanabban a memóriában tárolódik, bájtos formában. A CPU szekvenciálisan olvassa a memóriából az utasításokat, végrehajtja őket, és az eredményeket visszaírja a memóriába (vagy regiszterekbe).

### Fetch-Decode-Execute ciklus

Ez a CPU alapritmusának neve. Minden utasítás végrehajtása három fázison megy át:

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  FETCH   │────▶│  DECODE  │────▶│ EXECUTE  │
│  (olvas) │     │ (értel-  │     │ (végrehaj│
│ RIP→mem  │     │  mezi    │     │  t ALU/  │
│          │     │  opcode) │     │  mem)    │
└──────────┘     └──────────┘     └─────┬────┘
     ▲                                   │
     │             RIP += utasítás méret │
     └───────────────────────────────────┘
```

1. **Fetch:** A CPU beolvassa a következő utasítást a memóriából. A `RIP` (Instruction Pointer) regiszter mutatja, melyik memóriacímen van a következő utasítás.
2. **Decode:** A CPU értelmezi az opcode-ot (műveleti kód) – mi a művelet, mik az operandusok.
3. **Execute:** Az ALU elvégzi a számítást, vagy a memória-egység adatot olvas/ír.

Ezután `RIP` automatikusan a következő utasításra mutat, és a ciklus ismétlődik.

### Java analógia: JVM bytecode ↔ gépi kód

| Fogalom | JVM | x86-64 Natív |
|---------|-----|--------------|
| Utasításkészlet | Bytecode (pl. `iload`, `iadd`) | Assembly mnemonik (pl. `mov`, `add`) |
| "CPU" | JVM interpreter/JIT | Intel/AMD processzor |
| Utasításmutató | Program Counter (PC) | `RIP` regiszter |
| Operandus verem | JVM stack | CPU regiszterek + stack |
| Fordítás | `javac` → `.class` fájl | NASM → `.o` → ELF bináris |

> **💡 Tipp:** A JVM bytecode egy *virtuális* CPU utasításkészlete. Amikor a JIT bekapcsol, ezt lefordítja *valódi* x86-64 utasításokra – azzal a különbséggel, hogy a JVM stack-alapú, az x86-64 viszont regiszter-alapú gép.

---

## 3. Számrendszerek

### Miért fontos a bináris és hexadecimális?

A CPU biteken (0 és 1) dolgozik. A regiszterek, memóriacímek, flag-ek mind bináris értékek. A hexadecimális (hex) rendszer a bináris tömör jelölése: **1 hex jegy = 4 bit** (egy "nibble").

```
Bináris:      1111 0000 1010 0011
Hexadecimális:    F    0    A    3   →  0xF0A3
```

Egy 64-bites memóriacím binárisan 64 jegy, hexán 16 jegy – sokkal olvashatóbb.

### Konverziós táblázat: 0–15

| Decimális | Bináris | Hexadecimális |
|-----------|---------|---------------|
| 0         | 0000    | 0x0           |
| 1         | 0001    | 0x1           |
| 2         | 0010    | 0x2           |
| 3         | 0011    | 0x3           |
| 4         | 0100    | 0x4           |
| 5         | 0101    | 0x5           |
| 6         | 0110    | 0x6           |
| 7         | 0111    | 0x7           |
| 8         | 1000    | 0x8           |
| 9         | 1001    | 0x9           |
| 10        | 1010    | 0xA           |
| 11        | 1011    | 0xB           |
| 12        | 1100    | 0xC           |
| 13        | 1101    | 0xD           |
| 14        | 1110    | 0xE           |
| 15        | 1111    | 0xF           |

### Decimális → Bináris konverzió (egész osztásos módszer)

```
42 decimálisból bináris:

42 ÷ 2 = 21, maradék 0   ↑
21 ÷ 2 = 10, maradék 1   ↑  (alulról felfelé olvasva)
10 ÷ 2 =  5, maradék 0   ↑
 5 ÷ 2 =  2, maradék 1   ↑
 2 ÷ 2 =  1, maradék 0   ↑
 1 ÷ 2 =  0, maradék 1   ↑

Eredmény: 101010₂ = 42₁₀
```

### Bináris → Hexadecimális konverzió

Csoportosíts jobbról 4-esével, majd konvertáld táblázatból:

```
101010₂  →  0010 1010₂  →  2   A₁₆  →  0x2A
```

### Bináris → Decimális (helyiérték módszer)

```
1010 1010₂

Bit pozíciók (jobbról, 0-tól):
1×2⁷ + 0×2⁶ + 1×2⁵ + 0×2⁴ + 1×2³ + 0×2² + 1×2¹ + 0×2⁰
= 128  +  0   +  32  +  0   +  8   +  0   +  2   +  0
= 170₁₀
```

### Kettes komplemens (Two's Complement) – előjeles számok

Az x86-64 Assembly az előjeles egész számokat **kettes komplemens** (two's complement) formában tárolja. Ez az a módszer, amit a Java `int`, `long`, `byte` típusok is használnak.

**Hogyan működik?**

Egy `n` bites szám tartománya:
- **Előjel nélkül (unsigned):** 0-tól 2ⁿ−1-ig
- **Előjeles (signed, kettes komplemens):** −2ⁿ⁻¹-től 2ⁿ⁻¹−1-ig

A legmagasabb bit (MSB – Most Significant Bit) az **előjelbit**: 0 = pozitív, 1 = negatív.

**−1 kiszámítása 8 bitre:**
```
Módszer: bitenkénti NOT + 1

+1  =  0000 0001
NOT =  1111 1110
+1  =  1111 1111  →  ez a −1 (0xFF)
```

**8 bites példák:**

| Bináris   | Unsigned értéke | Signed értéke |
|-----------|-----------------|---------------|
| 0000 0000 | 0               | 0             |
| 0111 1111 | 127             | +127          |
| 1000 0000 | 128             | −128          |
| 1111 1111 | 255             | −1            |
| 1111 1110 | 254             | −2            |

> **⚠️ Figyelem:** Assembly-ben te döntöd el, hogy egy bitmintát előjelesként vagy előjel nélküliként értelmezed. A CPU nem tudja – te mondod meg az utasítással (pl. `imul` vs. `mul`, `idiv` vs. `div`). Ez a Java-tól eltérő gondolkodásmód, ahol a típusrendszer ezt eldönti helyetted.

**Java analógia:**

| Java típus | Méret   | Tartomány                    | Assembly ekvivalens |
|------------|---------|------------------------------|---------------------|
| `byte`     | 8 bit   | −128 … +127                  | 8-bites regiszter   |
| `short`    | 16 bit  | −32 768 … +32 767            | 16-bites regiszter  |
| `int`      | 32 bit  | −2 147 483 648 … +2 147 483 647 | 32-bites regiszter |
| `long`     | 64 bit  | −2⁶³ … 2⁶³−1                | 64-bites regiszter  |
| `char`     | 16 bit  | 0 … 65 535 (unsigned!)       | 16-bites regiszter  |

---

## 4. x86-64 regiszterek teljes áttekintése

### Mi a regiszter?

A regiszter a CPU-ban lévő ultra-gyors tárhely. Hozzáférési ideje ~0 ns (a cache-nél is gyorsabb, fizikailag a CPU-ban van). Az x86-64 CPU-nak 16 általános célú regisztere (general-purpose register – GPR) van, mindegyike 64 bites.

> **💡 Tipp:** Java analógia: a regiszterek olyanok, mint a lokális változók a JVM stack frame-jében – de a JVM-nek "végtelen" virtuális regisztere van, míg a valódi CPU-nak csak 16 GPR-je.

### Az általános célú regiszterek és részeik

Minden 64-bites regiszternek elérhetők kisebb részei visszafelé kompatibilitás miatt:

```
 63                    31            15      8  7      0
 ┌─────────────────────┬─────────────┬───────┬────────┐
 │                     │             │  AH   │   AL   │  ← 8-bites részek
 │                     │     AX      ├───────┴────────┤  ← 16-bit (AX)
 │                     │           EAX                │  ← 32-bit (EAX)
 │                    RAX                             │  ← 64-bit (RAX)
 └─────────────────────┴─────────────┴───────┴────────┘
```

### Teljes regiszter táblázat

| 64-bit | 32-bit | 16-bit | 8-bit (low) | 8-bit (high) | Hagyományos szerep |
|--------|--------|--------|-------------|--------------|-------------------|
| `rax`  | `eax`  | `ax`   | `al`        | `ah`         | Accumulator – visszatérési érték, szorzás/osztás |
| `rbx`  | `ebx`  | `bx`   | `bl`        | `bh`         | Base – callee-saved, általános célú |
| `rcx`  | `ecx`  | `cx`   | `cl`        | `ch`         | Counter – ciklusszámláló, shift műveletek |
| `rdx`  | `edx`  | `dx`   | `dl`        | `dh`         | Data – szorzás/osztás maradék, 3. syscall arg |
| `rsi`  | `esi`  | `si`   | `sil`       | –            | Source Index – string műveletek forrása, 2. arg |
| `rdi`  | `edi`  | `di`   | `dil`       | –            | Destination Index – string műveletek célja, 1. arg |
| `rbp`  | `ebp`  | `bp`   | `bpl`       | –            | Base Pointer – stack frame alap |
| `rsp`  | `esp`  | `sp`   | `spl`       | –            | Stack Pointer – stack teteje (mindig fenntartott!) |
| `r8`   | `r8d`  | `r8w`  | `r8b`       | –            | 5. syscall / függvényargumentum |
| `r9`   | `r9d`  | `r9w`  | `r9b`       | –            | 6. syscall / függvényargumentum |
| `r10`  | `r10d` | `r10w` | `r10b`      | –            | 4. syscall argumentum |
| `r11`  | `r11d` | `r11w` | `r11b`      | –            | Caller-saved, általános célú |
| `r12`  | `r12d` | `r12w` | `r12b`      | –            | Callee-saved, általános célú |
| `r13`  | `r13d` | `r13w` | `r13b`      | –            | Callee-saved, általános célú |
| `r14`  | `r14d` | `r14w` | `r14b`      | –            | Callee-saved, általános célú |
| `r15`  | `r15d` | `r15w` | `r15b`      | –            | Callee-saved, általános célú |

> **⚠️ Figyelem:** Az `ah`, `bh`, `ch`, `dh` (high byte) regiszterek csak a 4 régi regiszternél léteznek (`rax`, `rbx`, `rcx`, `rdx`). Az újabb `r8`–`r15` regisztereknél nincs "high byte" elérési mód.

### Fontos részlet: 32-bites írás nullázza a felső 32 bitet

```nasm
; Ha rax = 0xDEADBEEF_CAFEBABE
mov eax, 0x12345678    ; rax = 0x00000000_12345678 (!!)
                       ; A felső 32 bit automatikusan nullázódik!

; Ezzel szemben 16-bites vagy 8-bites írás NEM nullázza a magas biteket:
mov ax, 0x1234         ; rax felső 48 bitje változatlan marad
```

> **💡 Tipp:** Ez egy x86-64 tervezési döntés a kódméret csökkentéséért. Praktikus következmény: ha egy regiszter felső részeit nullázni akarod, egyszerűen a 32-bites verziójába írj.

### Speciális regiszterek

**`RIP` – Instruction Pointer (utasításmutató)**
Mindig a *következő* végrehajtandó utasítás memóriacímét tartalmazza. Közvetlenül nem lehet módosítani (nem írható `mov`-val) – csak `jmp`, `call`, `ret` utasítások módosítják.

```
Aktuális RIP értéke = a következő utasítás címe a memóriában
```

**`RFLAGS` – Flag regiszter (jelzőbitek)**
64 bites regiszter, amelynek egyes bitjei az utolsó aritmetikai/logikai művelet eredményét jelzik. A legfontosabb flag-ek:

| Flag | Bit | Neve | Mikor aktív (1)? |
|------|-----|------|-----------------|
| CF   | 0   | Carry Flag | Előjel nélküli átvitel/alulcsordulás |
| ZF   | 6   | Zero Flag | Eredmény = 0 |
| SF   | 7   | Sign Flag | Eredmény negatív (MSB = 1) |
| OF   | 11  | Overflow Flag | Előjeles túlcsordulás |
| PF   | 2   | Parity Flag | Az eredmény legkisebb bájtjában páros számú 1-es bit |

Ezeket a flag-eket a 4. napon (feltételes ugróutasítások) fogjuk részletesen használni.

### Szegmens regiszterek (rövid megemlítés)

A `CS`, `DS`, `ES`, `FS`, `GS`, `SS` szegmens regiszterek (segment registers) a régi 16-bites szegmentált memóriamodell maradványa. 64-bites módban (long mode) szinte teljesen figyelmen kívül hagyhatók – az operációs rendszer kezeli őket. Kivétel: az `FS` és `GS` regisztereket a Linux kernel és a szálkezelő könyvtárak (pl. `pthread`) thread-local storage (TLS) eléréséhez használják.

---

## 5. Adattípusok és méretek

### Alap méretek x86-64 Assembly-ben

| Név     | Méret   | Bitek | NASM kulcsszó | Java megfelelő |
|---------|---------|-------|---------------|----------------|
| byte    | 1 bájt  | 8 bit | `db` (define byte)  | `byte`  |
| word    | 2 bájt  | 16 bit| `dw` (define word)  | `short` |
| dword   | 4 bájt  | 32 bit| `dd` (define dword) | `int`   |
| qword   | 8 bájt  | 64 bit| `dq` (define qword) | `long`  |

> **💡 Tipp:** Az "word" (szó) x86 hagyományosan 16 bitet jelent – ez a 8086-os processzorból ered. A "dword" = double word (32 bit), a "qword" = quad word (64 bit). Ne keverd össze más architektúrák "word" definícióival!

### NASM adatdeklarációk az adatszegmensben

```nasm
section .data
    my_byte   db  42          ; 1 bájt, értéke 42 (0x2A)
    my_word   dw  1000        ; 2 bájt, értéke 1000 (0x03E8)
    my_dword  dd  100000      ; 4 bájt, értéke 100 000
    my_qword  dq  1234567890  ; 8 bájt, nagy szám

    ; Karakterek és stringek
    my_char   db  'A'         ; ASCII kód: 65 (0x41)
    my_str    db  "Hello", 0  ; null-terminált string (C stílus)

    ; Negatív szám (kettes komplemens automatikusan)
    my_neg    db  -1          ; binárisan: 1111 1111 (0xFF)

    ; Hexadecimális literál NASM-ban
    my_hex    dd  0xDEADBEEF  ; 4 bájt hex literál
```

### Inicializálatlan adatok (BSS szegmens)

```nasm
section .bss
    buffer    resb  64   ; 64 bájtot foglal le (reserve bytes)
    count     resw  1    ; 1 word (2 bájt) helyet foglal
    value     resd  1    ; 1 dword (4 bájt) helyet foglal
    result    resq  1    ; 1 qword (8 bájt) helyet foglal
```

### Memóriaelrendezés – kis endian (little-endian)

Az x86-64 **little-endian** architektúra: a legkisebb helyiértékű bájt tárolódik a legalacsonyabb memóriacímen.

```
dd 0x12345678 memóriában (4 bájton):

Cím:    0x1000  0x1001  0x1002  0x1003
Érték:   0x78    0x56    0x34    0x12
          ↑               ↑
     legkisebb bájt   legnagyobb bájt
```

> **⚠️ Figyelem:** A little-endian elrendezés meglepetést okozhat, ha hálózati protokollokon (amelyek big-endian / network byte order formátumot használnak) érkező adatot dolgozol fel. A `ntohl()`, `ntohs()` C függvények pontosan ezt a konverziót végzik. Java-ban a `ByteBuffer.order(ByteOrder.LITTLE_ENDIAN)` analógia.

---

## 6. Gyakorlatok

### 6.1 feladat – Számrendszer konverziók (kézzel)

Konvertáld az alábbi számokat a megadott számrendszerbe (számológép nélkül):

| Decimális | Bináris | Hexadecimális |
|-----------|---------|---------------|
| 255       | ?       | ?             |
| ?         | 0110 0100 | ?           |
| ?         | ?       | 0xFF          |
| 42        | ?       | ?             |
| ?         | 1000 0000 | ?           |

### 6.2 feladat – Kettes komplemens

1. Ábrázold a következő számokat **8 biten**, kettes komplemensben:
   - +50
   - −50
   - −128
   - −1

2. Mi a decimális értéke az alábbi 8-bites kettes komplementes számoknak?
   - `1001 0110`
   - `0111 1111`
   - `1000 0001`

### 6.3 feladat – Regiszter méretek és részek

Az `rax` regiszter értéke jelenleg: `0xDEADBEEF_CAFEBABE`

Mi az értéke az alábbi "alregisztereknek"?

| Regiszter | Méret  | Érték (hex) |
|-----------|--------|-------------|
| `rax`     | 64 bit | ?           |
| `eax`     | 32 bit | ?           |
| `ax`      | 16 bit | ?           |
| `al`      | 8 bit  | ?           |
| `ah`      | 8 bit  | ?           |

### 6.4 feladat – Adatméretek és NASM deklarációk

Melyik NASM adatdeklaráció (`db`, `dw`, `dd`, `dq`) a megfelelő az alábbi Java típusokhoz? Adjunk meg egy-egy példa deklarációt!

| Java kód | NASM deklaráció | Megjegyzés |
|----------|-----------------|------------|
| `byte x = 127;` | ? | |
| `short y = -1000;` | ? | |
| `int z = 0x1234ABCD;` | ? | |
| `long w = 1_000_000_000_000L;` | ? | |

---

## Megoldások

### 6.1 megoldás

| Decimális | Bináris       | Hexadecimális |
|-----------|---------------|---------------|
| 255       | 1111 1111     | 0xFF          |
| 100       | 0110 0100     | 0x64          |
| 255       | 1111 1111     | 0xFF          |
| 42        | 0010 1010     | 0x2A          |
| 128       | 1000 0000     | 0x80          |

### 6.2 megoldás

**8-bites kettes komplemens ábrázolás:**

- +50 → `0011 0010`
- −50 → +50 NOT-ja + 1 = `1100 1101` + 1 = `1100 1110`
- −128 → `1000 0000` (a 8 bites minimum)
- −1 → `1111 1111`

**Decimális értékek:**

- `1001 0110` → MSB=1, tehát negatív. Komplemens: `0110 1001` + 1 = `0110 1010` = 106 → érték: **−106**
- `0111 1111` → MSB=0, pozitív: **+127**
- `1000 0001` → MSB=1, negatív. Komplemens: `0111 1110` + 1 = `0111 1111` = 127 → érték: **−127**

### 6.3 megoldás

`rax = 0xDEADBEEF_CAFEBABE`

| Regiszter | Méret  | Érték (hex)       |
|-----------|--------|-------------------|
| `rax`     | 64 bit | `0xDEADBEEFCAFEBABE` |
| `eax`     | 32 bit | `0xCAFEBABE`      |
| `ax`      | 16 bit | `0xBABE`          |
| `al`      | 8 bit  | `0xBE`            |
| `ah`      | 8 bit  | `0xBA`            |

### 6.4 megoldás

| Java kód | NASM deklaráció | Megjegyzés |
|----------|-----------------|------------|
| `byte x = 127;` | `db 127` | 1 bájt |
| `short y = -1000;` | `dw -1000` | NASM kezeli a kettes komplementet |
| `int z = 0x1234ABCD;` | `dd 0x1234ABCD` | 4 bájt, little-endian sorrendben tárolva |
| `long w = 1_000_000_000_000L;` | `dq 1000000000000` | 8 bájt |

---

## 7. Ellenőrizd magad

1. **Mi a Fetch-Decode-Execute ciklus három fázisa, és mi történik mindegyikben?**

2. **Miért tároljuk az előjeles számokat kettes komplemens formában, és nem egyszerűen egy előjelbittel?**
   *(Tipp: mi történik, ha 0-t kétféleképpen lehetne ábrázolni?)*

3. **Mi a különbség az `eax`-ba és az `ax`-ba való írás között, ha az `rax` regiszter felső bitjeit tekintjük?**

4. **Egy 64-bites memóriacím hexadecimálisan hány jegyből áll? Miért praktikusabb hex-et használni, mint binárist?**

5. **Mi a little-endian byte-sorrend, és hogyan tárolódik a `0x12345678` érték memóriában 4 bájton?**

6. **Sorolj fel legalább 3 olyan gyakorlati helyzetet, ahol egy senior Java/DevOps mérnök assembly-szintű ismereteket hasznosíthat!**

---

## Összefoglalás

| Témakör | Kulcsfogalmak |
|---------|---------------|
| CPU modell | Regiszterek, ALU, CU, memória, busz |
| Végrehajtás | Fetch-Decode-Execute, RIP |
| Számrendszerek | Bináris, hex, kettes komplemens |
| Regiszterek | 16 GPR, 64/32/16/8-bites részek |
| Speciális reg. | RIP (utasításmutató), RFLAGS (jelzőbitek) |
| Adatméretek | byte=1, word=2, dword=4, qword=8 |
| NASM | `db`, `dw`, `dd`, `dq`, `resb`/`resw`/`resd`/`resq` |

---

## Következő lépés

➡️ **2. nap: Első program – NASM szintaxis, szekciók, Hello World, fordítás/linkelés**

A következő fejezetben megírjuk az első futtatható Assembly programunkat, megismerjük a NASM szintaxist, a `.text`, `.data`, `.bss` szekciók szerepét, és végigmegyünk a teljes fordítási folyamaton.
