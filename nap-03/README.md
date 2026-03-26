# 3. nap: Adatmozgatás és aritmetika

## Mit tanulunk ma?

Ma megismerjük az assembly programozás két alapkövét: az **adatmozgatást** és az **aritmetikát**. Megtanuljuk, hogyan mozgatjuk az adatokat regiszterek, memória és konstansok között (`MOV`), hogyan végzünk alapvető matematikai műveleteket (`ADD`, `SUB`, `MUL`, `DIV`), és megismerkedünk a **RFLAGS** regiszterrel, amely az utasítások mellékhatásait – például hogy az eredmény nulla volt-e, vagy túlcsordulás történt-e – rögzíti. A fejezet végére képes leszel önálló számológép programot írni assembly-ben.

> **💡 Java analógia:** Amit Java-ban egyetlen sorban megírsz (`int c = a + b;`), azt assembly-ben legalább 3 utasítás végzi el: betöltés regiszterbe, összeadás, eredmény tárolása. Az assembly-ben te irányítod az egész folyamatot.

---

## 1. A MOV utasítás részletesen

A `MOV` az assembly leggyakoribb utasítása. Adatot másol a forrásból a célba. A neve kissé félrevezető: nem mozgat, hanem **másol** (a forrás érintetlen marad).

### Szintaxis

```nasm
mov dst, src    ; dst = src
```

Az operandusok lehetnek:

| Forrás (src)      | Cél (dst)          | Érvényes? |
|-------------------|--------------------|-----------|
| regiszter         | regiszter          | ✅ igen   |
| konstans (imm)    | regiszter          | ✅ igen   |
| memória           | regiszter          | ✅ igen   |
| regiszter         | memória            | ✅ igen   |
| konstans (imm)    | memória            | ✅ igen   |
| memória           | memória            | ❌ NEM!   |

> **⚠️ Figyelem:** Memóriából memóriába közvetlenül **nem** lehet mozgatni. Mindig kell egy regiszter közvetítőként. Ez hardveres megkötés – az ALU csak regisztereken tud operálni.

### Példák

```nasm
; Regiszter ← regiszter
mov rax, rbx        ; rax = rbx értéke

; Regiszter ← konstans (immediate value)
mov rax, 42         ; rax = 42
mov rdi, 0          ; rdi = 0

; Regiszter ← memória
mov rax, [rbx]      ; rax = *(rbx)  (Java-ban: rax = *rbx pointer dereferencia)
mov rax, [szam]     ; rax = szam változó értéke (data szekcióból)

; Memória ← regiszter
mov [rbx], rax      ; *(rbx) = rax
mov [cim], rdx      ; cim változóba írás

; Memória ← memória: NEM MŰKÖDIK
; mov [cim1], [cim2]  ; ← FORDÍTÁSI HIBA!
; Helyette:
mov rax, [cim2]     ; köztes regiszter
mov [cim1], rax
```

> **💡 Java analógia:** A `mov rax, [szam]` olyan, mint Java-ban `int result = szam;` – ha `szam` egy mezőre mutató referencia. A szögletes zárójel a **dereferencia** operátor assembly-ben.

### Méretjelölők

Ha a cél mérete nem egyértelmű (pl. memóriát írunk közvetlen konstanssal), meg kell adni a méretet:

```nasm
mov byte  [rbx], 42    ; 1 bájt írása
mov word  [rbx], 42    ; 2 bájt (16-bit)
mov dword [rbx], 42    ; 4 bájt (32-bit)
mov qword [rbx], 42    ; 8 bájt (64-bit)
```

### MOVZX – Zero-Extend (előjelbitek kiegészítése nullával)

Kisebb méretű értéket tölt be egy nagyobb regiszterbe, a felső biteket **nullával** tölti ki.

```nasm
movzx rax, byte [rbx]   ; rax = (uint64_t)(uint8_t)(*rbx)
movzx rax, word [rbx]   ; rax = (uint64_t)(uint16_t)(*rbx)
movzx eax, bl           ; eax = (uint32_t)(uint8_t)bl
```

