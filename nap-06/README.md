# 6. nap – Memóriakezelés és címzési módok

## Mit tanulunk ma?

Ma mélyebbre merülünk a memória világába. Megismerjük az összes x86-64 **címzési módot** (hogyan hivatkozunk memóriacímekre), a **LEA** utasítás erejét, az adatszekciók részleteit, a **mutatók** működését Assembly szinten, és végül megértjük, miért fontos az **endianness**. Ez az egyik legfontosabb nap: a memória kezelése az Assembly programozás lelke.

**Java párhuzam:** Ha Javában azt írod, hogy `array[i]`, a JVM kiszámolja a tényleges memóriacímet (`base + i * elementSize`). x86-64-ben ugyanezt **te** csinálod, explicit módon — és pontosan így is kell megtenni.

---

## Tartalom

1. [Címzési módok](#1-c%C3%ADmz%C3%A9si-m%C3%B3dok)
2. [LEA részletesen](#2-lea-részletesen)
3. [Adatszekciók részletesen](#3-adatszekciók-részletesen)
4. [Mutatók Assembly-ben](#4-mutatók-assembly-ben)
5. [Endianness](#5-endianness)
6. [Példaprogramok](#6-példaprogramok)
7. [Gyakorlatok](#7-gyakorlatok)
8. [Ellenőrizd magad](#8-ellenőrizd-magad)

---

## 1. Címzési módok

Az x86-64 architektúra több különböző módot kínál arra, hogyan adjuk meg egy utasítás operandusát. Ezeket **addressing mode**-oknak (címzési módoknak) nevezzük.

### 1.1 Immediate (közvetlen) – `mov rax, 42`

A szám közvetlenül az utasításban van kódolva (nem memóriából jön).

```nasm
mov rax, 42          ; rax = 42
mov rbx, 0xFF        ; rbx = 255 (hexadecimális)
mov rcx, 0b1010      ; rcx = 10  (bináris, NASM-ban: 0b prefix)
```

> **💡 Tipp:** Az immediate érték az utasítás gépi kódjába van beégetve. Gyors, de nem változtatható futás közben.

### 1.2 Register (regiszter) – `mov rax, rbx`

Az operandus egy másik regiszterből jön. Ez a leggyorsabb módszer.

```nasm
mov rax, rbx         ; rax = rbx tartalma
mov rcx, rax         ; rcx = rax tartalma
add rdi, rsi         ; rdi += rsi
```

### 1.3 Direct / Absolute (közvetlen memória) – `mov rax, [label]`

A szögletes zárójel (`[...]`) azt jelenti: **olvasd ki a memóriából az adott címről**. Ez a dereference operátor Assembly-ben.

```nasm
section .data
    ertek  dq 1234          ; 64-bites egész, értéke 1234

section .text
    mov rax, [ertek]        ; rax = a memóriacímen lévő érték (1234)
    mov rax, ertek          ; rax = maga a CÍM (nem az érték!)
```

> **⚠️ Figyelem:** `mov rax, ertek` a **cím** értékét tölti be, **nem** a memória tartalmát! A dereference-hez mindig `[...]` kell.

### 1.4 Register Indirect (regiszter által mutatott cím) – `mov rax, [rbx]`

A regiszter tartalmát memóriacímként kezeljük.

```nasm
mov rbx, ertek_cime        ; rbx egy memóriacímet tartalmaz
mov rax, [rbx]             ; rax = a rbx által mutatott memória tartalma
```

**Java párhuzam:** `rbx` olyan, mint egy Java referencia. `[rbx]` annyit tesz: "kövesd a referenciát, és olvasd ki az objektumot".

### 1.5 Base + Displacement (alap + eltolás) – `mov rax, [rbx + 8]`

Regiszterhez fix eltolást adunk. Tipikusan struktúra-mezők elérésekor vagy a stacken lévő változók elérésekor használjuk.

```nasm
mov rax, [rbx + 8]          ; rbx által mutatott terület + 8 bájt offset
mov rax, [rbp - 16]         ; stack frame: rbp-16 bájtnál lévő lokális változó
mov rax, [rsp + 24]         ; stack tetejétől 24 bájttal lejjebb
```

### 1.6 Base + Index × Scale + Displacement – `mov rax, [rbx + rcx*8 + 16]`

Ez az x86-64 leggazdagabb (és legfontosabb) címzési módja. Formája:

```
[base + index * scale + displacement]
```

ahol:
- **base**: bármely általános regiszter (alap cím, pl. tömb eleje)
- **index**: bármely regiszter, **kivéve rsp** (az elem sorszáma)
- **scale**: 1, 2, 4 vagy 8 (az elem mérete bájtban)
- **displacement**: előjeles 8 vagy 32 bites konstans (offset)

```nasm
; int64 tömb: long[] arr = {10, 20, 30};
; arr[i] elérése, ahol i = rcx, arr eleje = rbx

mov rax, [rbx + rcx*8]        ; arr[i] — 8 bájtos elemek
mov rax, [rbx + rcx*8 + 16]   ; arr[i+2] — 16 = 2*8, két elemmel arrébb

; int32 tömb: int[] arr;
mov eax, [rbx + rcx*4]        ; arr[i] — 4 bájtos elemek
```

**Java párhuzam:** Egy Java tömb indexelése (`arr[i]`) pontosan ezt csinálja JVM szinten:
```
cím = arr_alap + i * elem_méret
```
Assembly-ben te magad számolod ki — és a hardver segít a `[base + index*scale]` szintaxissal.

### 1.7 Összefoglaló táblázat

| Mód | Szintaxis | Példa | Tipikus használat |
|-----|-----------|-------|-------------------|
| Immediate | `szám` | `mov rax, 42` | Konstans betöltés |
| Register | `reg` | `mov rax, rbx` | Regiszterek közötti másolás |
| Direct | `[label]` | `mov rax, [ertek]` | Globális változó olvasás |
| Register indirect | `[reg]` | `mov rax, [rbx]` | Mutató dereference |
| Base + disp | `[reg ± n]` | `mov rax, [rbp-8]` | Stack változó, struktúra mező |
| Base + idx*scale | `[r1 + r2*s]` | `mov rax, [rbx+rcx*8]` | Tömb indexelés |
| Base + idx*scale + disp | `[r1 + r2*s + n]` | `mov rax, [rbx+rcx*4+8]` | Tömb + offset |

> **💡 Tipp:** A scale értéke csak 1, 2, 4 vagy 8 lehet (mert ezek a tipikus adattípus-méretek). Ha más szorzóra van szükség, LEA-t kell használni.

---

## 2. LEA részletesen

A **LEA** (Load Effective Address) az egyik legsokoldalúbb utasítás x86-64-ben. Neve félrevezető lehet: nem tölt be memória-tartalmat, hanem **a kiszámított cím értékét** tölti be a cél regiszterbe.

### 2.1 Alap szintaxis

```nasm
lea rax, [rbx + rcx*4 + 8]    ; rax = rbx + rcx*4 + 8  (EZ EGY CÍM, nem memória-olvasás!)
```

**Különbség MOV-tól:**

```nasm
mov rax, [rbx + rcx*4 + 8]    ; rax = *(rbx + rcx*4 + 8)  → MEMÓRIÁBÓL OLVAS
lea rax, [rbx + rcx*4 + 8]    ; rax =  (rbx + rcx*4 + 8)  → CSAK KISZÁMOLJA A CÍMET
```

### 2.2 LEA aritmetikára – gyors szorzás és összeadás

A LEA nem csak memóriacímek kiszámítására jó — aritmetikai trükkökre is kiválóan alkalmas, mivel egyetlen utasítással három műveletet végez (szorzás, összeadás, eltolás).

```nasm
; rax = rbx * 3
lea rax, [rbx + rbx*2]        ; rax = rbx + rbx*2 = rbx*3

; rax = rbx * 5
lea rax, [rbx + rbx*4]        ; rax = rbx + rbx*4 = rbx*5

; rax = rbx * 9
lea rax, [rbx + rbx*8]        ; rax = rbx + rbx*8 = rbx*9

; rax = rbx * 3 + 7
lea rax, [rbx + rbx*2 + 7]    ; rax = rbx*3 + 7

; rax = rbx + rcx*8 (tömb cím kiszámítása, a cím tárolása)
lea rax, [rbx + rcx*8]        ; rax = tömb[rcx] TITLE (mint egy pointer)
```

> **💡 Tipp:** A LEA nem módosítja a flag-eket (RFLAGS-t), ezért biztonságosabb aritmetikai célokra, ha nem akarjuk megbolygatni a feltételes elágazásokat.

### 2.3 LEA string műveleteknél

A LEA kiválóan alkalmas arra, hogy egy szöveg/adat egy részébe mutassunk:

```nasm
section .data
    szoveg  db "Hello, World!", 0

section .text
    lea rdi, [szoveg + 7]     ; rdi → "World!" elejére mutat (7. bájttól)
    lea rsi, [szoveg]         ; rsi → szoveg elejére (ugyanaz mint: mov rsi, szoveg)
```

---

## 3. Adatszekciók részletesen

### 3.1 `.data` – inicializált adatok

A `.data` szekció programinduláskor betöltött, módosítható adatokat tartalmaz.

```nasm
section .data
    ; db = define byte (1 bájt)
    egy_bajt    db 42               ; értéke: 42
    betuk       db 'A', 'B', 'C'    ; 3 bájt egymás után
    szoveg      db "hello", 0       ; string + null terminator

    ; dw = define word (2 bájt, 16-bit)
    ket_bajt    dw 1000             ; 2 bájtos egész

    ; dd = define dword (4 bájt, 32-bit)
    negy_bajt   dd 100000           ; 4 bájtos egész

    ; dq = define qword (8 bájt, 64-bit)
    nyolc_bajt  dq 1234567890       ; 8 bájtos egész
    float_val   dq 3.14             ; 64-bites lebegőpontos (IEEE 754 double)

    ; times – ismétlés
    nullak      times 10 db 0       ; 10 darab 0 értékű bájt
    szazak      times 5  dw 100     ; 5 darab 100 értékű word
    tomb        times 8  dq 0       ; 8 darab 64-bites 0 (int64 tömb)
```

### 3.2 `.bss` – nem inicializált adatok

A `.bss` szekció lefoglalja a helyet, de nem tölt semmit — a program induláskor nullával lesz tele. **Nem vesz el helyet a futtatható fájlban** (csak a mérete van tárolva).

```nasm
section .bss
    ; resb = reserve byte
    puffer      resb 256            ; 256 bájtos puffer

    ; resw = reserve word (2 bájt)
    szavak      resw 10             ; 10 * 2 = 20 bájt

    ; resd = reserve dword (4 bájt)
    egeszek     resd 20             ; 20 * 4 = 80 bájt (int32 tömb)

    ; resq = reserve qword (8 bájt)
    hosszuak    resq 5              ; 5 * 8 = 40 bájt (int64 tömb)
```

> **⚠️ Figyelem:** `.bss`-be nem lehet kezdőértéket adni. Ha `resq 5` után értékeket akarsz, `.data`-ba tedd, vagy induláskor töltsd fel kóddal.

### 3.3 `.rodata` – csak olvasható adatok

NASM-ban a `.rodata` szekció használható olvasásvédett konstansokhoz. A linker és az OS gondoskodik arról, hogy írási kísérlet szegmentációs hibát okozzon.

```nasm
section .rodata
    PI          dq 3.141592653589793   ; double konstans
    UZENET      db "Hiba tortent!", 10, 0  ; hibaüzenet
    VERZIO      db "1.0.0", 0
```

> **💡 Tipp:** String literálokat érdemes `.rodata`-ba tenni, nem `.data`-ba, ha nem kell módosítani őket. Ez a program biztonságát is növeli.

### 3.4 EQU – szimbolikus konstansok

Az `EQU` nem foglal memóriát — csak egy nevet rendel egy konstans értékhez, mint a `#define` C-ben.

```nasm
STDIN       EQU 0           ; standard input file descriptor
STDOUT      EQU 1           ; standard output file descriptor
STDERR      EQU 2           ; standard error file descriptor

SYS_WRITE   EQU 1           ; write syscall szám
SYS_EXIT    EQU 60          ; exit syscall szám

MAX_MERET   EQU 1024        ; puffer maximális mérete
TOMB_MERET  EQU 10          ; tömb elemszáma
```

### 3.5 Tömbök definiálása és elérése

```nasm
section .data
    ; int64 tömb: long[] arr = {10, 20, 30, 40, 50};
    arr     dq 10, 20, 30, 40, 50
    ARR_N   EQU 5               ; elemszám (EQU, nem memória!)

    ; String tömb (mutatók tömbje)
    str1    db "elso", 0
    str2    db "masodik", 0
    str3    db "harmadik", 0
    strptr  dq str1, str2, str3  ; 3 db pointer (mint Java String[])
```

Tömb elemeinek elérése:

```nasm
    ; arr[0] olvasása
    mov rax, [arr]              ; első elem

    ; arr[2] olvasása (0-tól indexelt, 8 bájtos elemek)
    mov rax, [arr + 2*8]        ; arr + 2*sizeof(int64) = arr + 16
    ; vagy regiszterrel:
    mov rcx, 2
    mov rax, [arr + rcx*8]      ; arr[rcx]

    ; arr[i] írása
    mov rcx, 3
    mov qword [arr + rcx*8], 99  ; arr[3] = 99
```

---

## 4. Mutatók Assembly-ben

Assembly-ben **minden cím 64 bites** (x86-64-en). Nincs típus, nincs GC — a mutató egyszerűen egy 64-bites szám, ami egy memóriacímet jelöl.

### 4.1 Mutató alapok

```nasm
section .data
    ertek   dq 42

section .text
global _start
_start:
    mov rax, ertek          ; rax = a CÍM ahol 42 van tárolva
    mov rbx, [rax]          ; rbx = 42  (dereference: olvass a rax által mutatott helyről)
    mov qword [rax], 100    ; *rax = 100  (írás a mutatott helyre)
```

**Java párhuzam:**
```java
long[] ertek = {42};       // ertek egy referencia (cím)
long rbx = ertek[0];       // [rax] — dereference
ertek[0] = 100;            // [rax] = 100 — írás
```

### 4.2 Size specifier-ek – mikor kell?

Ha a fordítónak nem egyértelmű, mekkora memóriaterületet kell olvasni/írni, explicit **méretjelzőt** kell használni.

| Jelző | Méret | Regiszter-méret |
|-------|-------|-----------------|
| `BYTE PTR` / `byte` | 1 bájt | al, bl, ... |
| `WORD PTR` / `word` | 2 bájt | ax, bx, ... |
| `DWORD PTR` / `dword` | 4 bájt | eax, ebx, ... |
| `QWORD PTR` / `qword` | 8 bájt | rax, rbx, ... |

```nasm
section .bss
    adat    resb 8      ; 8 bájt lefoglalva

section .text
    ; Egyértelmű: mov rax, [adat] → 8 bájt (rax 64-bites)
    mov rax, [adat]

    ; Egyértelmű: mov eax, [adat] → 4 bájt (eax 32-bites)
    mov eax, [adat]

    ; NEM egyértelmű: immediate írás → kell a méretjelző!
    mov byte  [adat], 42        ; 1 bájt írása
    mov word  [adat], 1000      ; 2 bájt írása
    mov dword [adat], 100000    ; 4 bájt írása
    mov qword [adat], 99999999  ; 8 bájt írása
```

> **⚠️ Figyelem:** Ha immediate értéket írsz memóriába, a méretjelző **kötelező**, mert a fordító nem tudja megállapítani, hány bájtot kell módosítani.

### 4.3 Mutató aritmetika

```nasm
    mov rbx, tomb_cime      ; rbx → tömb elejére mutat
    add rbx, 8              ; rbx → következő int64 elemre (1-es index)
    add rbx, 8              ; rbx → 2-es indexű elemre
    mov rax, [rbx]          ; rax = tomb[2]

    ; Gyorsabb: LEA-val lépünk előre
    lea rbx, [tomb_cime + 2*8]   ; rbx → tomb[2] TITLE
```

---

## 5. Endianness

### 5.1 Mi az endianness?

Az **endianness** (bájtsorrend) azt határozza meg, hogy egy több-bájtos szám bájtjait milyen sorrendben tárolja a memória.

- **Little-endian (kis-végű):** Az alacsony helyiértékű bájt kerül az alacsonyabb memóriacímre.
- **Big-endian (nagy-végű):** A magas helyiértékű bájt kerül az alacsonyabb memóriacímre.

**Az x86-64 architektúra mindig little-endian.**

### 5.2 Szemléltetés

Tároljuk el a `0x0102030405060708` értéket a `0x1000` memóriacímtől:

| Memóriacím | Little-endian | Big-endian |
|------------|---------------|------------|
| 0x1000 | `08` (LSB) | `01` (MSB) |
| 0x1001 | `07` | `02` |
| 0x1002 | `06` | `03` |
| 0x1003 | `05` | `04` |
| 0x1004 | `04` | `05` |
| 0x1005 | `03` | `06` |
| 0x1006 | `02` | `07` |
| 0x1007 | `01` (MSB) | `08` (LSB) |

```nasm
section .data
    szam    dq 0x0102030405060708

; Ha GDB-vel megnézed a memóriát:
; x/8xb &szam
; Eredmény: 08 07 06 05 04 03 02 01  ← fordított sorrend!
```

### 5.3 Mikor számít az endianness?

**Helyi adatfeldolgozásban (ugyanazon x86 gépen) általában nem számít** — az x86 automatikusan helyes sorrendben kezeli az adatokat.

Számít viszont:
1. **Hálózati kommunikáció:** A hálózati protokollok (TCP/IP) big-endian sorrendet ("network byte order") használnak. Linux: `htonl()`, `ntohl()` függvények.
2. **Fájlformátumok:** Egyes bináris fájlformátumok big-endiant használnak (pl. régi JPEG, PNG chunk fejlécek, Java `.class` fájlok).
3. **Cross-platform adatcsere:** Ha x86-on írt bináris fájlt ARM big-endian rendszeren olvasol.
4. **Protokoll implementáció:** Ha manuálisan olvasol/írsz bináris protokollt.

```nasm
; Bájtsorrend csere (32-bit): BSWAP utasítás
mov eax, 0x01020304
bswap eax              ; eax = 0x04030201 (big → little vagy fordítva)

; 64-bit:
mov rax, 0x0102030405060708
bswap rax              ; rax = 0x0807060504030201
```

> **💡 Tipp:** A `bswap` utasítás egyetlen óraciklus alatt elvégzi a bájtsorrendcserét. Hálózati programozásnál vagy bináris fájlformátumok olvasásakor hasznos.

---

## 6. Példaprogramok

### 6.1 Tömbösszegzés (for ciklus + tömbindexelés)

```nasm
; tomb_osszeg.asm
; Összegzi egy int64 tömb elemeit
; Fordítás és futtatás:
;   nasm -f elf64 -g -F dwarf tomb_osszeg.asm -o tomb_osszeg.o
;   ld tomb_osszeg.o -o tomb_osszeg
;   ./tomb_osszeg
;   echo $?   ; visszatérési kód = összeg (de max 255 az echo $? miatt)

section .data
    tomb    dq 10, 20, 30, 40, 50    ; long[] tomb = {10,20,30,40,50}
    MERET   EQU 5                    ; elemszám

section .text
global _start

_start:
    xor rax, rax        ; rax = 0 (összeg)
    xor rcx, rcx        ; rcx = 0 (ciklusszámláló i)

.ciklus:
    cmp rcx, MERET      ; if (i >= MERET) break
    jge .vege

    ; rax += tomb[i]  →  tomb eleje + i*8 bájt offset
    add rax, [tomb + rcx*8]

    inc rcx             ; i++
    jmp .ciklus

.vege:
    ; rax = 150 (10+20+30+40+50)
    ; Kilépés: exit(rax) — de az exit kód max 255!
    mov rdi, rax        ; exit kód = összeg
    mov rax, 60         ; sys_exit
    syscall
```

**Kimenet:**
```bash
./tomb_osszeg
echo $?   # 150
```

### 6.2 LEA aritmetika demonstráció

```nasm
; lea_demo.asm
; LEA különböző aritmetikai célokra
; Fordítás:
;   nasm -f elf64 -g -F dwarf lea_demo.asm -o lea_demo.o
;   ld lea_demo.o -o lea_demo
;   ./lea_demo && echo $?

section .text
global _start

_start:
    mov rbx, 7          ; rbx = 7 (bemeneti érték)

    ; rbx * 3 = 21
    lea rax, [rbx + rbx*2]      ; rax = 7 + 7*2 = 21

    ; rbx * 5 = 35
    lea rcx, [rbx + rbx*4]      ; rcx = 7 + 7*4 = 35

    ; rbx * 10 = 70  (két lépésben: *5, majd *2)
    lea rdx, [rbx + rbx*4]      ; rdx = rbx * 5 = 35
    lea rdx, [rdx + rdx]        ; rdx = rdx * 2 = 70

    ; Ellenőrzés: rax + rcx = 21 + 35 = 56
    add rax, rcx                ; rax = 56

    ; Kilépési kód = rax értéke
    mov rdi, rax
    mov rax, 60
    syscall
; echo $? → 56
```

### 6.3 Mutató demonstráció: indirect access + módosítás

```nasm
; mutato_demo.asm
; Mutatók és indirect memóriaelérés bemutatása
; Fordítás:
;   nasm -f elf64 -g -F dwarf mutato_demo.asm -o mutato_demo.o
;   ld mutato_demo.o -o mutato_demo
;   ./mutato_demo && echo $?

section .data
    ertek   dq 10           ; long ertek = 10;

section .bss
    masik   resq 1          ; long masik;  (inicializálatlan)

section .text
global _start

_start:
    ; --- OLVASÁS mutatón keresztül ---
    ; rbx egy "mutató" ertek-re (Java: Long ref = &ertek)
    mov rbx, ertek          ; rbx = ertek TITLE (a memóriacím)
    mov rax, [rbx]          ; rax = *rbx = 10  (dereference)

    ; --- ÍRÁS mutatón keresztül ---
    ; ertek = ertek * 3 + 5  (LEA-val)
    lea rax, [rax + rax*2]  ; rax = 10*3 = 30
    add rax, 5              ; rax = 35
    mov [rbx], rax          ; *rbx = 35  (visszaírjuk ertek-be)

    ; --- MUTATÓ MÁSOLÁSA, majd olvasás ---
    mov rcx, rbx            ; rcx = rbx (mindkettő ertek-re mutat)
    mov rdx, [rcx]          ; rdx = *rcx = 35

    ; --- BSS szekció írása ---
    mov rbx, masik          ; rbx → masik változóra mutat
    mov qword [rbx], rdx    ; *rbx = 35 (masik = 35)

    ; Kilépés: exit(masik értéke) = exit(35)
    mov rdi, [masik]
    mov rax, 60
    syscall
; echo $? → 35
```

---

## 7. Gyakorlatok

### 1. feladat – Tömb maximuma
Írj programot, amely megkeresi az alábbi tömb legnagyobb elemét, és azzal tér vissza (exit kód):
```nasm
tomb    dq 15, 3, 99, 42, 7, 88, 1
```
*Segítség:* Használj `cmp` + feltételes ugró utasítást (pl. `jg`/`jl`).

### 2. feladat – LEA szorzás
Számítsd ki LEA-val a következőket (ne használj `imul`-t):
- `rax = rbx * 7` (ahol `rbx = 6`)
- `rax = rbx * 12`
- `rax = rbx * 15`

*Segítség:* `*7 = *8 - 1`, `*12 = *4 * 3`, `*15 = *16 - 1` — vagy más felbontás.

### 3. feladat – Mutató lánc
Hozz létre három `dq` változót: `a`, `b`, `c`. Tárold el `b`-ben az `a` TITLE-t, `c`-ben a `b` TITLE-t. Majd `c` segítségével add hozzá 10-et az `a` értékéhez (dupla dereference: `[[c]]`).

### 4. feladat – Struct szimuláció
Szimuláld az alábbi C struktúrát Assembly-ben:
```c
struct Pont { int64_t x; int64_t y; };
struct Pont p = {3, 7};
```
Számítsd ki a Pythagoras-tétel szerinti távolságot az origótól: `x*x + y*y` (egész). Tipp: az `imul` utasítás.

### 5. feladat – Endianness vizsgálat
Tárolj el egy `dq 0xDEADBEEF01020304` értéket, majd GDB-ben nézd meg `x/8xb &ertek` paranccsal a tényleges bájtelrendezést. Rajzold fel papírra a memória tartalmát, jelöld meg az MSB-t és LSB-t.

---

## 8. Ellenőrizd magad

1. **Mi a különbség** `mov rax, ertek` és `mov rax, [ertek]` között? Mikor melyiket használod?

2. **Mire való a LEA** utasítás? Miért jobb ez `mov + add + imul` kombinációnál néhány esetben?

3. **Miért csak 1, 2, 4 vagy 8 lehet** a scale értéke a `[base + index*scale]` címzésben?

4. **Mikor kötelező** a méretjelző (BYTE, WORD, DWORD, QWORD) a memóriahivatkozásnál?

5. **Mi a különbség** `.data` és `.bss` szekció között? Mi a `.bss` előnye?

6. **Milyen esetekben fontos** az endianness figyelembe vétele x86-64 programozásnál?

---

*Következő fejezet: [7. nap – Syscall-ok: I/O és fájlkezelés](../nap-07/README.md)*
