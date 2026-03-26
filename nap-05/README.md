# 5. nap: Stack és függvények

> **Mit tanulunk ma?**
> Megismerjük a stack (verem) adatszerkezetet és azt, hogyan kezeli az x86-64 architektúra a függvényhívásokat. Megtanuljuk a `push`/`pop`, `call`/`ret` utasításokat, a System V AMD64 ABI calling convention-t, a stack frame felépítését, és két klasszikus rekurzív/iteratív algoritmust implementálunk Assembly-ben. A Java párhuzamok különösen erősek lesznek ma: amit a JVM automatikusan csinál, azt itt mi kézzel végezzük el.

---

## Tartalom

1. [A stack (verem) fogalma](#1-a-stack-verem-fogalma)
2. [Push és Pop utasítások](#2-push-és-pop-utasítások)
3. [Függvényhívás alapjai – call és ret](#3-függvényhívás-alapjai--call-és-ret)
4. [System V AMD64 ABI Calling Convention](#4-system-v-amd64-abi-calling-convention)
5. [Stack frame felépítése](#5-stack-frame-felépítése)
6. [Teljes példa: faktoriális rekurzívan](#6-teljes-példa-faktoriális-rekurzívan)
7. [Teljes példa: Fibonacci iteratívan](#7-teljes-példa-fibonacci-iteratívan)
8. [Több függvény egy programban](#8-több-függvény-egy-programban)
9. [Gyakorlatok](#9-gyakorlatok)
10. [Ellenőrizd magad](#10-ellenőrizd-magad)

---

## 1. A stack (verem) fogalma

### Mi az a stack?

A stack egy **LIFO** (Last In, First Out – utoljára be, először ki) adatszerkezet. Képzeld el, mint egy tálcaoszlop egy étteremben: mindig a legfelső tálcát veszed le, és mindig a tetejére raksz újat.

**Java analógia:** Amikor Java-ban egy metódust hívasz, a JVM automatikusan létrehoz egy *stack frame*-et a hívási veremben (call stack). Az `Exception` stack trace-ben is látod ezt: minden egymásba ágyazott metódushívás egy-egy réteg a veremben. Assembly-ben ugyanez történik, csak mi csináljuk kézzel.

### Az RSP regiszter

Az `RSP` (Register Stack Pointer) regiszter **mindig a stack tetejére mutat** – pontosabban az utoljára érvényesen elhelyezett értékre.

> **⚠️ Figyelem:** A stack **lefelé nő** a memóriában! Ez azt jelenti, hogy amikor valamit a stack-re teszünk, az RSP értéke **csökken** (alacsonyabb memóriacímet vesz fel). Ez sokaknak intuitív-ellenes, de az x86-64 architektúra így működik – és minden más modern architektúra is hasonlóan.

### Vizuális ábra – hogyan nő a stack lefelé?

```
Memóriában lévő címek (illusztráció):

Magas cím
┌─────────────────────────────┐  0xFFFFFF00
│  (program indításakor RSP   │
│   ide mutat kb.)            │
├─────────────────────────────┤  0xFFFFFEF8  ◄── RSP (üres stack)
│  [szabad terület]           │
│  ...                        │
│  ...                        │
├─────────────────────────────┤  0xFFFFFEF0  ◄── RSP (1 push után)
│  push rax → ide kerül rax   │
├─────────────────────────────┤  0xFFFFFEE8  ◄── RSP (2 push után)
│  push rbx → ide kerül rbx   │
├─────────────────────────────┤  0xFFFFFEE0  ◄── RSP (3 push után)
│  push rcx → ide kerül rcx   │
│  ...                        │
│  (stack tovább nő lefelé)   │
│  ...                        │
├─────────────────────────────┤
│  [program kód, adatok]      │
└─────────────────────────────┘  0x00000000
Alacsony cím
```

**Összefoglalás:**
- Stack teteje = RSP által mutatott cím (a **legkisebb** cím, amit a stack foglal)
- `push` → RSP csökken → stack nő lefelé
- `pop` → RSP nő → stack csökken (felszabadul a hely)

---

## 2. Push és Pop utasítások

### `push src`

```
push rax
```

Ez pontosan ezt csinálja:
```
sub rsp, 8        ; RSP csökkentése 8 bájttal
mov [rsp], rax    ; az érték elmentése a stack tetejére
```

Tehát `push` = helyfoglalás + értékmásolás. Az `src` lehet regiszter, közvetlen érték (immediate) vagy memóriacím.

### `pop dst`

```
pop rbx
```

Ez pontosan ezt csinálja:
```
mov rbx, [rsp]    ; érték visszaolvasása a stack tetejéről
add rsp, 8        ; RSP növelése 8 bájttal (hely "felszabadul")
```

> **💡 Tipp:** A `pop` nem törli a memóriából az értéket – csupán az RSP-t mozgatja. Az adat ott marad, de a következő `push` felül fogja írni. Ez fontos GDB-s debuggolásnál: ne lepődjön meg, ha `pop` után a régi érték még látható a memóriában.

### Példa: regiszter mentése és visszaállítása

Ha egy függvényt használunk, és a hívott kódnak nem szabad felülírnia bizonyos regisztereket (callee-saved), el kell mentenünk őket:

```nasm
; Regisztermentés push/pop-pal
section .text
global _start

_start:
    mov rax, 42         ; rax = 42
    mov rbx, 100        ; rbx = 100

    push rax            ; rax elmentése a stack-re
    push rbx            ; rbx elmentése a stack-re

    ; ... itt valami kód, ami esetleg felülírja rax-ot és rbx-et ...
    mov rax, 0          ; rax "felülírva"
    mov rbx, 0          ; rbx "felülírva"

    pop rbx             ; rbx visszaállítva = 100  (LIFO: rbx-et tettük be másodiknak → vesszük ki először)
    pop rax             ; rax visszaállítva = 42

    ; rax = 42, rbx = 100 újra
    mov rdi, rax        ; exit kód = 42
    mov rax, 60
    syscall
```

> **⚠️ Figyelem:** A `push`/`pop` sorrendje fordított! Ha `push rax`, `push rbx` sorrendben tettél be, akkor `pop rbx`, `pop rax` sorrendben kell kivenni. Ha elrontod a sorrendet, a regisztereid rossz értéket kapnak – és ez az egyik leggyakoribb assembly hiba.

### Stack alignment – 16 bájtos igazítás

Az x86-64 System V ABI **megköveteli**, hogy a stack pointer **16 bájtra igazítva** legyen (`rsp % 16 == 0`) minden `call` utasítás előtt. Pontosabban: a `call` utasítás végrehajtásakor `rsp` 16 bájtra igazított kell legyen, de mivel a `call` maga 8 bájtot tesz a stack-re (visszatérési cím), a hívott függvény elejére érve `rsp % 16 == 8`.

**Miért fontos ez?**
- Az SSE/AVX lebegőpontos utasítások (`movaps`, `movdqa` stb.) memória-operandusai 16 bájtos igazítást igényelnek
- Ha az igazítás rossz, **szegmentációs hiba** (SIGSEGV) keletkezhet, sokszor teljesen véletlenszerűnek tűnő helyeken – például egy `printf` hívásban, amelyet te nem is írtál
- A Linux kernel gondoskodik arról, hogy a program belépési pontján (`_start`) az RSP 16 bájtra igazított legyen

```
Program indulásakor:  rsp % 16 == 0  ✓
main()-be call-al:    rsp % 16 == 8  (mert a call push-olta a visszatérési címet)
prologue push rbp:    rsp % 16 == 0  ✓ (újra igazított)
```

---

## 3. Függvényhívás alapjai – call és ret

### `call label`

```
call my_function
```

Ez két lépést csinál egyszerre:
```
push rip           ; a következő utasítás (visszatérési cím) a stack-re kerül
jmp  my_function   ; ugrás a függvényhez
```

Az RIP (Instruction Pointer) a `call` utáni következő utasítás címét tartalmazza – ez lesz a visszatérési cím.

### `ret`

```
ret
```

Ez egyszerűen:
```
pop rip            ; visszatérési cím visszaállítása a stack-ről
```

Ezután a CPU folytatja a hívás utáni utasítással.

### Egyszerű függvény példa

```nasm
; fuggveny_pelda.asm
; Demonstrálja a call/ret működését
; Fordítás:
;   nasm -f elf64 -g -F dwarf fuggveny_pelda.asm -o fuggveny_pelda.o
;   ld fuggveny_pelda.o -o fuggveny_pelda
;   ./fuggveny_pelda
;   echo $?   → 7

section .data
    msg db "Fuggvenybol hello!", 0x0A
    msg_len equ $ - msg

section .text
global _start

; Ez a függvényünk: kiírja az üzenetet
print_hello:
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    mov rsi, msg        ; buffer pointer
    mov rdx, msg_len    ; hossz
    syscall
    ret                 ; visszatérés a hívóhoz

_start:
    call print_hello    ; hívjuk a függvényt

    ; A call után itt folytatódik a végrehajtás
    mov rax, 60         ; sys_exit
    mov rdi, 7          ; exit kód = 7
    syscall
```

**Mi történik pontosan a `call print_hello` során?**

```
Hívás előtt:
  RSP → [... üres ...]

call print_hello végrehajtásakor:
  RSP - 8 → [visszatérési cím: "_start következő utasítása"]
  RIP ← print_hello címe

print_hello belsejében:
  [syscall, stb.]

ret végrehajtásakor:
  RIP ← pop [RSP]   (visszatérési cím kivéve)
  RSP + 8           (stack "helyreáll")

Visszatérés után:
  A végrehajtás az _start-ban a call utáni mov rax, 60-nál folytatódik
```

---

## 4. System V AMD64 ABI Calling Convention

Az ABI (Application Binary Interface) meghatározza, hogyan kommunikálnak egymással a függvények bináris szinten. A Linux x86-64 rendszereken a **System V AMD64 ABI** az érvényes. Ha ezt betartod, az Assembly kódod képes lesz C könyvtárakat hívni és C kóddal együttműködni.

**Java analógia:** Ahogy a JVM belső method invocation protokollja definiálja, hogyan adódnak át az argumentumok (stack-en, vagy regisztereken), úgy definiálja az ABI is, hogyan kell Assembly szinten paramétereket átadni.

### Argumentum átadás – egész számok és pointerek

| Argumentum sorszám | Regiszter |
|--------------------|-----------|
| 1. argumentum      | `rdi`     |
| 2. argumentum      | `rsi`     |
| 3. argumentum      | `rdx`     |
| 4. argumentum      | `rcx`     |
| 5. argumentum      | `r8`      |
| 6. argumentum      | `r9`      |
| 7+. argumentum     | stack-en  |

### Visszatérési érték

- Egész szám / pointer: **`rax`**
- 128 bites visszatérés: `rax` (alacsony 64 bit) + `rdx` (magas 64 bit)
- Lebegőpontos: `xmm0`

### Caller-saved vs. Callee-saved regiszterek

Ez a legfontosabb tábla – könyv nélkül is érdemes megjegyezni:

| Kategória | Regiszterek | Mit jelent? |
|-----------|-------------|-------------|
| **Caller-saved** (scratch) | `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`, `r9`, `r10`, `r11` | A hívott függvény **felülírhatja** ezeket! Ha a hívó kódnak kell az értékük, **a hívónak** kell elmenteni |
| **Callee-saved** (preserved) | `rbx`, `rbp`, `r12`, `r13`, `r14`, `r15`, `rsp` | Ha a hívott függvény használja ezeket, **a hívottnak** kell elmenteni (push) és visszaállítani (pop) |

**Emlékezeti szabály:**
- *Caller-saved* = "Ha fontosak nekem, én mentem el, mielőtt call-olok"
- *Callee-saved* = "Ha használom, én mentem el, és én állítom vissza"

```
Java analógia:
  caller-saved ~ volatile változók: nem garantált az értékük megőrzése
  callee-saved ~ a metódus lokális változói: a metódus felelős a saját "rendjéért"
```

### Miért fontos ezt betartani?

1. **C könyvtár kompatibilitás:** Ha `printf`-et vagy más libc függvényt hívsz, pontosan ezt a konvenciót várja
2. **Konzisztencia:** Csapatban dolgozva, vagy amikor az Assembly kódat gcc/clang fordított kóddal kombinálod, az ABI a "közös nyelv"
3. **Debugolhatóság:** A GDB és más eszközök feltételezik az ABI betartását a stack trace-ek megjelenítésekor

---

## 5. Stack frame felépítése

### Function Prologue (függvénybelépő)

Minden "komoly" függvény elején ezeket az utasításokat látod:

```nasm
my_function:
    push rbp            ; az előző frame pointer elmentése
    mov  rbp, rsp       ; rbp = rsp (az aktuális stack teteje = a frame alapja)
    sub  rsp, 32        ; helyfoglalás a lokális változóknak (pl. 4 db 8 bájtos változó)
```

Az `rbp` (Base Pointer) regiszter **stabil referencia pontként** szolgál a stack frame-en belül:
- A lokális változók `[rbp - N]` címeken érhetők el
- A paraméterek (ha stack-en vannak) `[rbp + N]` címeken érhetők el
- Míg az `rsp` állandóan mozog (push/pop-ok miatt), az `rbp` fix marad a függvény futása alatt

### Function Epilogue (függvénykilépő)

```nasm
    ; 1. módszer (kézzel):
    mov  rsp, rbp       ; rsp visszaállítása (lokális változók "felszabadítása")
    pop  rbp            ; előző rbp visszaállítása
    ret                 ; visszatérés

    ; 2. módszer (leave utasítással – tömörebb):
    leave               ; egyenértékű: mov rsp, rbp + pop rbp
    ret
```

### Stack frame diagram két egymásba ágyazott hívással

```
Magas cím
┌─────────────────────────────────┐
│  ...caller stack frame...       │
├─────────────────────────────────┤ ◄── caller rbp
│  [caller lokális változók]      │
│  [caller lokális változók]      │
├─────────────────────────────────┤
│  VISSZATÉRÉSI CÍM               │ ← call utasítás ide push-olta
├─────────────────────────────────┤ ◄── callee rbp (push rbp utáni érték)
│  mentett rbp (caller rbp-je)    │ ← push rbp
├─────────────────────────────────┤
│  [lokális változó #1]  [rbp-8]  │
├─────────────────────────────────┤
│  [lokális változó #2]  [rbp-16] │
├─────────────────────────────────┤
│  [lokális változó #3]  [rbp-24] │
├─────────────────────────────────┤ ◄── rsp (sub rsp, N után)
│  [szabad terület]               │
│  ...                            │
Alacsony cím
```

### Lokális változók elérése

```nasm
my_function:
    push rbp
    mov  rbp, rsp
    sub  rsp, 24        ; 3 db 8 bájtos lokális változó helye

    ; Lokális változók:
    mov qword [rbp - 8],  10   ; lokális változó #1 = 10
    mov qword [rbp - 16], 20   ; lokális változó #2 = 20
    mov qword [rbp - 24], 30   ; lokális változó #3 = 30

    ; Olvasás lokális változóból:
    mov rax, [rbp - 8]         ; rax = 10
    add rax, [rbp - 16]        ; rax = 30
    ; rax most 30 (a visszatérési értékünk)

    leave
    ret
```

> **💡 Tipp:** A `sub rsp, N` értékét mindig **16 bájt többszörösére** kerekítsd fel, hogy a stack alignment megmaradjon! Ha 3 db 8 bájtos változó kell (24 bájt), és a prologue előtt az RSP 16 bájtra volt igazítva, akkor a `push rbp` után 8 bájt az RSP shift, így `sub rsp, 24`-gyel RSP shift összesen 32 bájt lesz, ami 16 bájt többszöröse. Általában biztonságos, ha 16 bájt többszörösével csökkented.

---

## 6. Teljes példa: faktoriális rekurzívan

### Java referenciakód

```java
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```

### Assembly megvalósítás – lépésről lépésre

**Stratégia:**
- Az argumentum `n` az `rdi` regiszterben érkezik (1. argumentum, System V ABI)
- A visszatérési érték `rax`-ban lesz
- Rekurzív hívás előtt `n`-t el kell menteni (caller-saved `rdi` helyett, a rekurzív hívás felülírja!), ezért a stack-re mentjük

```nasm
; faktorial.asm
; Rekurzív faktoriális számítás
; Fordítás:
;   nasm -f elf64 -g -F dwarf faktorial.asm -o faktorial.o
;   ld faktorial.o -o faktorial
;   ./faktorial
;   echo $?   → 120  (5! = 120)

section .text
global _start

; factorial(n): n! kiszámítása
; Bemenet:  rdi = n
; Kimenet:  rax = n!
; Módosítja: rax (visszatérési érték)
factorial:
    push rbp                ; --- PROLOGUE ---
    mov  rbp, rsp
    sub  rsp, 16            ; helyfoglalás: [rbp-8] = elmentett rdi (n értéke)
                            ; (16-ra kerekítve az alignment miatt)

    ; Alapeset: ha n <= 1, return 1
    cmp  rdi, 1
    jg   .recursive_case    ; ha n > 1, rekurzív eset

    ; Alapeset: return 1
    mov  rax, 1
    leave
    ret

.recursive_case:
    ; El kell menteni n-t, mert a call felülírja rdi-t
    mov  [rbp - 8], rdi     ; n elmentése lokális változóba

    ; factorial(n - 1) hívása
    dec  rdi                ; rdi = n - 1
    call factorial          ; rax = factorial(n-1)

    ; visszatérés után: rax = factorial(n-1), [rbp-8] = n
    imul rax, [rbp - 8]     ; rax = n * factorial(n-1)

    leave                   ; --- EPILOGUE ---
    ret

_start:
    mov  rdi, 5             ; n = 5
    call factorial          ; rax = 5! = 120

    mov  rdi, rax           ; exit kód = factorial(5)
    mov  rax, 60            ; sys_exit
    syscall
```

### Stack állapot nyomon követése – 5! rekurzió

```
factorial(5) hívja factorial(4) → factorial(3) → factorial(2) → factorial(1)

Stack (leegyszerűsítve, Magas cím felül):

_start call factorial(5):
├─ [ret addr → _start]     ← call push-olta
├─ rbp (_start frame)
└─ n=5 mentve [rbp-8]
        │
        ▼ call factorial(4)
        ├─ [ret addr → factorial(5)]
        ├─ rbp (factorial(5) frame)
        └─ n=4 mentve [rbp-8]
                │
                ▼ call factorial(3)
                ├─ [ret addr → factorial(4)]
                ├─ rbp (factorial(4) frame)
                └─ n=3 mentve [rbp-8]
                        │
                        ▼ call factorial(2)
                        ├─ [ret addr → factorial(3)]
                        ├─ rbp (factorial(3) frame)
                        └─ n=2 mentve [rbp-8]
                                │
                                ▼ call factorial(1)
                                ├─ [ret addr → factorial(2)]
                                ├─ rbp (factorial(2) frame)
                                └─ (alapeset: rax=1, ret)

Visszafelé:
  factorial(1) ret → rax = 1
  factorial(2): rax = 2 * 1 = 2, ret
  factorial(3): rax = 3 * 2 = 6, ret
  factorial(4): rax = 4 * 6 = 24, ret
  factorial(5): rax = 5 * 24 = 120, ret
```

> **⚠️ Figyelem:** A `rdi` regiszter **caller-saved** (scratch), vagyis a `call factorial` felülírja! Ezért mentjük el a lokális `[rbp-8]` változóba **a hívás előtt**. Ha ezt elfelejtjük, `n` értéke elvész, és a szorzás rossz eredményt ad.

---

## 7. Teljes példa: Fibonacci iteratívan

A Fibonacci sorozat: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, ...

**Java referenciakód (iteratív):**
```java
int fibonacci(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}
```

```nasm
; fibonacci.asm
; Iteratív Fibonacci szám kiszámítása
; Az n-edik Fibonacci szám kiszámítása, exit kóddal visszaadva
; Fordítás:
;   nasm -f elf64 -g -F dwarf fibonacci.asm -o fibonacci.o
;   ld fibonacci.o -o fibonacci
;   ./fibonacci
;   echo $?   → 55  (fib(10) = 55)

section .text
global _start

; fibonacci(n): n-edik Fibonacci szám
; Bemenet:  rdi = n  (0-indexed)
; Kimenet:  rax = fib(n)
fibonacci:
    push rbp
    mov  rbp, rsp

    ; Alapesetek
    cmp  rdi, 0
    je   .return_zero       ; fib(0) = 0
    cmp  rdi, 1
    je   .return_one        ; fib(1) = 1

    ; Iteratív számítás
    ; rax = a = 0  (fib(i-2))
    ; rbx = b = 1  (fib(i-1))
    ; rcx = számláló i = 2..n
    ; rdx = tmp

    ; rbx callee-saved → elmentjük
    push rbx

    mov  rax, 0             ; a = fib(0) = 0
    mov  rbx, 1             ; b = fib(1) = 1
    mov  rcx, 2             ; i = 2

.loop:
    cmp  rcx, rdi           ; i <= n ?
    jg   .done              ; ha i > n, kilépünk

    mov  rdx, rax           ; tmp = a
    add  rdx, rbx           ; tmp = a + b
    mov  rax, rbx           ; a = b
    mov  rbx, rdx           ; b = tmp

    inc  rcx                ; i++
    jmp  .loop

.done:
    mov  rax, rbx           ; visszatérési érték = b = fib(n)
    pop  rbx                ; rbx visszaállítása (callee-saved!)
    pop  rbp
    ret

.return_zero:
    mov  rax, 0
    pop  rbp
    ret

.return_one:
    mov  rax, 1
    pop  rbp
    ret

_start:
    mov  rdi, 10            ; n = 10
    call fibonacci          ; rax = fib(10) = 55

    mov  rdi, rax           ; exit kód = fib(10)
    mov  rax, 60
    syscall
```

> **💡 Tipp:** Az `rbx` regiszter **callee-saved**, ezért a `fibonacci` függvénynek el kell mentenie, mielőtt használja, és vissza kell állítania a `ret` előtt. Ha ezt elfelejtjük, a hívó kód (esetünkben `_start`) azt hinné, hogy `rbx` értéke változatlan maradt – és tönkremenne a logikája.

**Ellenőrzés:**
```bash
./fibonacci
echo $?   # → 55
```

---

## 8. Több függvény egy programban

### Hívási lánc: main → func1 → func2

Ez a példa demonstrálja, hogyan adódnak át argumentumok és hogyan mentjük a regisztereket egy háromszintű hívási láncban.

```nasm
; lancolt_fuggvenyek.asm
; Demonstrálja a main→func1→func2 hívási láncot
; Fordítás:
;   nasm -f elf64 -g -F dwarf lancolt_fuggvenyek.asm -o lancolt_fuggvenyek.o
;   ld lancolt_fuggvenyek.o -o lancolt_fuggvenyek
;   ./lancolt_fuggvenyek
;   echo $?   → 42

section .data
    msg1 db "func1 fut, arg: ", 0
    msg2 db "func2 fut, arg: ", 0

section .text
global _start

; func2(x): visszaad x * 2
; Bemenet:  rdi = x
; Kimenet:  rax = x * 2
func2:
    push rbp
    mov  rbp, rsp

    ; Számítás: rax = rdi * 2
    mov  rax, rdi
    add  rax, rax           ; rax = x + x = 2*x

    pop  rbp
    ret

; func1(x): meghívja func2(x + 1), visszaadja az eredményt
; Bemenet:  rdi = x
; Kimenet:  rax = func2(x + 1) = (x + 1) * 2
func1:
    push rbp
    mov  rbp, rsp
    sub  rsp, 16

    ; Elmentjük rdi-t, mert a call felülírhatja
    mov  [rbp - 8], rdi     ; x elmentve

    ; Argumentum előkészítése func2-höz: x + 1
    inc  rdi                ; rdi = x + 1
    call func2              ; rax = (x + 1) * 2

    ; rax már tartalmazza a visszatérési értéket, nem kell módosítani
    leave
    ret

; _start (="main"): meghívja func1(20), visszaad 42-t
_start:
    ; func1(20) hívása
    ; Elvárt: func1(20) = func2(21) = 21 * 2 = 42
    mov  rdi, 20
    call func1              ; rax = 42

    ; exit kód = func1(20) = 42
    mov  rdi, rax
    mov  rax, 60
    syscall
```

### Hívási lánc stack állapota

```
Stack állapot func2 futásakor (legmélyebb szint):

Magas cím
┌───────────────────────────────────────┐
│  [_start-ból jövő: nincs frame, RSP]  │
│  ret addr → _start (sys_exit előtt)   │ ← call func1 push-olta
├───────────────────────────────────────┤ ◄── func1 frame alapja
│  mentett rbp (_start-é)               │ ← push rbp
│  x=20 elmentve [rbp-8]               │ ← mov [rbp-8], rdi
│  padding [rbp-16]                    │
│  ret addr → func1 (inc rdi után)      │ ← call func2 push-olta
├───────────────────────────────────────┤ ◄── func2 frame alapja
│  mentett rbp (func1-é)                │ ← push rbp
│  (func2 nem foglal lokális változót)  │
└───────────────────────────────────────┘ ◄── rsp
Alacsony cím

Visszatérés:
  func2 ret → rax = 42, RSP visszaáll func1 frame-be
  func1 ret → rax = 42, RSP visszaáll _start-ba
  _start: mov rdi, rax → exit(42)
```

**Regiszter mentési összefoglaló ebben a példában:**

| Regiszter | Ki menti? | Miért? |
|-----------|-----------|--------|
| `rbp`     | mindkét függvény (callee) | callee-saved |
| `rdi`     | func1 (caller szerepben) | caller-saved; func1 hívja func2-t, és utána már nem kell rdi |
| `rax`     | senki | caller-saved; func2 visszatérési értéke |

---

## 9. Gyakorlatok

### 1. feladat – Összeadó függvény

Írj egy `add_two` függvényt, amely két egész számot kap argumentumként (`rdi` és `rsi`), és visszaadja az összegüket `rax`-ban. Hívd meg `_start`-ból `add_two(15, 27)`-tel, és az exit kód legyen az eredmény (42).

<details>
<summary>Megoldás megtekintése</summary>

```nasm
section .text
global _start

add_two:
    push rbp
    mov  rbp, rsp
    mov  rax, rdi       ; rax = első argumentum
    add  rax, rsi       ; rax += második argumentum
    pop  rbp
    ret

_start:
    mov  rdi, 15
    mov  rsi, 27
    call add_two        ; rax = 42
    mov  rdi, rax
    mov  rax, 60
    syscall
```
</details>

---

### 2. feladat – Maximum két szám közül

Írj egy `max_of_two` függvényt, amely `rdi`-ben és `rsi`-ben kapja a két számot, és `rax`-ban adja vissza a nagyobbikat. Teszteld `max_of_two(17, 99)` hívással (várt eredmény: 99).

<details>
<summary>Megoldás megtekintése</summary>

```nasm
section .text
global _start

max_of_two:
    push rbp
    mov  rbp, rsp
    mov  rax, rdi       ; rax = első (alapértelmezett max)
    cmp  rsi, rdi       ; rsi > rdi ?
    jle  .done          ; ha nem, rax marad rdi
    mov  rax, rsi       ; különben rax = rsi
.done:
    pop  rbp
    ret

_start:
    mov  rdi, 17
    mov  rsi, 99
    call max_of_two     ; rax = 99
    mov  rdi, rax
    mov  rax, 60
    syscall
```
</details>

---

### 3. feladat – Stack-en tárolt értékek

Írj programot, amely a `push` utasítással három értéket (10, 20, 30) tesz a stack-re, majd `pop`-okkal fordított sorrendben kiveszi őket, és az utolsóként kivett értékkel (10) tér ki.

<details>
<summary>Megoldás megtekintése</summary>

```nasm
section .text
global _start

_start:
    push 10
    push 20
    push 30

    pop  rax        ; rax = 30
    pop  rax        ; rax = 20
    pop  rax        ; rax = 10

    mov  rdi, rax   ; exit kód = 10
    mov  rax, 60
    syscall
```
</details>

---

### 4. feladat – Hatványozás iteratívan

Írj egy `power` függvényt, amely `rdi`-ben kapja az alapszámot, `rsi`-ben a kitevőt (≥ 0), és `rax`-ban adja vissza az eredményt. Implementálj szorzásos ciklust. Teszteld `power(3, 4)` = 81-gyel.

<details>
<summary>Megoldás megtekintése</summary>

```nasm
section .text
global _start

; power(base, exp): base^exp
; Bemenet: rdi = base, rsi = exp
; Kimenet: rax = base^exp
power:
    push rbp
    mov  rbp, rsp
    push rbx                ; rbx callee-saved

    mov  rax, 1             ; eredmény = 1
    mov  rcx, rsi           ; számláló = exp
    mov  rbx, rdi           ; base eltárolása

.loop:
    test rcx, rcx           ; rcx == 0 ?
    jz   .done
    imul rax, rbx           ; rax *= base
    dec  rcx
    jmp  .loop

.done:
    pop  rbx
    pop  rbp
    ret

_start:
    mov  rdi, 3             ; base = 3
    mov  rsi, 4             ; exp = 4
    call power              ; rax = 3^4 = 81
    mov  rdi, rax
    mov  rax, 60
    syscall
```
</details>

---

### 5. feladat – Callee-saved regiszter hiba javítása

Az alábbi kódban van egy hiba a callee-saved regiszterekkel kapcsolatban. Találd meg és javítsd!

```nasm
section .text
global _start

modify_rbx:
    push rbp
    mov  rbp, rsp
    mov  rbx, 999       ; rbx felülírva, de nem mentettük el!
    pop  rbp
    ret

_start:
    mov  rbx, 42        ; rbx = 42
    call modify_rbx
    ; itt rbx-ben 42-t várunk, de 999 lesz!
    mov  rdi, rbx       ; ha hibás: rdi = 999, ha helyes: rdi = 42
    mov  rax, 60
    syscall
```

<details>
<summary>Megoldás megtekintése</summary>

```nasm
modify_rbx:
    push rbp
    mov  rbp, rsp
    push rbx            ; ← EZT KELLETT HOZZÁADNI (callee-saved mentés)
    mov  rbx, 999       ; rbx felülírható, mert majd visszaállítjuk
    pop  rbx            ; ← ÉS EZT (visszaállítás ret előtt)
    pop  rbp
    ret
```
A hiba: `rbx` callee-saved regiszter, tehát ha `modify_rbx` módosítja, köteles elmenteni (push) és visszaállítani (pop) a `ret` előtt.
</details>

---

## 10. Ellenőrizd magad

**1. Melyik irányba nő a stack x86-64 Linux rendszeren, és mit jelent ez az RSP értékére nézve?**

<details>
<summary>Válasz</summary>
A stack **lefelé nő** – a magasabb memóriacímektől az alacsonyabbak felé. Ez azt jelenti, hogy `push` esetén az RSP értéke **csökken** 8-cal, `pop` esetén pedig **nő** 8-cal.
</details>

---

**2. Mit csinál pontosan a `call my_func` utasítás? Írd le két elemi lépésben!**

<details>
<summary>Válasz</summary>

1. `push rip` – a visszatérési cím (a `call` utáni következő utasítás címe) a stack-re kerül
2. `jmp my_func` – a vezérlés átadódik `my_func`-nak

</details>

---

**3. Melyik hat regiszterben adódnak át az első hat egész/pointer argumentumok a System V AMD64 ABI szerint? (Sorrendben!)**

<details>
<summary>Válasz</summary>
`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`
</details>

---

**4. Mi a különbség caller-saved és callee-saved regiszterek között? Adj egy-egy példát mindkét kategóriára!**

<details>
<summary>Válasz</summary>

- **Caller-saved (scratch):** A hívott függvény felülírhatja ezeket; ha a hívónak kell az értékük, a hívónak kell elmenteni call előtt. Példák: `rax`, `rcx`, `rdx`, `rdi`, `rsi`, `r8`-`r11`
- **Callee-saved (preserved):** Ha a hívott függvény ezeket használja, köteles push-sal elmenteni és pop-pal visszaállítani a ret előtt. Példák: `rbx`, `rbp`, `r12`-`r15`

</details>

---

**5. Mi a function prologue célja? Írd le a három utasítást és magyarázd el mindegyiket!**

<details>
<summary>Válasz</summary>

```nasm
push rbp        ; Az előző frame pointer elmentése a stack-re
mov  rbp, rsp   ; rbp = jelenlegi rsp (az új frame alapja)
sub  rsp, N     ; Helyfoglalás a lokális változóknak (N bájt)
```

- `push rbp`: megőrzi a hívó frame pointer-ét, hogy a ret után helyreállítható legyen
- `mov rbp, rsp`: az `rbp` stabil referencia pontként szolgál a lokális változókhoz (`[rbp-8]`, `[rbp-16]` stb.), miközben `rsp` változhat
- `sub rsp, N`: lefoglalja a helyet a lokális változóknak

</details>

---

**6. Miért kell a faktoriális rekurzív megvalósításában a `rdi` értékét a stack-re (lokális változóba) menteni a rekurzív `call` előtt?**

<details>
<summary>Válasz</summary>

Az `rdi` **caller-saved** (scratch) regiszter. A rekurzív `call factorial` végrehajtásakor a hívott `factorial` példány szabadon felülírhatja `rdi`-t (valójában szükségszerűen meg is teszi, hogy átadja a következő argumentumot). Ha a hívó `factorial` példány nem menti el előtte a saját `n` értékét (`rdi`-t), az `imul rax, [rbp-8]` szorzás elveszett adattal dolgozna. Az elmentés a lokális változóba (`[rbp-8]`) biztosítja, hogy a `call` után is hozzáférünk az eredeti `n`-hez.
</details>

---

## Összefoglalás

| Fogalom | Leírás |
|---------|--------|
| **Stack** | LIFO adatszerkezet, lefelé nő a memóriában |
| **RSP** | Stack pointer, mindig a stack tetejére mutat |
| **push src** | `rsp -= 8; [rsp] = src` |
| **pop dst** | `dst = [rsp]; rsp += 8` |
| **call label** | `push rip; jmp label` |
| **ret** | `pop rip` |
| **Prologue** | `push rbp; mov rbp, rsp; sub rsp, N` |
| **Epilogue** | `leave; ret` vagy `mov rsp, rbp; pop rbp; ret` |
| **Caller-saved** | rax, rcx, rdx, rsi, rdi, r8-r11 – hívó menti ha kell |
| **Callee-saved** | rbx, rbp, r12-r15 – hívott menti ha módosítja |
| **Argumentumok** | rdi, rsi, rdx, rcx, r8, r9 (első 6) |
| **Visszatérési érték** | rax |

> **💡 Tipp a következő napra:** Holnap a memóriakezelést és a címzési módokat vesszük, ahol a stack frame ismerete kulcsfontosságú lesz: lokális változókat, tömböket és pointereket kezelünk majd Assembly-ben.

---

*5. nap vége – Stack és függvények | x86-64 Assembly tananyag*