> **💡 Java analógia:** Olyan, mint a **widening cast** unsigned irányban: `long result = (byte) b & 0xFF;` – a `& 0xFF` rész biztosítja, hogy az előjelbit ne terjedjen ki.

### MOVSX – Sign-Extend (előjelbit kiterjesztése)

Kisebb méretű **előjeles** értéket tölt be, az előjelbitet **másolja** a felső bitpozíciókba.

```nasm
movsx rax, byte [rbx]   ; rax = (int64_t)(int8_t)(*rbx)
movsx rax, word [rbx]   ; rax = (int64_t)(int16_t)(*rbx)
movsx eax, bl           ; eax = (int32_t)(int8_t)bl
```

> **💡 Java analógia:** Ez a **widening cast** előjeles irányban: `long result = (byte) b;` – Java automatikusan elvégzi a sign extension-t.

**Miért fontosak?** Ha egy `int`-et (32-bit) `long`-ba (64-bit) kell konvertálni, MOVSX gondoskodik arról, hogy a negatív számok (-1 = `0xFF...FF`) helyesen maradjanak negatívak a nagyobb típusban. MOVZX esetén -1 (8-bit: `0xFF`) → 255 lesz 64-bitben.

---

## 2. Aritmetikai utasítások

### ADD és SUB

```nasm
add dst, src    ; dst = dst + src
sub dst, src    ; dst = dst - src
```

Mindkét operandus lehet regiszter vagy memória (de egyszerre nem mindkettő memória!). A `src` lehet közvetlen konstans is.

```nasm
mov rax, 10
mov rbx, 3
add rax, rbx    ; rax = 13
sub rax, rbx    ; rax = 10
add rax, 100    ; rax = 110  (konstans hozzáadása)
sub rbx, 1     ; rbx = 2
```

### INC és DEC

```nasm
inc dst     ; dst = dst + 1
dec dst     ; dst = dst - 1
```

Hatékonyabb, mint `add rax, 1`, és **nem módosítja a Carry Flag-et** (ez fontos multi-precision aritmetikánál).

```nasm
inc rax      ; rax++
dec rcx      ; rcx--
inc qword [szamlalo]  ; memóriában lévő értéket növeli
```

### NEG – kettes komplemens negálás

```nasm
neg dst     ; dst = 0 - dst
```

```nasm
mov rax, 5
neg rax     ; rax = -5
neg rax     ; rax = 5 (ismét)
```

> **💡 Java analógia:** `rax = -rax;`

### XOR reg, reg – regiszter nullázás

A leggyakoribb módszer egy regiszter **nullára állítására**:

```nasm
xor rax, rax    ; rax = 0
xor rdx, rdx    ; rdx = 0
```

**Miért ez a legjobb módszer?** Mert:
1. **Rövidebb gépi kód** mint `mov rax, 0` (3 bájt vs. 7 bájt 64-biten)
2. **Gyorsabb** – a CPU felismeri és optimalizálja (nem függ az rax előző értékétől)
3. Hagyomány és széles körben elfogadott idioma

`xor eax, eax` is tökéletesen nullázza `rax`-ot, mert 32-bites műveletek automatikusan nullázzák a felső 32 bitet!

### MUL – unsigned szorzás

```nasm
mul src     ; rdx:rax = rax * src
```

A `MUL` mindig az `rax`-szal szoroz, és az eredményt a **rdx:rax regiszter párba** teszi (128 bites eredmény).

```nasm
mov rax, 6
mov rbx, 7
xor rdx, rdx    ; rdx nullázása (fontos!)
mul rbx         ; rdx:rax = 6 * 7 = 42
                ; rax = 42, rdx = 0 (ha az eredmény belefér 64-bitbe)
```

> **⚠️ Figyelem:** Ha `rdx`-et nem nullázod `MUL` előtt, a szorzás eredménye hibás lesz, mert a `rdx:rax` pár tartalmazza a szorzandót.

### IMUL – signed szorzás (3 variáns)

Az `IMUL` sokkal rugalmasabb, mint `MUL`:

