# 4. nap: Vezérlési szerkezetek

> **Mit tanulunk ma?**
> Megismerjük az x86-64 Assembly vezérlési szerkezeteit: hogyan hasonlítunk össze értékeket (`cmp`, `test`), hogyan ugrik a program feltétel alapján (feltételes ugró utasítások, Jcc), és hogyan valósítjuk meg a Java `if-else`, `for`, `while` konstrukciókat Assembly-ben. Megismerkedünk a `loop` utasítással, a `cmovcc` feltételes MOV-val, és egy teljes példaprogrammal zárjuk a napot.

---

## Tartalomjegyzék

1. [Az összehasonlítás: CMP és TEST](#1-az-összehasonlítás-cmp-és-test)
2. [Feltétel nélküli ugrás](#2-feltétel-nélküli-ugrás)
3. [Feltételes ugró utasítások (Jcc)](#3-feltételes-ugró-utasítások-jcc)
4. [If-else megvalósítása Assembly-ben](#4-if-else-megvalósítása-assembly-ben)
5. [Ciklusok Assembly-ben](#5-ciklusok-assembly-ben)
6. [Beágyazott feltételek és ciklusok](#6-beágyazott-feltételek-és-ciklusok)
7. [CMOVcc – feltételes MOV](#7-cmovcc--feltételes-mov)
8. [Teljes példaprogram: szám kitalálós logika](#8-teljes-példaprogram-szám-kitalálós-logika)
9. [Gyakorlatok](#9-gyakorlatok)
10. [Ellenőrizd magad](#10-ellenőrizd-magad)

---

## 1. Az összehasonlítás: CMP és TEST

A Java `if (a > b)` feltétel Assembly-ben két lépésből áll:
1. Egy összehasonlító utasítás, amely beállítja a **RFLAGS** regiszter flag-jeit
2. Egy feltételes ugró utasítás, amely a flag-ek alapján dönt

### 1.1 A CMP utasítás

```
cmp a, b
```

A `cmp` elvégzi az `a - b` kivonást, de **az eredményt eldobja** – csak a flag-eket állítja be. Gondolj rá úgy, mint egy `sub` amelynek nincs visszatérési értéke.

| Flag | Mikor 1? |
|------|----------|
| **ZF** (Zero Flag) | Ha `a == b` (az eredmény nulla) |
| **SF** (Sign Flag) | Ha az eredmény negatív (MSB = 1) |
| **CF** (Carry Flag) | Ha unsigned alulcsordulás történt (borrow) |
| **OF** (Overflow Flag) | Ha signed túlcsordulás történt |

**Példa:**
```nasm
mov rax, 10
mov rbx, 7
cmp rax, rbx    ; 10 - 7 = 3 → ZF=0, SF=0, CF=0, OF=0
                ; rax és rbx VÁLTOZATLANOK maradnak
```

```nasm
mov rax, 5
mov rbx, 5
cmp rax, rbx    ; 5 - 5 = 0 → ZF=1 (egyenlők)
```

```nasm
mov rax, 3
mov rbx, 7
cmp rax, rbx    ; 3 - 7 = -4 → SF=1, CF=1 (unsigned esetben borrow)
```

> **💡 Tipp:** A `cmp` operandusai ugyanolyan kombinációban megengedettek, mint a `sub`-nál: reg,reg / reg,imm / mem,reg / reg,mem. Két memória operandus egyszerre nem lehetséges.

### 1.2 A TEST utasítás

```
test a, b
```

A `test` elvégzi az `a AND b` bitenkénti AND műveletet, de szintén **az eredményt eldobja** – csak ZF, SF, PF flag-eket állítja be. CF és OF mindig 0 lesz.

**Leggyakoribb használat: nulla-vizsgálat**

```nasm
test rax, rax   ; ha rax == 0, akkor ZF=1
jz  is_zero     ; ugrás ha rax == 0
```

Ez ekvivalens a `cmp rax, 0`-val, de **rövidebb és gyorsabb** – a processzor egy operandusból tudja elvégezni.

**Bit-vizsgálat:**
```nasm
test rax, 1     ; az LSB (bit 0) vizsgálata
jnz is_odd      ; ha nem nulla → rax páratlan
```

```nasm
test rax, 0x80  ; a 7. bit vizsgálata
jnz bit7_set    ; ha be van állítva
```

### 1.3 CMP vs TEST – mikor melyiket?

| Feladat | Ajánlott | Miért? |
|---------|----------|--------|
| Két értéket hasonlítunk össze | `cmp a, b` | Minden relációhoz (>, <, ==, !=) működik |
| Egy értéket 0-val hasonlítunk | `test rax, rax` | Rövidebb kód, ugyanolyan gyors |
| Visszatérési értéket vizsgálunk | `test rax, rax` + `jz` | Idiomatikus Assembly |
| Bit-maszkot vizsgálunk | `test rax, maszk` | AND pontosan ezt csinálja |
| Pointer null-vizsgálat | `test rdi, rdi` | Pointer == 0 → null |

---

## 2. Feltétel nélküli ugrás

### 2.1 JMP utasítás

```nasm
jmp label       ; közvetlen (direct) ugrás – a label-re
jmp rax         ; közvetett (indirect) ugrás – a rax-ban lévő címre
jmp [rax]       ; memória-indirekt ugrás – a rax által mutatott cím
```

A `jmp` a C `goto` megfelelője – feltétel nélkül átirányítja a vezérlést.

### 2.2 Előre és hátra ugrás

```nasm
section .text
global _start

_start:
    jmp elore       ; előre ugrás (forward jump) – még nem látott label felé

hatra:
    mov rax, 1      ; ezt hamarabb látja az assembler
    jmp vege

elore:
    mov rbx, 2
    jmp hatra       ; hátra ugrás (backward jump) – már látott label felé

vege:
    ; ...
```

Az assembler kétmenetes (two-pass): az első menetben begyűjti a label-eket, a másodikban generálja a gépi kódot, így mindkét irányú ugrás rendben működik.

### 2.3 Végtelen ciklus demonstráció

```nasm
; vegtelen.asm – Ctrl+C-vel állítható le (vagy GDB-vel)
section .text
global _start

_start:
    mov rcx, 0          ; számláló = 0

ciklus:
    inc rcx             ; rcx++
    jmp ciklus          ; vissza a ciklus elejére → végtelen ciklus
```

> **⚠️ Figyelem:** A valódi programokban a végtelen ciklust általában KERÜLJÜK. Egyetlen jogos alkalmazás: szerver event loop, amelyből explicit `jmp`-pel vagy `ret`-tel lépünk ki.

---

## 3. Feltételes ugró utasítások (Jcc)

A `Jcc` az összes feltételes ugró utasítás gyűjtőneve (cc = condition code). Mind `cmp` vagy `test` után használjuk.

### 3.1 Teljes táblázat

#### Egyenlőség

| Utasítás | Alternatíva | Ugrik ha | Flag feltétel | Java analógia |
|----------|-------------|----------|---------------|---------------|
| `je` | `jz` | a == b | ZF=1 | `if (a == b)` |
| `jne` | `jnz` | a != b | ZF=0 | `if (a != b)` |

#### Signed (előjeles) összehasonlítás – `int`, `long` típusokhoz

| Utasítás | Alternatíva | Ugrik ha | Flag feltétel | Java analógia |
|----------|-------------|----------|---------------|---------------|
| `jg` | `jnle` | a > b (signed) | ZF=0 AND SF=OF | `if (a > b)` |
| `jge` | `jnl` | a >= b (signed) | SF=OF | `if (a >= b)` |
| `jl` | `jnge` | a < b (signed) | SF≠OF | `if (a < b)` |
| `jle` | `jng` | a <= b (signed) | ZF=1 OR SF≠OF | `if (a <= b)` |

#### Unsigned (előjel nélküli) összehasonlítás – pointerekhez, byte értékekhez

| Utasítás | Alternatíva | Ugrik ha | Flag feltétel | Java analógia |
|----------|-------------|----------|---------------|---------------|
| `ja` | `jnbe` | a > b (unsigned) | CF=0 AND ZF=0 | nincs közvetlen Java analog |
| `jae` | `jnb` | a >= b (unsigned) | CF=0 | `>>>` unsigned shift analógia |
| `jb` | `jnae` | a < b (unsigned) | CF=1 | — |
| `jbe` | `jna` | a <= b (unsigned) | CF=1 OR ZF=1 | — |

#### Egyéb hasznos feltételek

| Utasítás | Ugrik ha | Tipikus használat |
|----------|----------|-------------------|
| `js` | SF=1 (negatív) | `if (result < 0)` |
| `jns` | SF=0 (nem negatív) | `if (result >= 0)` |
| `jo` | OF=1 (overflow) | Signed overflow detektálás |
| `jno` | OF=0 | — |
| `jc` | CF=1 (carry) | Unsigned overflow / borrow |
| `jnc` | CF=0 | — |
| `jp` | PF=1 (even parity) | Ritka, float NaN detektálás |

### 3.2 Signed vs Unsigned – mikor melyiket?

Ez az egyik leggyakoribb Assembly hiba-forrás. A Java-ban nincs unsigned integer (kivéve a Java 8+ `Integer.compareUnsigned()`), Assembly-ben viszont a CPU nem tudja, hogy az érték signed vagy unsigned – **te döntöd el**, melyik ugró utasítást használod.

```nasm
; Példa: -1 (signed) vs 0xFFFFFFFFFFFFFFFF (unsigned)
mov rax, -1         ; bináris: 1111...1111
mov rbx, 1

cmp rax, rbx

jg  signed_nagyobb  ; signed: -1 > 1? NEM – nem ugrik
ja  unsigned_nagy   ; unsigned: 0xFFFF...FFFF > 1? IGEN – ugrik!
```

**Ökölszabály:**
- **Signed** (`jg`, `jl`, `jge`, `jle`): egész számok, ahol negatív értékek lehetségesek (`int`, `long`)
- **Unsigned** (`ja`, `jb`, `jae`, `jbe`): pointerek, memóriacímek, `size_t`, byte értékek, bitmanipuláció

> **💡 Tipp:** A `je` / `jne` mindkét esetben ugyanaz – az egyenlőség nem függ az előjelezéstől.

---

## 4. If-else megvalósítása Assembly-ben

### 4.1 Java → Assembly fordítás: egyszerű if-else

**Java kód:**
```java
int a = 10, b = 7;
int result;
if (a > b) {
    result = a;
} else {
    result = b;
}
```

**Assembly megfelelő (NASM):**

```nasm
; if_else.asm
section .text
global _start

_start:
    mov rax, 10         ; rax = a = 10
    mov rbx, 7          ; rbx = b = 7

    cmp rax, rbx        ; a - b → flag-ek beállítása
    jle else_ag         ; ha a <= b → ugrik az else ágra
                        ; (jg ellentéte: ha NEM a>b, ugrunk)

; --- then ág: a > b ---
then_ag:
    mov rcx, rax        ; result = a
    jmp vege            ; kötelező: átugorjuk az else ágat!

; --- else ág ---
else_ag:
    mov rcx, rbx        ; result = b

; --- folytatás ---
vege:
    ; rcx most tartalmazza a max(a,b) értéket
    mov rax, 60         ; sys_exit
    mov rdi, rcx        ; kilépési kód = result (ellenőrzés: echo $?)
    syscall
```

**Fordítás és futtatás:**
```bash
nasm -f elf64 -g -F dwarf if_else.asm -o if_else.o
ld if_else.o -o if_else
./if_else
echo $?     # 10-et kell kapnunk (max(10, 7) = 10)
```

### 4.2 Lépésről lépésre magyarázat

```
1. mov rax, 10     →  rax = 10
2. mov rbx, 7      →  rbx = 7
3. cmp rax, rbx    →  10 - 7 = 3; ZF=0, SF=0, CF=0, OF=0
4. jle else_ag     →  ugrik-e? ZF=0 AND SF=OF(0=0) → NEM ugrik
                       (mert 10 > 7, tehát NEM "less or equal")
5. mov rcx, rax    →  rcx = 10   (then ág fut le)
6. jmp vege        →  átugorjuk az else ágat
   [else_ag:]      →  NEM fut le
7. [vege:]         →  rcx = 10
```

> **💡 Tipp:** Assembly-ben az `if` feltételét általában **megfordítva** írjuk: a `jle` az `if (a > b)` „ellentéte". Ha a feltétel HAMIS, ugrunk az `else` ágra. Ez az idiomatikus assembly stílus.

### 4.3 Java → Assembly: if-else if-else (switch-szerű)

**Java kód:**
```java
int szam = 5;
String eredmeny;
if (szam < 0)       eredmeny = "negativ";
else if (szam == 0) eredmeny = "nulla";
else                eredmeny = "pozitiv";
```

**Assembly:**
```nasm
; if_elif.asm
section .data
    msg_neg   db "negativ", 10, 0
    msg_nul   db "nulla", 10, 0
    msg_pos   db "pozitiv", 10, 0

section .text
global _start

_start:
    mov rax, 5          ; szam = 5

    cmp rax, 0
    jl  negativ         ; ha szam < 0
    je  nulla           ; ha szam == 0
    ; különben: pozitív

pozitiv:
    mov rsi, msg_pos
    jmp kiir

negativ:
    mov rsi, msg_neg
    jmp kiir

nulla:
    mov rsi, msg_nul

kiir:
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    mov rdx, 8          ; max hossz
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 5. Ciklusok Assembly-ben

### 5.1 While ciklus

**Java:**
```java
int i = 0;
while (i < 10) {
    // do something
    i++;
}
```

**Assembly (while minta):**
```nasm
    mov rcx, 0          ; i = 0

while_eleje:
    cmp rcx, 10         ; i < 10?
    jge while_vege      ; ha i >= 10 → kilépés

    ; === ciklus törzse ===
    inc rcx             ; i++

    jmp while_eleje     ; vissza az elejére

while_vege:
    ; folytatás...
```

**Jellemzők:**
- Az összehasonlítás a ciklus **elején** van (pre-test loop)
- Ha a feltétel kezdetben hamis, a törzs egyszer sem fut le (Java `while`-lal azonos viselkedés)
- `jge` a `< 10` ellentéte

### 5.2 For ciklus – 1-től 10-ig összegzés

**Java:**
```java
int sum = 0;
for (int i = 1; i <= 10; i++) {
    sum += i;
}
```

**Teljes Assembly program:**
```nasm
; osszeg.asm – 1-től 10-ig összegzés
; Eredmény: 55 (echo $? → 55)

section .text
global _start

_start:
    mov rax, 0          ; sum = 0
    mov rcx, 1          ; i = 1  (számláló)

for_eleje:
    cmp rcx, 10         ; i <= 10?
    jg  for_vege        ; ha i > 10 → kilépés

    add rax, rcx        ; sum += i

    inc rcx             ; i++
    jmp for_eleje

for_vege:
    ; rax = 55
    mov rdi, rax        ; kilépési kód = sum
    mov rax, 60         ; sys_exit
    syscall
```

**Fordítás és futtatás:**
```bash
nasm -f elf64 -g -F dwarf osszeg.asm -o osszeg.o
ld osszeg.o -o osszeg
./osszeg
echo $?     # 55
```

### 5.3 A LOOP utasítás

Az x86-64 rendelkezik egy beépített `loop` utasítással, amely a `rcx`-regisztert használja számlálóként:

```nasm
    mov rcx, 10         ; ciklusszám

ciklus:
    ; === ciklus törzse ===

    loop ciklus         ; rcx--; ha rcx != 0 → ugrás ciklus-ra
```

A `loop` három dolgot csinál egyszerre: `dec rcx` + `test rcx, rcx` + `jnz`.

**Ugyanaz a for ciklus `loop`-pal:**
```nasm
    mov rax, 0          ; sum = 0
    mov rbx, 1          ; aktuális szám (1-től indul)
    mov rcx, 10         ; 10 iteráció

ciklus:
    add rax, rbx        ; sum += rbx
    inc rbx             ; következő szám
    loop ciklus         ; rcx--; ha rcx != 0 → újra

    ; rax = 55
```

> **⚠️ Figyelem:** A `loop` utasítás modern processzorokon **lassabb** lehet, mint az explicit `dec rcx` + `jnz` kombináció! Ennek oka: az x86-64 pipeline optimalizálva van a `cmp`/`jcc` párosra, de a `loop`-ot kevésbé preferálja a branch predictor. Oktatási célból megmutatjuk, de éles kódban kerüld.

### 5.4 Do-while – a leghatékonyabb Assembly ciklus

**Java:**
```java
int i = 1, sum = 0;
do {
    sum += i;
    i++;
} while (i <= 10);
```

**Assembly (do-while minta):**
```nasm
    mov rax, 0          ; sum = 0
    mov rcx, 1          ; i = 1

do_while:
    add rax, rcx        ; sum += i  ← törzs ELŐSZÖR fut
    inc rcx             ; i++
    cmp rcx, 10         ; i <= 10?
    jle do_while        ; ha igen → ismét

    ; rax = 55
```

**Miért hatékonyabb?**
- Nincs szükség a ciklus elejére visszaugró `jmp`-re
- Az összehasonlítás a végén van → **egy ugrás kevesebb** ciklusonként
- A processzor branch predictora jobban kezeli a hátramutató feltételes ugrásokat
- A legtöbb compiler is do-while mintára fordítja a for/while ciklusokat!

### 5.5 Párhuzamos összefoglaló táblázat

| Java konstrukció | Assembly minta | Összehasonlítás helye | Ugrás iránya |
|-----------------|---------------|----------------------|--------------|
| `while (cond)` | cmp + Jcc (ha hamis, kiugrik) + jmp vissza | Elején | Előre (kilépés) + Hátra (vissza) |
| `for (init; cond; step)` | init + while minta | Elején | Előre + Hátra |
| `do { } while (cond)` | törzs + cmp + Jcc (ha igaz, vissza) | Végén | Hátra (vissza) |
| `loop` utasítás | rcx számlálóval | Végén (rcx != 0) | Hátra |

---

## 6. Beágyazott feltételek és ciklusok

### 6.1 If-else if-else lánc (switch analógia)

Az Assembly-ben a `switch`-szerű elágazást egymást követő `cmp` + `je` párokkal valósítjuk meg:

```nasm
; switch_pelda.asm
; Egy "note érték" (1-5) alapján szöveget ír ki
section .data
    msg1 db "Jeleses", 10
    len1 equ $ - msg1
    msg2 db "Jo", 10
    len2 equ $ - msg2
    msg3 db "Kozepes", 10
    len3 equ $ - msg3
    msg_default db "Ervenytelen", 10
    len_default equ $ - msg_default

section .text
global _start

_start:
    mov rbx, 4          ; jegy = 4 (ezt változtasd meg teszteléshez)

    cmp rbx, 5
    je  jeles

    cmp rbx, 4
    je  jo

    cmp rbx, 3
    je  kozepes

    jmp ervenytelen     ; default ág

jeles:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg1]
    mov rdx, len1
    syscall
    jmp vege

jo:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg2]
    mov rdx, len2
    syscall
    jmp vege

kozepes:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg3]
    mov rdx, len3
    syscall
    jmp vege

ervenytelen:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg_default]
    mov rdx, len_default
    syscall

vege:
    mov rax, 60
    xor rdi, rdi
    syscall
```

**Fordítás:**
```bash
nasm -f elf64 -g -F dwarf switch_pelda.asm -o switch_pelda.o
ld switch_pelda.o -o switch_pelda
./switch_pelda      # "Jo"
```

### 6.2 Ciklus break-kel (korai kilépés)

**Java:**
```java
int i = 0;
while (i < 100) {
    if (i * i > 50) break;  // korai kilépés
    i++;
}
```

**Assembly:**
```nasm
    mov rcx, 0          ; i = 0

while_start:
    cmp rcx, 100
    jge while_end       ; while feltétel: i < 100

    ; break feltétel: i*i > 50
    mov rax, rcx
    imul rax, rax       ; rax = i * i
    cmp rax, 50
    jg  while_end       ; break → kilép a ciklusból

    inc rcx             ; i++
    jmp while_start

while_end:
    ; rcx = az első i, amelyre i*i > 50 (vagy 100, ha nem volt break)
```

> **💡 Tipp:** A `break` Assembly megfelelője egyszerűen egy `jmp ciklus_vege` – ugrunk a ciklus utáni első utasításra.

### 6.3 Beágyazott ciklus

**Java:**
```java
// Szorzótábla 3×3 eredményeinek összegzése
int sum = 0;
for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
        sum += i * j;
    }
}
```

**Assembly:**
```nasm
; begyazott_ciklus.asm
section .text
global _start

_start:
    mov r8, 0           ; sum = 0
    mov rcx, 1          ; i = 1 (külső számláló)

kulso_ciklus:
    cmp rcx, 3
    jg  kulso_vege

    mov rdx, 1          ; j = 1 (belső számláló)

belso_ciklus:
    cmp rdx, 3
    jg  belso_vege

    ; sum += i * j
    mov rax, rcx        ; rax = i
    imul rax, rdx       ; rax = i * j
    add r8, rax         ; sum += i * j

    inc rdx             ; j++
    jmp belso_ciklus

belso_vege:
    inc rcx             ; i++
    jmp kulso_ciklus

kulso_vege:
    ; r8 = 36 (1*1 + 1*2 + 1*3 + 2*1 + 2*2 + 2*3 + 3*1 + 3*2 + 3*3)
    mov rdi, r8
    mov rax, 60
    syscall
```

> **⚠️ Figyelem:** Beágyazott ciklusoknál figyelj a **regiszterütközésre**! Ha a külső ciklus `rcx`-et használja, és belül egy `loop` utasítást akarsz, az felülírja `rcx`-et. Használj különböző regisztereket a különböző szinteken.

---

## 7. CMOVcc – feltételes MOV

### 7.1 Elágazás nélküli feltételes értékadás

A `CMOVcc` (Conditional MOVe) utasítások feltétel teljesülése esetén hajtanak végre egy `mov` műveletet – de **nem generálnak elágazást** a gépi kódban.

```nasm
cmove  dst, src    ; mov, ha ZF=1 (equal)
cmovne dst, src    ; mov, ha ZF=0
cmovg  dst, src    ; mov, ha greater (signed)
cmovl  dst, src    ; mov, ha less (signed)
cmovge dst, src    ; mov, ha greater or equal (signed)
cmovle dst, src    ; mov, ha less or equal (signed)
cmova  dst, src    ; mov, ha above (unsigned)
cmovb  dst, src    ; mov, ha below (unsigned)
```

### 7.2 Miért gyorsabb a JMP-nél?

Modern processzorokban **branch prediction** (elágazás-előrejelzés) van: a CPU megpróbálja kitalálni, melyik ágra ugrik a program, és előre elkezdi végrehajtani. Ha eltalálja → gyors. Ha nem → **pipeline flush** = 15-20 óraciklus büntetés.

A `cmovcc` nem igényel elágazás-előrejelzést, mert **mindig ugyanazt az utasítást hajtja végre** – feltétel esetén másolja az értéket, feltétel hiányában semmit sem csinál (de az utasítás végrehajtódik).

```
jmp alapú:                    cmov alapú:
  cmp rax, rbx                  cmp rax, rbx
  jle else_ag                   cmovg rcx, rax    ← mindig fut
  mov rcx, rax                  cmovle rcx, rbx   ← mindig fut
  jmp vege
else_ag:
  mov rcx, rbx
vege:
```

**Branch misprediction esetén**: a `jmp` változat 3-4x lassabb lehet véletlenszerű adatokon. A `cmov` változat konstans idejű.

### 7.3 Példa: max() és min() CMOVcc-vel

```nasm
; cmov_minmax.asm
; max(rax, rbx) és min(rax, rbx) kiszámítása

section .text
global _start

_start:
    mov rax, 42         ; a = 42
    mov rbx, 17         ; b = 17

    ; === max(a, b) ===
    mov rcx, rax        ; rcx = a (alapértelmezett: max = a)
    cmp rax, rbx        ; a vs b
    cmovl rcx, rbx      ; ha a < b → max = b
    ; rcx = 42 (max)

    ; === min(a, b) ===
    mov rdx, rax        ; rdx = a (alapértelmezett: min = a)
    cmp rax, rbx        ; a vs b
    cmovg rdx, rbx      ; ha a > b → min = b
    ; rdx = 17 (min)

    ; Kilépés: max értékkel
    mov rdi, rcx
    mov rax, 60
    syscall
```

**Fordítás:**
```bash
nasm -f elf64 -g -F dwarf cmov_minmax.asm -o cmov_minmax.o
ld cmov_minmax.o -o cmov_minmax
./cmov_minmax
echo $?     # 42 (max(42, 17))
```

> **💡 Tipp:** A GCC `-O2` szintjén szinte minden egyszerű `if`-et `cmov`-ra fordít, ha lehetséges. Az Assembly-ben ezt manuálisan kell megtenned – és érdemes, ha az adat véletlenszerű.

---

## 8. Teljes példaprogram: szám kitalálós logika

Ez a program összehasonlít egy beégetett "titkos számot" egy másik értékkel, és különböző üzeneteket ír ki az eredmény alapján (nagyobb / kisebb / egyenlő). Demonstrálja az összes mai tananyagot: `cmp`, feltételes ugrások, több ág.

```nasm
; talalat.asm
; Titkos szám: 42
; Tipp: 35 → "Tipp kisebb!"
;        42 → "Talalt!"
;        60 → "Tipp nagyobb!"

section .data
    titkos      dq 42           ; a titkos szám

    msg_kisebb  db "Tipp kisebb! Probald nagyobbal.", 10
    len_kisebb  equ $ - msg_kisebb

    msg_egyenlo db "Talalt! Gratulalok!", 10
    len_egyenlo equ $ - msg_egyenlo

    msg_nagyobb db "Tipp nagyobb! Probald kisebbel.", 10
    len_nagyobb equ $ - msg_nagyobb

section .text
global _start

_start:
    ; --- A tipp értéke (módosítsd és futtasd újra!) ---
    mov rax, 35             ; tipp = 35

    ; --- Betöltjük a titkos számot ---
    mov rbx, [titkos]      ; rbx = 42

    ; --- Összehasonlítás ---
    cmp rax, rbx            ; tipp - titkos

    ; --- Elágazás ---
    je  talalt              ; ha tipp == titkos
    jl  kisebb              ; ha tipp < titkos
    jg  nagyobb             ; ha tipp > titkos

; --- Ág: kisebb ---
kisebb:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg_kisebb]
    mov rdx, len_kisebb
    syscall
    jmp kilepes

; --- Ág: egyenlő ---
talalt:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg_egyenlo]
    mov rdx, len_egyenlo
    syscall
    jmp kilepes

; --- Ág: nagyobb ---
nagyobb:
    mov rax, 1
    mov rdi, 1
    lea rsi, [rel msg_nagyobb]
    mov rdx, len_nagyobb
    syscall

kilepes:
    mov rax, 60
    xor rdi, rdi
    syscall
```

**Fordítás és tesztelés:**
```bash
nasm -f elf64 -g -F dwarf talalat.asm -o talalat.o
ld talalat.o -o talalat
./talalat       # "Tipp kisebb! Probald nagyobbal."
```

**Módosítsd a tippet és futtasd újra:**
```bash
# A forrásban: mov rax, 42
# Újrafordítás után:
./talalat       # "Talalt! Gratulalok!"

# A forrásban: mov rax, 99
# Újrafordítás után:
./talalat       # "Tipp nagyobb! Probald kisebbel."
```

**Mi történik pontosan a `cmp rax, rbx` után?**

| tipp (rax) | titkos (rbx) | rax - rbx | ZF | SF | CF | Futó ág |
|-----------|--------------|-----------|----|----|-----|---------|
| 35 | 42 | -7 | 0 | 1 | 1 | `jl` → kisebb |
| 42 | 42 | 0 | 1 | 0 | 0 | `je` → talalt |
| 60 | 42 | +18 | 0 | 0 | 0 | `jg` → nagyobb |

> **💡 Tipp:** Éles alkalmazásban a titkos számot természetesen nem hardcode-oljuk, hanem memóriából vagy bemenetről olvassuk. Ez a pattern (cmp + három ág) azonban pontosan így jelenik meg minden összehasonlítás-alapú keresési algoritmusban (pl. bináris keresés).

---

## 9. Gyakorlatok

### 9.1 Abszolút érték (abs)

**Feladat:** Írj programot, amely kiszámítja egy előjeles egész abszolút értékét `cmov` használatával!

**Vázlat:**
```nasm
; abs.asm
section .text
global _start

_start:
    mov rax, -15        ; input érték

    ; Abszolút érték: ha rax < 0 → rax = -rax
    mov rbx, rax        ; rbx = rax (másolat)
    neg rbx             ; rbx = -rax
    test rax, rax       ; SF beállítása
    cmovs rax, rbx      ; ha SF=1 (negatív) → rax = -rax

    ; rax = 15
    mov rdi, rax
    mov rax, 60
    syscall
    ; echo $?  → 15
```

**Feladat:** Módosítsd úgy, hogy `jns` + `neg` párossal oldd meg `cmov` helyett, és hasonlítsd össze a két megközelítést!

---

### 9.2 Faktoriális számítás

**Feladat:** Számítsd ki az 5! = 120 értéket rekurzió nélkül, ciklussal!

**Vázlat:**
```nasm
; faktorial.asm
; n! = 1 * 2 * 3 * ... * n
section .text
global _start

_start:
    mov rcx, 5          ; n = 5
    mov rax, 1          ; eredmény = 1

ciklus:
    ; TODO: szorozzuk rax-ot rcx-szel,
    ; majd csökkentsük rcx-et,
    ; amíg rcx > 0
    imul rax, rcx       ; rax *= rcx
    dec rcx
    jnz ciklus         ; ha rcx != 0 → folytatás

    ; rax = 120
    mov rdi, rax
    mov rax, 60
    syscall
    ; echo $? → 120
```

---

### 9.3 Fibonacci-sorozat (N-edik elem)

**Feladat:** Számítsd ki a Fibonacci-sorozat 10. elemét (F(10) = 55) ciklussal!

**Vázlat:**
```nasm
; fibonacci.asm
; F(0)=0, F(1)=1, F(n) = F(n-1) + F(n-2)
section .text
global _start

_start:
    mov rcx, 10         ; N = 10
    mov rax, 0          ; F(0) = 0  → "előző előző"
    mov rbx, 1          ; F(1) = 1  → "előző"

    cmp rcx, 0
    je  vege_nulla
    cmp rcx, 1
    je  vege_egy

    sub rcx, 1          ; már 1 értékünk van (rbx = F(1))

ciklus:
    ; rdx = F(n) = F(n-1) + F(n-2) = rbx + rax
    mov rdx, rax
    add rdx, rbx
    mov rax, rbx        ; előre lépés: előző-előző = előző
    mov rbx, rdx        ; előre lépés: előző = new
    dec rcx
    jnz ciklus

    jmp vege

vege_nulla:
    xor rbx, rbx
    jmp vege
vege_egy:
    ; rbx már = 1

vege:
    mov rdi, rbx        ; F(10) = 55
    mov rax, 60
    syscall
    ; echo $? → 55
```

---

### 9.4 Legkisebb közös osztó keresése

**Feladat:** Keress prímszámokat 1 és 20 között! Adj vissza a kilépési kódban a talált prímek számát.

**Vázlat:**
```nasm
; primek.asm
; Számolj össze prímszámokat 2..20 között
; Kilépési kód = prímek száma (8 darab: 2,3,5,7,11,13,17,19)

section .text
global _start

_start:
    mov r8, 0           ; prímek száma
    mov r9, 2           ; aktuális vizsgált szám (2-től kezdjük)

kulso:
    cmp r9, 20
    jg  kulso_vege

    ; Próbáljuk elosztani 2-vel és 3-mal egyszerűsítve:
    ; (teljes megoldáshoz belső ciklus kell 2..sqrt(n)-ig)
    mov rax, r9
    mov rbx, 2
    xor rdx, rdx
    div rbx             ; rax = r9 / 2, rdx = r9 % 2
    cmp rdx, 0
    je  nem_prim        ; osztható 2-vel → nem prím (kivéve ha r9==2)

    cmp r9, 2
    je  prim            ; 2 maga prím

    ; (egyszerűsített – teljes implementáció belső ciklussal)
    ; TODO: implementáld a teljes prímtesztet!
prim:
    inc r8

nem_prim:
    inc r9
    jmp kulso

kulso_vege:
    mov rdi, r8
    mov rax, 60
    syscall
```

---

### 9.5 Egyszerű buborékrendezés (3 elem)

**Feladat:** Rendezz növekvő sorrendbe 3 egész számot csak `cmp` és `cmov` segítségével!

**Vázlat:**
```nasm
; buborek3.asm
; Rendezz: [7, 2, 5] → [2, 5, 7]
section .data
    tomb dq 7, 2, 5

section .text
global _start

_start:
    mov rax, [tomb]         ; a[0] = 7
    mov rbx, [tomb + 8]     ; a[1] = 2
    mov rcx, [tomb + 16]    ; a[2] = 5

    ; 1. pass: összehasonlít a[0] és a[1]
    ; ha a[0] > a[1] → csere
    cmp rax, rbx
    jle nincs_csere_01
    xchg rax, rbx           ; csere
nincs_csere_01:

    ; 2. pass: összehasonlít a[1] és a[2]
    cmp rbx, rcx
    jle nincs_csere_12
    xchg rbx, rcx
nincs_csere_12:

    ; 3. pass: ismét a[0] és a[1]
    cmp rax, rbx
    jle rendezve
    xchg rax, rbx

rendezve:
    ; rax=2, rbx=5, rcx=7
    mov rdi, rax            ; kilépési kód = legkisebb elem (2)
    mov rax, 60
    syscall
    ; echo $? → 2
```

---

## 10. Ellenőrizd magad

**1. Mi a különbség a `cmp` és a `test` utasítás között?**

<details>
<summary>Válasz</summary>

A `cmp a, b` az `a - b` kivonást végzi el és a flag-eket állítja be (ZF, SF, CF, OF), de az eredményt eldobja. A `test a, b` az `a AND b` bitenkénti AND műveletet végzi és csak ZF, SF, PF flag-eket állítja be (CF és OF mindig 0 lesz). A `test rax, rax` idiomatikusan nulla-vizsgálatra használatos (hatékonyabb mint `cmp rax, 0`), a `test rax, maszk` bitmaszk vizsgálatára.

</details>

---

**2. Mi a különbség a `jg` és `ja` utasítás között? Mikor melyiket kell használni?**

<details>
<summary>Válasz</summary>

A `jg` (jump if greater) **signed** (előjeles) összehasonlítás eredményeképpen ugrik: SF=OF ÉS ZF=0 feltétel esetén. A `ja` (jump if above) **unsigned** (előjel nélküli) összehasonlításhoz való: CF=0 ÉS ZF=0 esetén ugrik. Ha pl. `rax = -1` (0xFFFF...FFFF) és `rbx = 1` után `cmp rax, rbx`-et végzünk: `jg` NEM ugrik (-1 < 1 signed), de `ja` UGRIK (0xFFFF...FFFF > 1 unsigned). Java int/long típusoknál mindig `jg`/`jl`/`jge`/`jle` használandó, pointereknél és size_t értékeknél `ja`/`jb`/`jae`/`jbe`.

</details>

---

**3. Miért kell a `then` ág végére `jmp vege` ugró utasítást írni egy if-else szerkezetben?**

<details>
<summary>Válasz</summary>

Mert Assembly-ben a végrehajtás lineárisan folytatódik tovább, nincs automatikus blokk-határoló (mint a Java kapcsos zárójele). Ha a `then` ág lefut és nincs `jmp`, a CPU egyszerűen "belesétál" az `else` ágba is, és mindkét ágat végrehajtja. A `jmp vege` átugorja az `else` ágat, így csak az egyik ág fut le.

</details>

---

**4. Mi a `loop` utasítás hátránya modern x86-64 processzorokon?**

<details>
<summary>Válasz</summary>

A `loop` utasítás (`dec rcx` + `jnz`) modern processzorokon lassabb lehet, mint az explicit `dec rcx` + `jnz` kombináció. Ennek oka, hogy a CPU branch predictor és pipeline az explicit `cmp`/`jcc` párosra van optimalizálva, a `loop`-ot nem részesíti előnyben. Ezért éles kódban kerülendő – a legtöbb modern assembler-oktatás csak történelmi érdekességként mutatja be.

</details>

---

**5. Mit jelent az, hogy a do-while ciklus "hatékonyabb" Assembly-ben?**

<details>
<summary>Válasz</summary>

A do-while minta esetén az összehasonlítás a ciklus végén van, így egy `jmp`-pel kevesebb szükséges ciklusonként. A while/for mintánál: minden iteráció `jcc` (ha hamis, kilép) + törzs + `jmp` (vissza) = 2 ugrás. A do-while mintánál: törzs + `jcc` (ha igaz, visszaugrik) = 1 ugrás. Ezért a GCC `-O2` szintjén szinte mindig do-while mintára fordít.

</details>

---

**6. Mikor érdemes `cmovcc`-t használni `jcc` helyett?**

<details>
<summary>Válasz</summary>

A `cmovcc` akkor előnyös, ha az adat **véletlenszerű** vagy **megjósolhatatlan** (pl. rendezetlen tömbök elemei, felhasználói input), mert kiküszöböli a branch misprediction büntetést (15-20 ciklus). Ha az elágazás eredménye **megjósolható** (pl. egy ciklus, amely mindig 999-szer fut tovább és egyszer lép ki), akkor a `jcc` is hatékony, sőt jobb lehet, mert a `cmovcc` mindig olvas memóriából, még akkor is, ha az eredményt végül nem használja fel. A `cmovcc` hátránya: nem használható memóriaírásra (csak olvasásra), és nem alkalmazható, ha az egyik ágnak mellékhatásai vannak (pl. függvényhívás).

</details>

---

## Összefoglalás

| Fogalom | Assembly megvalósítás | Java analógia |
|---------|----------------------|---------------|
| Összehasonlítás | `cmp a, b` | `a - b` (flag-ekhez) |
| Nulla-vizsgálat | `test rax, rax` + `jz` | `if (x == 0)` |
| If-else | `cmp` + `jcc` (ellentétes) + `jmp vege` | `if (...) { } else { }` |
| While ciklus | `cmp` + `jge vege` + törzs + `jmp eleje` | `while (cond) { }` |
| For ciklus | init + while minta | `for (init; cond; step) { }` |
| Do-while | törzs + `cmp` + `jcc eleje` | `do { } while (cond)` |
| Switch | egymás utáni `cmp` + `je` párok | `switch (x) { case ... }` |
| Break | `jmp ciklus_vege` | `break;` |
| Continue | `jmp ciklus_eleje` | `continue;` |
| Feltételes értékadás | `cmp` + `cmovcc` | `result = cond ? a : b` |
| Elágazásmentes max | `cmp` + `cmovl` | `Math.max(a, b)` |

**Következő fejezet:** [5. nap – Stack és függvények: push/pop, call/ret, calling convention, stack frame](../nap-05/README.md)