```nasm
; 1 operandus – mint MUL, de előjeles (rdx:rax = rax * src)
imul rbx            ; rdx:rax = rax * rbx (signed)

; 2 operandus – dst *= src
imul rax, rbx       ; rax = rax * rbx (64-bit eredmény)

; 3 operandus – dst = src1 * imm
imul rax, rbx, 10   ; rax = rbx * 10
```

A 2 és 3 operandusú változat **nem módosítja rdx-et**, és az eredmény csonkítva van 64-bitre – ez az általánosan használt forma.

```nasm
mov rbx, 7
imul rax, rbx, 6    ; rax = 7 * 6 = 42
```

### DIV – unsigned osztás

```nasm
div src     ; rax = rdx:rax / src (hányados)
            ; rdx = rdx:rax % src (maradék)
```

> **⚠️ Figyelem:** `rdx`-et **mindig nullázni kell** `DIV` előtt (ha 64-bites osztást végzünk egyszerű számokkal), különben `divide overflow` exception keletkezik!

```nasm
mov rax, 42
xor rdx, rdx    ; rdx = 0 (KÖTELEZŐ!)
mov rbx, 5
div rbx         ; rax = 42 / 5 = 8
                ; rdx = 42 % 5 = 2
```

### IDIV – signed osztás

```nasm
idiv src    ; rax = rdx:rax / src (signed hányados)
            ; rdx = rdx:rax % src (signed maradék)
```

Signed osztás előtt `rdx`-et `rax` előjelétől függően kell beállítani. Erre a `CQO` utasítás való:

```nasm
mov rax, -42
cqo             ; rdx:rax = sign-extend(rax) → rdx = 0xFFFFFFFFFFFFFFFF ha rax negatív
mov rbx, 5
idiv rbx        ; rax = -42 / 5 = -8 (csonkítás nulla felé)
                ; rdx = -42 % 5 = -2
```

> **💡 Java analógia:** `int q = a / b;` és `int r = a % b;` – assembly-ben egyetlen `IDIV` mindkettőt egyszerre számolja.

---

## 3. A RFLAGS regiszter és flag-ek

A RFLAGS egy 64-bites regiszter, amelynek egyes bitjei (**flag-ek**) az utolsó aritmetikai/logikai művelet eredményéről tárolnak információt. Ezeket a flag-eket a feltételes ugróutasítások (következő nap!) olvassák.

### A legfontosabb flag-ek

| Flag | Neve            | Beállítás feltétele                                        |
|------|-----------------|------------------------------------------------------------|
| CF   | Carry Flag      | Unsigned túlcsordulás (átvitel a legfelső bitből)          |
| ZF   | Zero Flag       | Az eredmény nulla                                          |
| SF   | Sign Flag       | Az eredmény negatív (a legfelső bit értéke)                |
| OF   | Overflow Flag   | Signed túlcsordulás (az előjelbit hibásan változott)       |
| PF   | Parity Flag     | Az alacsony 8 bit páros számú 1-es bitet tartalmaz         |
| AF   | Auxiliary Flag  | Átvitel a 3-4. bit közt (BCD aritmetikához)               |

### Melyik utasítás melyik flag-et állítja?

| Utasítás       | CF | ZF | SF | OF |
|----------------|----|----|----|-----|
| `ADD`          | ✅ | ✅ | ✅ | ✅ |
| `SUB` / `CMP` | ✅ | ✅ | ✅ | ✅ |
| `INC` / `DEC` | ❌ | ✅ | ✅ | ✅ |
| `MUL`          | ✅ | ❌ | ❌ | ✅ |
| `IMUL`         | ✅ | ❌ | ❌ | ✅ |
| `AND`/`OR`/`XOR` | 0 (töröl) | ✅ | ✅ | 0 (töröl) |
| `SHL`/`SHR`   | ✅ | ✅ | ✅ | ✅ |
| `NEG`          | ✅ | ✅ | ✅ | ✅ |

> **💡 Miért fontosak a flag-ek?** A 4. napon tanulunk feltételes ugrásokat (`je`, `jne`, `jl`, `jg` stb.), amelyek közvetlenül ezeket a flag-eket olvassák. Például `je` (jump if equal) akkor ugrik, ha ZF=1.

### Flag példák

```nasm
mov rax, 0
mov rbx, 0
add rax, rbx    ; ZF = 1 (eredmény nulla), SF = 0, CF = 0, OF = 0

mov rax, 0xFFFFFFFFFFFFFFFF  ; rax = -1 (unsigned max)
add rax, 1      ; CF = 1 (unsigned overflow!), ZF = 1 (eredmény = 0)
                ; OF = 0 (signed: -1 + 1 = 0, rendben)

mov rax, 0x7FFFFFFFFFFFFFFF  ; signed max (9223372036854775807)
add rax, 1      ; OF = 1 (signed overflow!), SF = 1 (eredmény negatív lett)
                ; CF = 0 (unsigned szempontból nincs overflow)
```

---

## 4. Bitműveletek

### AND, OR, XOR, NOT

```nasm
and dst, src    ; dst = dst & src   (Java: &)
or  dst, src    ; dst = dst | src   (Java: |)
xor dst, src    ; dst = dst ^ src   (Java: ^)
not dst         ; dst = ~dst        (Java: ~)
```

**Gyakorlati alkalmazások:**

```nasm
; Bit maszkolás – csak az alsó 4 bit megtartása
mov rax, 0b10110111    ; rax = 183
and rax, 0x0F          ; rax = 0b00000111 = 7 (felső 4 bit törlése)

; Bit beállítása (set) – 3. bit bekapcsolása
mov rax, 0b00001010    ; rax = 10
or  rax, 0b00000100    ; rax = 0b00001110 = 14 (3. bit = 1)

; Bit törlése (clear) – 1. bit kikapcsolása
mov rax, 0b00001110    ; rax = 14
and rax, 0b11111101    ; rax = 0b00001100 = 12 (1. bit = 0)

; Bit váltása (toggle) – 3. bit megfordítása
mov rax, 0b00001110
xor rax, 0b00000100    ; rax = 0b00001010 = 10 (3. bit megfordult)

; Regiszter nullázás
xor rax, rax           ; rax = 0 (a leggyakoribb idioma!)
```

> **💡 Java analógia:** `int masked = value & 0x0F;` – teljesen azonos viselkedés.

### SHL, SHR, SAR – Bit eltolás

```nasm
shl dst, cl     ; dst <<= cl  (shift left, logikai)     Java: <<
shr dst, cl     ; dst >>= cl  (shift right, logikai)    Java: >>>
sar dst, cl     ; dst >>= cl  (shift right, aritmetikai) Java: >>
```

A shift mennyisége lehet közvetlen konstans (max 63) vagy a `cl` regiszter:

```nasm
shl rax, 1      ; rax *= 2 (2^1)
shl rax, 3      ; rax *= 8 (2^3)
shr rax, 1      ; rax /= 2 (unsigned)
sar rax, 1      ; rax /= 2 (signed, az előjelbit megmarad)

; Változó shift: cl regiszterből
mov rcx, 4
shl rbx, cl     ; rbx <<= 4  (rbx *= 16)
```

**SHR vs SAR különbség:**

```nasm
mov rax, -8     ; rax = 0xFFFFFFFFFFFFFFF8
shr rax, 1      ; rax = 0x7FFFFFFFFFFFFFFC = nagy pozitív szám! (Java: >>>)
                ; A felső bit 0 lesz!

mov rax, -8     ; rax = 0xFFFFFFFFFFFFFFF8
sar rax, 1      ; rax = 0xFFFFFFFFFFFFFFFC = -4 (helyes signed osztás 2-vel)
                ; Az előjelbit megmarad!
```

> **💡 Java analógia:** `>>` (SAR, előjeltartó) vs `>>>` (SHR, logikai, nullával tölt).

**2 hatványával szorzás/osztás – gyors trükk:**

```nasm
; n * 4  →  shl n, 2   (2^2 = 4)
; n * 8  →  shl n, 3   (2^3 = 8)
; n / 2  →  sar n, 1   (signed esetén)
; n / 4  →  sar n, 2
```

---

## 5. LEA – Load Effective Address

A `LEA` (Load Effective Address) kiszámít egy memóriacímet, de **nem olvassa ki** az adott memória tartalmát – csupán a kiszámolt **számot** (a címet) tölti be a célregiszterbe.

```nasm
lea dst, [kifejezés]
```

### Memóriacím kiszámítása

```nasm
lea rax, [rbx + 8]          ; rax = rbx + 8
lea rax, [rbx + rcx]        ; rax = rbx + rcx
lea rax, [rbx + rcx*4]      ; rax = rbx + rcx*4
lea rax, [rbx + rcx*4 + 8]  ; rax = rbx + rcx*4 + 8
```

> **⚠️ Különbség MOV-tól:** `mov rax, [rbx+8]` → rax = *(rbx+8) (kiolvasás a memóriából). `lea rax, [rbx+8]` → rax = rbx+8 (csak a számítás, nem olvasás).

### LEA aritmetikai trükkként

A LEA **egy utasításban** végez összeadást és szorzást (2, 4, 8 szorzókkal):

```nasm
; rax = rbx * 3  (egy utasítással!)
lea rax, [rbx + rbx*2]

; rax = rbx * 5
lea rax, [rbx + rbx*4]

; rax = rbx * 9
lea rax, [rbx + rbx*8]

; rax = rbx * 4 + 7  (egy utasítással!)
lea rax, [rbx*4 + 7]
```

> **💡 Miért hasznos?** Gyorsabb, mint `imul rax, rbx, 3`, nem módosítja a flag-eket, és egy órajelciklus alatt fut. Nagyon hasznos indexszámításoknál (pl. tömbindexelés, struktúra mezők elérése).

---

## 6. Teljes példaprogramok

### Program 1: Két szám összeadása, eredmény az exit kódban

```nasm
; osszead.asm – két szám összeadása
; Fordítás:
;   nasm -f elf64 -g -F dwarf osszead.asm -o osszead.o
;   ld osszead.o -o osszead
;   ./osszead
;   echo $?   ; → 42

section .data
    szam1 dq 17       ; első szám (8 bájt = quadword)
    szam2 dq 25       ; második szám

section .text
    global _start

_start:
    ; Betöltjük a két számot regiszterekbe
    mov rax, [szam1]   ; rax = 17
    mov rbx, [szam2]   ; rbx = 25

    ; Összeadás
    add rax, rbx       ; rax = 17 + 25 = 42

    ; Kilépés: exit(rax)
    ; syscall 60 = exit, argumentum: rdi = visszatérési kód
    mov rdi, rax       ; rdi = 42 (exit kód)
    mov rax, 60        ; syscall: exit
    syscall
```

**Fordítás és futtatás:**

```bash
nasm -f elf64 -g -F dwarf osszead.asm -o osszead.o
ld osszead.o -o osszead
./osszead
echo $?    # kimenet: 42
```

---

### Program 2: Szorzás és osztás demonstráció

```nasm
; szorzas_osztas.asm – szorzás és osztás bemutatása
; A program kiszámolja: (10 * 7) / 2 = 35
; Fordítás:
;   nasm -f elf64 -g -F dwarf szorzas_osztas.asm -o szorzas_osztas.o
;   ld szorzas_osztas.o -o szorzas_osztas
;   ./szorzas_osztas
;   echo $?   ; → 35

section .text
    global _start

_start:
    ; === SZORZÁS: 10 * 7 ===
    mov rax, 10        ; rax = 10 (MUL mindig rax-szal szoroz)
    mov rbx, 7         ; rbx = 7

    imul rax, rbx      ; rax = 10 * 7 = 70  (2 operandusú imul, nem érinti rdx-et)

    ; === OSZTÁS: 70 / 2 ===
    xor rdx, rdx       ; rdx = 0 (KÖTELEZŐ! div előtt!)
    mov rcx, 2         ; osztó
    div rcx            ; rax = 70 / 2 = 35, rdx = 70 % 2 = 0

    ; Az eredmény rax-ban van (35), rdx-ben a maradék (0)

    ; === MARADÉK bemutatása egy másik példán ===
    ; 17 / 5 = ?
    mov rax, 17
    xor rdx, rdx
    mov rcx, 5
    div rcx            ; rax = 3 (hányados), rdx = 2 (maradék)
    ; Ezt most figyelmen kívül hagyjuk, de rdx = 2

    ; Eredmény: az eredeti 35 még rax-ban volt, de felülírtuk
    ; Szándékosan: most az exit kód az (10*7)/2 = 35 legyen
    ; Számítsuk újra kompaktan:
    mov rax, 10
    imul rax, rax, 7   ; rax = 70  (3 operandusú imul)
    xor rdx, rdx
    mov rcx, 2
    div rcx            ; rax = 35

    ; Kilépés
    mov rdi, rax       ; exit kód = 35
    mov rax, 60
    syscall
```

**Fordítás és futtatás:**

```bash
nasm -f elf64 -g -F dwarf szorzas_osztas.asm -o szorzas_osztas.o
ld szorzas_osztas.o -o szorzas_osztas
./szorzas_osztas
echo $?    # kimenet: 35
```

---

### Program 3: Bitmaszkolás példa

```nasm
; bitmaszk.asm – bitmanipuláció bemutatása
; Feladat: egy byte-ból kivonjuk a felső és alsó 4 bitet (nibble) külön-külön
; Bemenet: 0xAB = 171 (tizedesen)
;   Felső nibble: 0xA = 10
;   Alsó nibble:  0xB = 11
;   Eredmény: felső + alsó = 21  (exit kód)
;
; Fordítás:
;   nasm -f elf64 -g -F dwarf bitmaszk.asm -o bitmaszk.o
;   ld bitmaszk.o -o bitmaszk
;   ./bitmaszk
;   echo $?   ; → 21

section .text
    global _start

_start:
    mov rax, 0xAB      ; rax = 0b10101011 = 171

    ; === Alsó nibble (bit 0-3) ===
    mov rbx, rax       ; rbx = 0xAB (másolat)
    and rbx, 0x0F      ; rbx = 0x0B = 11  (felső 4 bitet lenullázzuk)

    ; === Felső nibble (bit 4-7) ===
    mov rcx, rax       ; rcx = 0xAB (másolat)
    shr rcx, 4         ; rcx = 0x0A = 10  (jobbra tolás 4-gyel)
    ; Most rcx = 10, rbx = 11

    ; === Összegzés ===
    add rbx, rcx       ; rbx = 11 + 10 = 21

    ; === Bit beállítás / törlés / tükrözés demonstrációja ===
    mov rax, 0b00001111   ; rax = 15

    or  rax, 0b00110000   ; rax = 0b00111111 = 63  (5. és 6. bit beállítása)
    and rax, 0b11111110   ; rax = 0b00111110 = 62  (0. bit törlése)
    xor rax, 0b00000010   ; rax = 0b00111100 = 60  (1. bit váltása)
    ; Ezt most nem az exit kódban adjuk vissza

    ; Exit: az első feladat eredménye (21)
    mov rdi, rbx       ; exit kód = 21
    mov rax, 60
    syscall
```

**Fordítás és futtatás:**

```bash
nasm -f elf64 -g -F dwarf bitmaszk.asm -o bitmaszk.o
ld bitmaszk.o -o bitmaszk
./bitmaszk
echo $?    # kimenet: 21
```

---

## 7. Gyors referencia táblázat

| Utasítás                | Mit csinál                                      | Java ekvivalens           |
|-------------------------|-------------------------------------------------|---------------------------|
| `mov rax, rbx`          | rax = rbx                                       | `a = b;`                  |
| `mov rax, [mem]`        | rax = *mem                                      | `a = mem;` (deref)        |
| `mov [mem], rax`        | *mem = rax                                      | `mem = a;`                |
| `movzx rax, bl`         | rax = (uint64_t)(uint8_t)bl                     | `a = b & 0xFF;`           |
| `movsx rax, bl`         | rax = (int64_t)(int8_t)bl                       | `a = (byte)b;`            |
| `add rax, rbx`          | rax += rbx                                      | `a += b;`                 |
| `sub rax, rbx`          | rax -= rbx                                      | `a -= b;`                 |
| `inc rax`               | rax++                                           | `a++;`                    |
| `dec rax`               | rax--                                           | `a--;`                    |
| `neg rax`               | rax = -rax                                      | `a = -a;`                 |
| `imul rax, rbx, 6`      | rax = rbx * 6                                   | `a = b * 6;`              |
| `div rbx`               | rax = rax/rbx, rdx = rax%rbx (unsigned)         | `q=a/b; r=a%b;`           |
| `xor rax, rax`          | rax = 0                                         | `a = 0;`                  |
| `and rax, 0x0F`         | rax &= 0x0F                                     | `a &= 0x0F;`              |
| `or  rax, 0x80`         | rax \|= 0x80                                    | `a \|= 0x80;`             |
| `xor rax, 0xFF`         | rax ^= 0xFF                                     | `a ^= 0xFF;`              |
| `shl rax, 3`            | rax <<= 3 (rax *= 8)                            | `a <<= 3;`                |
| `shr rax, 2`            | rax >>= 2 (unsigned)                            | `a >>>= 2;`               |
| `sar rax, 1`            | rax >>= 1 (signed)                              | `a >>= 1;`                |
| `lea rax, [rbx+rcx*4]`  | rax = rbx + rcx*4 (nincs memória hozzáférés!)   | `a = b + c*4;`            |

---

## 8. Gyakorlatok

### 1. feladat: Celsius → Fahrenheit konverzió

Konvertálj 25°C-t Fahrenheitbe az `(C * 9) / 5 + 32` képlettel. Az eredményt add vissza exit kódként (77 kell legyen).

**Megoldás vázlat:**
```nasm
mov rax, 25        ; C = 25
imul rax, rax, 9   ; rax = 25 * 9 = 225
xor rdx, rdx       ; rdx nullázása div előtt
mov rcx, 5
div rcx            ; rax = 225 / 5 = 45
add rax, 32        ; rax = 45 + 32 = 77
mov rdi, rax
mov rax, 60
syscall
```

---

### 2. feladat: Bitmaszk – páros/páratlan ellenőrzés

Írj programot, amely meghatározza, hogy egy szám páros-e! A szám legyen 17. Az exit kód: 1 ha páratlan, 0 ha páros.

**Megoldás vázlat:**
```nasm
mov rax, 17
and rax, 1         ; ha rax páratlan: rax = 1, ha páros: rax = 0
mov rdi, rax
mov rax, 60
syscall            ; exit kód: 1 (páratlan)
```

---

### 3. feladat: LEA trükk – szorzás 3-mal

Használj `lea`-t, hogy kiszámítsd `rbx * 3`-at anélkül, hogy `imul`-t használnál!

**Megoldás vázlat:**
```nasm
mov rbx, 14
lea rax, [rbx + rbx*2]   ; rax = rbx + rbx*2 = rbx*3 = 42
mov rdi, rax
mov rax, 60
syscall
```

---

### 4. feladat: Maradék kiszámítása

Számold ki a 100 mod 7 értékét! (Elvárt eredmény: 2)

**Megoldás vázlat:**
```nasm
mov rax, 100
xor rdx, rdx
mov rcx, 7
div rcx         ; rax = 14 (hányados), rdx = 2 (maradék)
mov rdi, rdx   ; a maradék az exit kód
mov rax, 60
syscall
```

---

### 5. feladat: MOVSX vs MOVZX különbség

Tölts be egy -10 értékű bájt értéket (`0xF6`) egy 64-bites regiszterbe kétféleképpen:
- `movsx`-szel: az eredmény legyen -10 (signed)
- `movzx`-szel: az eredmény legyen 246 (unsigned)

**Megoldás vázlat:**
```nasm
section .data
    bajt db 0xF6    ; -10 signed, 246 unsigned

section .text
    global _start
_start:
    movsx rax, byte [bajt]   ; rax = -10 (0xFFFFFFFFFFFFFFF6)
    movzx rbx, byte [bajt]   ; rbx = 246 (0x00000000000000F6)

    ; Ellenőrzés: rbx - rax = 246 - (-10) = 256
    sub rbx, rax              ; rbx = 256... de ez > 127, exit kódban csonkul!
    ; Helyett csak az exit kód alacsony 8 bitje számít:
    mov rdi, rbx
    mov rax, 60
    syscall
```

---

## 9. Ellenőrizd magad

**1. Miért nem lehet közvetlenül memóriából memóriába másolni?**

> Az x86-64 architektúrában az ALU (Arithmetic Logic Unit) csak regisztereken tud operálni. A memóriából való olvasás és a memóriába való írás külön buszműveletek, amelyek közé egy regiszternek kell kerülnie pufferkéntszámítási közvetítőként. Emellett az utasítás kódolása sem engedélyezi két memória operandust egyidejűleg (az opcode formátum korlátai).

---

**2. Mi a különbség a `shr` és a `sar` utasítások között?**

> `SHR` (Shift Right Logical): a felső bitpozíciót **nullával** tölti fel – előjeltől függetlenül. `SAR` (Shift Right Arithmetic): a felső bitpozíciót az **eredeti előjelbittel** tölti fel, így a negatív számok negatívak maradnak. Java-ban: `SHR` ↔ `>>>`, `SAR` ↔ `>>`.

---

**3. Miért kell `rdx`-et nullázni `div` előtt?**

> A `DIV` utasítás a **`rdx:rax` regiszterpárban** lévő 128-bites értéket osztja a megadott osztóval. Ha `rdx` nem nulla, az osztandó gigantikus lehet, és az eredmény nem fér el `rax`-ban → **Divide Overflow** kivétel keletkezik (általában SIGFPE jellel leáll a program).

---

**4. Mi a különbség a `mul` és az `imul` 2-operandusú változata között?**

> `MUL src`: mindig `rax`-szal szoroz, az eredmény a `rdx:rax` párba kerül (128 bites), **unsigned** szorzás. `IMUL dst, src` (2 op): `dst = dst * src`, az eredmény csonkítva `dst`-be kerül, **signed** szorzás, nem érinti `rdx`-et. Az `IMUL` rugalmasabb és a legtöbb esetben ezt használjuk.

---

**5. Mikor állítódik be az Overflow Flag (OF)?**

> Az OF akkor válik 1-esre, ha **előjeles** aritmetikai túlcsordulás történik: az eredmény nem fér el az adott bitmezőben előjeles értékként. Például két pozitív szám összeadásakor az eredmény negatív lesz (pl. `0x7FFF...FFFF + 1`), vagy két negatív szám összeadásakor pozitív lesz.

---

**6. Mit csinál a `lea rax, [rbx + rcx*4 + 8]` utasítás, és miben különbözik a `mov rax, [rbx + rcx*4 + 8]`-tól?**

> `LEA`: kiszámolja a `rbx + rcx*4 + 8` értéket és **beírja `rax`-ba** – **nem éri el a memóriát**, csak aritmetikát végez. `MOV`: kiszámolja ugyanezt a *memóriacímet*, majd **kiolvas 8 bájtot az adott memóriacímről** és azt írja `rax`-ba. A LEA-t hatékony összeadás+szorzás elvégzésére is használjuk, nem csak memóriacím számításra.

---

## Összefoglalás

| Témakör           | Kulcsfogalmak                                          |
|-------------------|--------------------------------------------------------|
| MOV               | regiszter↔regiszter, imm, memória; mem→mem tiltott     |
| MOVZX / MOVSX     | zero-extend (unsigned), sign-extend (signed)           |
| Aritmetika        | ADD, SUB, INC, DEC, NEG, IMUL, DIV, IDIV              |
| MUL / DIV         | rdx:rax pár, rdx nullázás kötelező                     |
| XOR reg, reg      | legjobb módszer regiszter nullázásra                   |
| RFLAGS            | CF, ZF, SF, OF – a következő nap feltételes ugrásoknál|
| Bitműveletek      | AND, OR, XOR, NOT, SHL, SHR, SAR                      |
| LEA               | cím kiszámítása memória-hozzáférés nélkül, aritmetikai trükk |

A **4. napon** ezekre az alapokra építve megismerjük a feltételes ugróutasításokat (`jmp`, `je`, `jne`, `jl`, `jg`...), amelyek a flag-ek alapján vezérlik a program futását – ezzel Assembly-ben is képesek leszünk `if/else` logikát és ciklusokat megvalósítani.
