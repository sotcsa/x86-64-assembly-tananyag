# 10. NAP – Záró projekt és továbblépés

## Mit tanulunk ma?

Az utolsó napon összerakjuk a 10 nap tudását: megnézzük, hogyan kommunikál az Assembly a C-vel (és így közvetve bármely C-t hívó rendszerrel), megírjuk a záró projektet, majd irányt mutatunk a továbblépéshez.

**Java párhuzam:** A C interoperabilitás olyan, mint a JNI (Java Native Interface) – Java hív natív kódot és fordítva. Az Assembly-C határon ugyanazok a szabályok érvényesek: System V AMD64 ABI, stack igazítás, callee-saved regiszterek.

---

## 1. C interoperabilitás

### Assembly függvény hívása C-ből

Az Assembly függvény pontosan ugyanúgy néz ki, mint egy C függvény – a hívási konvenciót (System V AMD64 ABI) kell betartani:

**Szabályok emlékeztetőül:**
- Argumentumok: `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`
- Visszatérési érték: `rax`
- Callee-saved: `rbx`, `rbp`, `r12`–`r15`, `rsp`
- Caller-saved: `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11`

```nasm
; add_asm.asm – C-ből hívható Assembly függvény
; int64_t add_asm(int64_t a, int64_t b);

global add_asm          ; exportáljuk a szimbólumot

section .text

; Bemenet:  RDI = a, RSI = b
; Kimenet:  RAX = a + b
add_asm:
    mov     rax, rdi    ; RAX = a
    add     rax, rsi    ; RAX = a + b
    ret
```

```c
/* main.c */
#include <stdio.h>
#include <stdint.h>

extern int64_t add_asm(int64_t a, int64_t b);

int main(void) {
    int64_t eredmeny = add_asm(10, 32);
    printf("10 + 32 = %ld\n", eredmeny);   /* 42 */
    return 0;
}
```

```bash
# Fordítás és linkelés:
nasm -f elf64 add_asm.asm -o add_asm.o
gcc -c main.c -o main.o
gcc main.o add_asm.o -o program
./program
# Kimenet: 10 + 32 = 42
```

> **💡 Tipp:** A `global` direktíva nélkül a linker nem találja meg a szimbólumot (`undefined reference` hiba).

---

### C függvény hívása Assembly-ből: `printf`

A leggyakoribb eset: `printf` hívása sztring + szám kiírásához.

```nasm
; printf_demo.asm – printf hívása Assembly-ből
; Fordítás:
;   nasm -f elf64 printf_demo.asm -o printf_demo.o
;   gcc -no-pie printf_demo.o -o printf_demo
;   ./printf_demo

extern printf           ; külső szimbólum: C printf
extern exit             ; külső szimbólum: C exit

section .data
    fmt_str     db  "Szam: %ld, String: %s", 10, 0   ; format string
    szoveg      db  "Hello!", 0

section .text
    global main         ; C runtime main függvény

main:
    ; Stack igazítás: belépéskor RSP = 8-ra igazított (ret. cím pushed)
    ; 16-ra igazításhoz: push rbp (8 byte), RSP lesz 16-ra igazított
    push    rbp
    mov     rbp, rsp

    ; printf("Szam: %ld, String: %s\n", 42LL, "Hello!")
    lea     rdi, [rel fmt_str]      ; 1. arg: format string
    mov     rsi, 42                  ; 2. arg: szám (long)
    lea     rdx, [rel szoveg]       ; 3. arg: string pointer
    xor     eax, eax                 ; AL = 0 (nincs float arg)
    call    printf

    ; exit(0)
    xor     edi, edi
    call    exit
```

```bash
nasm -f elf64 printf_demo.asm -o printf_demo.o
gcc -no-pie printf_demo.o -o printf_demo
./printf_demo
# Kimenet: Szam: 42, String: Hello!
```

> **⚠️ Figyelem:** A `gcc -no-pie` linker flag szükséges, mert alapértelmezetten a GCC PIE (Position Independent Executable) binárist készít, ami más linkelési modellt vár. Assembly kódunknál ez konfliktust okozhat, ezért a `no-pie` megkönnyíti a linkelést.

---

### `malloc` és `free` hívása Assembly-ből

```nasm
; malloc_demo.asm – dinamikus memóriafoglalás C malloc-kal
extern malloc
extern free
extern printf

section .data
    fmt     db  "Foglalt cim: %p, ertek: %d", 10, 0

section .text
    global main

main:
    push    rbp
    mov     rbp, rsp
    push    rbx             ; callee-saved, megőrizzük

    ; ptr = malloc(4)  – 4 byte int-nek
    mov     rdi, 4
    call    malloc
    ; RAX = foglalt memória pointer (vagy NULL hiba esetén)
    mov     rbx, rax        ; mentjük callee-saved regiszterbe

    ; *ptr = 99
    mov     dword [rbx], 99

    ; printf("Foglalt cim: %p, ertek: %d\n", ptr, *ptr)
    lea     rdi, [rel fmt]
    mov     rsi, rbx            ; pointer
    mov     edx, dword [rbx]    ; érték
    xor     eax, eax
    call    printf

    ; free(ptr)
    mov     rdi, rbx
    call    free

    pop     rbx
    xor     eax, eax        ; return 0
    pop     rbp
    ret
```

```bash
nasm -f elf64 malloc_demo.asm -o malloc_demo.o
gcc -no-pie malloc_demo.o -o malloc_demo
./malloc_demo
```

---

## 2. Záró projekt: Szám → Szöveg konverter

### A feladat

Írj egy programot, amely egy egész számot (pl. 12345) ASCII szöveggé alakít és kiír stdout-ra. Csak a `write` syscall-t használd – semmilyen C könyvtárat!

### Az algoritmus

```
12345 → számjegyek kinyerése:
  12345 mod 10 = 5  → '5' (0x35)
  1234  mod 10 = 4  → '4'
  123   mod 10 = 3  → '3'
  12    mod 10 = 2  → '2'
  1     mod 10 = 1  → '1'
  0     → STOP

Sorrendben hátulról előre: "12345"
→ Stack-re tesszük fordítva, majd elővesszük sorban
```

### Teljes, kommentezett megvalósítás

```nasm
; ============================================================
; szam_kiir.asm – Egész szám kiírása stdout-ra
;
; Algoritmus:
;   1. Osztjuk 10-zel, a maradék adja az aktuális jegyet
;   2. A jegyet ASCII-vé alakítjuk (+ '0' = + 48)
;   3. A jegyeket stack-re toljuk (fordított sorrend)
;   4. Előszzedjük a stack-ről és pufferbe írjuk
;   5. write syscall
;
; Fordítás:
;   nasm -f elf64 -g -F dwarf szam_kiir.asm -o szam_kiir.o
;   ld szam_kiir.o -o szam_kiir
;   ./szam_kiir
; ============================================================

section .data
    newline     db  10              ; '\n' karakter

section .bss
    puffer      resb 24             ; max 20 számjegy (int64) + előjel + null

section .text
    global _start

; -------------------------------------------------------
; print_uint64(rdi = szam)
; Kiírja az előjel nélküli 64-bites egészet stdout-ra.
; -------------------------------------------------------
print_uint64:
    push    rbp
    mov     rbp, rsp
    push    rbx                     ; callee-saved
    push    r12                     ; callee-saved
    push    r13                     ; callee-saved

    mov     rbx, rdi                ; RBX = szam (megőrizzük)
    lea     r12, [rel puffer]       ; R12 = puffer kezdete
    xor     r13, r13                ; R13 = jegyszámláló

    ; Speciális eset: szám == 0
    test    rbx, rbx
    jnz     .convert_loop
    mov     byte [r12], '0'
    mov     r13, 1
    jmp     .write_out

.convert_loop:
    ; Osztás 10-zel: rdx:rax / rbx = hányados:maradék
    test    rbx, rbx
    jz      .reverse_done
    xor     rdx, rdx                ; RDX = 0 (felső rész)
    mov     rax, rbx                ; RAX = aktuális szám
    mov     rcx, 10
    div     rcx                     ; RAX = hányados, RDX = maradék
    mov     rbx, rax                ; rbx = hányados (következő iteráció)

    ; Maradék → ASCII karakter, stack-re toljuk
    add     dl, '0'                 ; '0' + (0..9) = '0'..'9'
    push    rdx                     ; stack-re (8 byte-ként, de csak 1 byte kell)
    inc     r13                     ; jegyszámláló++
    jmp     .convert_loop

.reverse_done:
    ; Jegyek stack-ről pufferbe másolása (most helyes sorrendben)
    mov     rcx, r13                ; rcx = jegyek száma
    xor     rbx, rbx                ; rbx = puffer index

.pop_loop:
    test    rcx, rcx
    jz      .write_out
    pop     rax                     ; legfelső jegy
    mov     [r12 + rbx], al         ; pufferbe
    inc     rbx
    dec     rcx
    jmp     .pop_loop

.write_out:
    ; sys_write(1, puffer, r13)
    mov     rax, 1                  ; sys_write
    mov     rdi, 1                  ; stdout
    mov     rsi, r12                ; puffer cím
    mov     rdx, r13                ; jegyek száma
    syscall

    pop     r13
    pop     r12
    pop     rbx
    pop     rbp
    ret

; -------------------------------------------------------
; print_int64(rdi = előjeles szám)
; Kezeli a negatív számokat is.
; -------------------------------------------------------
print_int64:
    push    rbp
    mov     rbp, rsp
    push    rbx

    mov     rbx, rdi                ; mentjük a számot
    test    rdi, rdi
    jns     .nonneg                 ; >= 0? ugrás

    ; Negatív: írjuk ki a '-' jelet
    push    rbx                     ; RBX mentése a syscall előtt
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel minus_jel]
    mov     rdx, 1
    syscall
    pop     rbx

    neg     rbx                     ; RBX = -RBX (abszolút érték)

.nonneg:
    mov     rdi, rbx
    call    print_uint64

    pop     rbx
    pop     rbp
    ret

_start:
    ; Tesztesetek: különböző számok kiírása
    mov     rdi, 12345
    call    print_uint64
    ; Newline
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel newline]
    mov     rdx, 1
    syscall

    mov     rdi, 0
    call    print_uint64
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel newline]
    mov     rdx, 1
    syscall

    mov     rdi, 9999999999         ; nagy szám
    call    print_uint64
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel newline]
    mov     rdx, 1
    syscall

    ; Kilépés
    xor     rdi, rdi
    mov     rax, 60
    syscall

section .data
    minus_jel   db  '-'
```

```bash
nasm -f elf64 -g -F dwarf szam_kiir.asm -o szam_kiir.o
ld szam_kiir.o -o szam_kiir
./szam_kiir
# Kimenet:
# 12345
# 0
# 9999999999
```

---

## 3. Záró projekt 2: Egyszerű szöveg kereső (opcionális)

```nasm
; ============================================================
; char_kereses.asm – Karakter keresése stringben (scasb)
; Visszaadja az első előfordulás indexét, vagy -1-et ha nincs.
;
; Fordítás:
;   nasm -f elf64 -g -F dwarf char_kereses.asm -o char_kereses.o
;   ld char_kereses.o -o char_kereses
;   ./char_kereses
;   echo $?    ; 'l' első indexe "Hello"-ban = 2
; ============================================================

section .data
    uzenet  db  "Hello, Assembly!", 0
    UZENET_MERET    equ 17

section .text
    global _start

; -------------------------------------------------------
; strchr_idx(rdi=string, rsi=karakter) -> rax = index / -1
; -------------------------------------------------------
strchr_idx:
    push    rdi                 ; RDI eredeti értéke
    mov     al, sil             ; AL = keresett karakter (rsi alsó byte-ja)
    cld                         ; DF=0, előre haladás

    ; Meghatározzuk a string hosszát először (hogy rcx-et beállítsuk)
    ; Egyszerűbb megközelítés: byte-onként haladunk
    xor     rcx, rcx            ; rcx = index

.loop:
    cmp     byte [rdi + rcx], 0 ; null terminátor?
    je      .not_found
    cmp     byte [rdi + rcx], al ; megtaláltuk?
    je      .found
    inc     rcx
    jmp     .loop

.found:
    mov     rax, rcx            ; RAX = index
    pop     rdi
    ret

.not_found:
    mov     rax, -1             ; RAX = -1 (nem találtuk)
    pop     rdi
    ret

_start:
    lea     rdi, [rel uzenet]
    mov     rsi, 'l'            ; 'l' betű keresése
    call    strchr_idx          ; RAX = 2 ("He**l**lo...")

    ; Kilépés az indexszel
    mov     rdi, rax
    mov     rax, 60
    syscall
```

---

## 4. Amit NEM tanultunk (és merre tovább)

### Lebegőpontos műveletek (SSE/AVX)

Az x86-64 lebegőpontos számításokhoz az **XMM/YMM/ZMM** regisztereket és az **SSE/SSE2/AVX/AVX-512** utasításkészletet használja.

```nasm
; Lebegőpontos összeadás SSE2-vel:
section .data
    a   dd  3.14        ; float (32-bit)
    b   dd  2.71

section .text
    movss   xmm0, [a]   ; XMM0 = 3.14
    addss   xmm0, [b]   ; XMM0 = 5.85
```

**Mire jó?** Tudományos számítások, képfeldolgozás, gépi tanulás (matrixszorzás), jelfeldolgozás.

---

### SIMD utasítások

A **SIMD** (Single Instruction, Multiple Data) egyszerre több adaton végez műveletet. Például az `addps` egyszerre 4 float-ot ad össze egyetlen utasítással:

```nasm
addps   xmm0, xmm1  ; 4x float összeadás egyszerre (SSE)
addpd   xmm0, xmm1  ; 2x double összeadás egyszerre (SSE2)
vaddps  ymm0, ymm1, ymm2   ; 8x float összeadás (AVX)
```

**Mire jó?** Videókodekek, ML inferencia (ONNX Runtime), adatbázisok (vectorized query processing).

---

### Inline Assembly (GCC `__asm__`)

C/C++ kódba ágyazva közvetlenül írhatunk Assembly utasításokat:

```c
int add_asm(int a, int b) {
    int result;
    __asm__ volatile (
        "addl %2, %1\n\t"
        "movl %1, %0"
        : "=r"(result)          /* kimenet */
        : "r"(a), "r"(b)        /* bemenetek */
        :                       /* clobbered registers */
    );
    return result;
}
```

**Mire jó?** Kritikus teljesítményszakaszok, speciális utasítások (pl. `rdtsc` időmérés, `cpuid` CPU infó), kernel kód.

---

### Operációs rendszer fejlesztés

A bootloader (0x7C00 szegmens), a kernel belépési pont, az IRQ kezelők – mind Assembly-ben íródnak. A Linux kernel `arch/x86/` könyvtára tele van Assembly fájlokkal (`.S` kiterjesztés).

**Ajánlott kezdőpont:** [OSDev Wiki](https://wiki.osdev.org/), bare-metal x86 programozás QEMU emulátorral.

---

### Reverse Engineering és malware elemzés

Az Assembly tudás elengedhetetlen a reverse engineering munkához:
- **Ghidra** (NSA open-source tool) – bináris disassembly + decompiler
- **IDA Pro / IDA Free** – ipari standard
- **Radare2** – command-line RE framework
- **Binary Ninja** – modern RE platform

---

### Hasznos könyvek, kurzusok, oldalak

| Forrás | Leírás | Link/Megjegyzés |
|--------|--------|-----------------|
| **Computer Systems: A Programmer's Perspective (CS:APP)** | Bryant & O'Hallaron – Az x86 Assembly legjobb átfogó könyve, tankönyvi szinten | 3. kiadás ajánlott |
| **Programming from the Ground Up** | Jonathan Bartlett – Ingyenes könyv, Linux x86 Assembly kezdőknek | PDF ingyenesen letölthető |
| **The Art of Assembly Language** | Randall Hyde – HLA (High Level Assembler), részletes, hosszú | randallhyde.com |
| **Intel 64 and IA-32 Architectures Software Developer Manuals** | Hivatalos Intel referencia – 4000+ oldal | intel.com (ingyenes PDF) |
| **Compiler Explorer** | C/C++/Rust kód → Assembly, böngészőből | godbolt.org |
| **OSDev Wiki** | OS fejlesztés Assembly-vel | wiki.osdev.org |
| **pwn.college** | Ingyenes, interaktív CTF/security kurzus, Assembly feladatokkal | pwn.college |
| **x86 Intrinsics Guide** | Intel SIMD/SSE/AVX utasítások referenciája | intel.com/content/www/us/en/docs/intrinsics-guide |
| **Agner Fog optimalizálási útmutatók** | CPU microarchitecture, pipeline, optimalizálás | agner.org/optimize |
| **nasm.us dokumentáció** | NASM hivatalos dokumentáció | nasm.us/doc |

---

## 5. Összefoglalás – A 10 nap legfontosabb tanulságai

### 1. NAP: Alapok
A CPU regiszterek (RAX, RBX, ..., R15), a számrendszerek (hex, bináris), és az a felismerés, hogy a gépi kód nem "mágia" – csak bytes a memóriában.

### 2. NAP: Első program
NASM szintaxis, `.text`/`.data`/`.bss` szekciók, az első `Hello, World!`, fordítás (`nasm`) és linkelés (`ld`).

### 3. NAP: Adatmozgatás és aritmetika
`mov`, `add`, `sub`, `imul`, `idiv`, `inc`, `dec`. Flag-ek (ZF, SF, CF, OF) – a processzor "hangulatjelzői".

### 4. NAP: Vezérlési szerkezetek
`jmp`, `cmp`, feltételes ugrások (`je`, `jne`, `jl`, `jg`, ...), ciklusok (`loop`, `dec + jnz`).

### 5. NAP: Stack és függvények
`push`/`pop`, `call`/`ret`, a stack frame struktúrája, a System V AMD64 ABI hívási konvenció.

### 6. NAP: Memóriakezelés
Címzési módok (`[base + index * scale + offset]`), `lea`, pointerek Assembly-ben, kétdimenziós tömbök.

### 7. NAP: Syscall-ok
Linux syscall interface (RAX = számkód, RDI/RSI/RDX = argumentumok), `write`, `read`, `open`, `close`, `exit`.

### 8. NAP: Stringek és adatstruktúrák
String utasítások (`movsb`, `stosb`, `lodsb`, `scasb`, `cmpsb`), `rep` prefix, null-terminált stringek, `strlen`/`strcpy` implementáció, struktúrák és tömbök.

### 9. NAP: Debugging és optimalizálás
GDB (`stepi`, `nexti`, `info registers`, `x/...`), `strace`, `objdump`, gyakori hibák (SIGSEGV, stack alignment), teljesítmény tippek (`xor reg, reg`, `lea`, cache-barát kód).

### 10. NAP: Záró projekt
C interoperabilitás (Assembly ↔ C), szám→szöveg konverter stack alapú számjegy-megfordítással, továbblépési irányok.

---

## 6. Cheat Sheet – Gyors referencia

### Regiszterek

```
64-bit   32-bit   16-bit   8-bit (low)
rax      eax      ax       al      ← visszatérési érték, akkumulátor
rbx      ebx      bx       bl      ← callee-saved
rcx      ecx      cx       cl      ← számláló (loop, rep)
rdx      edx      dx       dl      ← adat, div maradék
rsi      esi      si       sil     ← forrás, 2. arg
rdi      edi      di       dil     ← cél, 1. arg
rsp      esp      sp       spl     ← stack pointer
rbp      ebp      bp       bpl     ← frame pointer, callee-saved
r8-r15   r8d-r15d r8w-r15w r8b-r15b ← 5-6. arg, callee(r12-r15)
```

### Adatmozgatás

| Utasítás | Példa | Leírás |
|----------|-------|--------|
| `mov`    | `mov rax, rbx` | Másolás |
| `movzx`  | `movzx rax, bl` | Nullával kiterjesztett másolás |
| `movsx`  | `movsx rax, ebx` | Előjellel kiterjesztett másolás |
| `lea`    | `lea rax, [rbx + rcx*4]` | Cím számítás (nem dereferál) |
| `push`   | `push rax` | Stack-re tolás (RSP -= 8) |
| `pop`    | `pop rax` | Stack-ről szedés (RSP += 8) |
| `xchg`   | `xchg rax, rbx` | Csere |

### Aritmetika

| Utasítás | Példa | Leírás |
|----------|-------|--------|
| `add`    | `add rax, rbx` | Összeadás |
| `sub`    | `sub rax, 5` | Kivonás |
| `imul`   | `imul rax, rbx, 3` | Előjeles szorzás |
| `idiv`   | `idiv rcx` | Előjeles osztás (rdx:rax / rcx) |
| `div`    | `div rcx` | Előjel nélküli osztás |
| `inc`    | `inc rax` | Növelés |
| `dec`    | `dec rax` | Csökkentés |
| `neg`    | `neg rax` | Negálás (kettes komplemens) |
| `cqo`    | `cqo` | RAX előjelkiterjesztése RDX:RAX-ba |

### Logikai műveletek

| Utasítás | Példa | Leírás |
|----------|-------|--------|
| `and`    | `and rax, 0xFF` | Bitenkénti ÉS |
| `or`     | `or rax, rbx` | Bitenkénti VAGY |
| `xor`    | `xor rax, rax` | Bitenkénti XOR (nullázásra is!) |
| `not`    | `not rax` | Bitenkénti NOT |
| `shl`    | `shl rax, 3` | Bal léptetés (× 8) |
| `shr`    | `shr rax, 1` | Jobbra léptetés előjel nélkül (÷ 2) |
| `sar`    | `sar rax, 1` | Aritmetikai jobbra léptetés (előjeles ÷ 2) |
| `test`   | `test rax, rax` | AND flag-ekre (eredmény eldobva) |
| `cmp`    | `cmp rax, 5` | SUB flag-ekre (eredmény eldobva) |

### Ugró utasítások

| Utasítás | Feltétel | Típus |
|----------|----------|-------|
| `jmp`    | mindig   | feltétel nélküli |
| `je` / `jz` | ZF=1 | egyenlő / nulla |
| `jne` / `jnz` | ZF=0 | nem egyenlő |
| `jl` / `jnge` | SF≠OF | kisebb (előjeles) |
| `jg` / `jnle` | ZF=0, SF=OF | nagyobb (előjeles) |
| `jle` / `jng` | ZF=1 \| SF≠OF | kisebb-egyenlő (előjeles) |
| `jge` / `jnl` | SF=OF | nagyobb-egyenlő (előjeles) |
| `jb` / `jnae` | CF=1 | kisebb (előjel nélküli) |
| `ja` / `jnbe` | CF=0, ZF=0 | nagyobb (előjel nélküli) |
| `js` | SF=1 | negatív |
| `jns` | SF=0 | nem negatív |
| `loop` | RCX--; RCX≠0 | ciklus |

### Stack és függvényhívás

```nasm
call label      ; RIP stackre, ugrás label-re
ret             ; POP RIP
ret N           ; POP RIP, RSP += N (stdcall)
enter N, 0      ; push rbp; mov rbp, rsp; sub rsp, N
leave           ; mov rsp, rbp; pop rbp
```

### String utasítások

| Utasítás | Hatás | DF=0 lépés |
|----------|-------|-----------|
| `movsb/w/d/q` | [RDI]←[RSI] | RSI++, RDI++ |
| `stosb/w/d/q` | [RDI]←AL/AX/EAX/RAX | RDI++ |
| `lodsb/w/d/q` | AL/AX/EAX/RAX←[RSI] | RSI++ |
| `scasb/w`     | AL/AX - [RDI] (flag) | RDI++ |
| `cmpsb/w`     | [RSI]-[RDI] (flag) | RSI++, RDI++ |
| `rep X`       | X × RCX-szor | RCX-- |
| `repne scasb` | amíg [RDI]≠AL | — |
| `repe cmpsb`  | amíg [RSI]=[RDI] | — |

### Syscall konvenció (Linux x86-64)

```
rax = syscall szám
rdi = 1. argumentum
rsi = 2. argumentum
rdx = 3. argumentum
r10 = 4. argumentum
r8  = 5. argumentum
r9  = 6. argumentum
syscall → rax = visszatérési érték (negatív = errno)
```

Legfontosabb syscall számok:

| Szám | Név | Leírás |
|------|-----|--------|
| 0 | `read` | Olvasás fd-ről |
| 1 | `write` | Írás fd-re |
| 2 | `open` | Fájl megnyitása |
| 3 | `close` | Fájl lezárása |
| 9 | `mmap` | Memória leképezés |
| 12 | `brk` | Heap módosítás |
| 60 | `exit` | Folyamat kilépés |

### NASM direktívák

| Direktíva | Példa | Leírás |
|-----------|-------|--------|
| `db` | `db 0x41, "Hello", 0` | Byte(s) definiálása |
| `dw` | `dw 1000` | Word (2 byte) |
| `dd` | `dd 3.14` | Dword (4 byte) |
| `dq` | `dq 0xDEADBEEF` | Qword (8 byte) |
| `resb` | `resb 64` | N byte foglalás (.bss) |
| `resw/d/q` | `resq 10` | N word/dword/qword |
| `equ` | `MAX equ 100` | Konstans definíció |
| `global` | `global _start` | Szimbólum exportálás |
| `extern` | `extern printf` | Külső szimbólum |
| `section` | `section .data` | Szekció váltás |
| `%define` | `%define SIZE 10` | Makró definiálás |

---

## Gyakorlatok

### 1. C + Assembly integráció
Írj egy C programot, amely egy Assembly függvényt hív meg: a függvény kiszámolja a Fibonacci-sorozat N-edik elemét rekurzió nélkül, csak regiszterekben. Linkeld össze `gcc`-vel.

### 2. Szám konverter bővítése
Bővítsd a `szam_kiir.asm` programot: legyen képes hexadecimális formában is kiírni a számot (pl. `0x3039` = 12345). Tipp: 16-tal való osztás, `0..9` → `'0'..'9'`, `10..15` → `'A'..'F'`.

### 3. Karakter statisztika
Írj programot, amely beolvas egy szöveget stdin-ről (`read` syscall), majd kiírja: hány betűt, számjegyet és szóközt talált. Használj tömböt a számlálóknak.

### 4. Egyszerű stack kalkulátor
Implementálj egy RPN (Reverse Polish Notation) kalkulátort: `3 4 + 2 *` → 14. Olvass karaktereket, értelmezd a számokat és operátorokat, használd a stack-et.

### 5. String palindróma ellenőrzés
Írj egy `is_palindrome` függvényt, amely megmondja, hogy egy null-terminált string palindróma-e (pl. "racecar" → igen). Két pointert használj (eleje és vége), és haladj befelé.

---

## Ellenőrizd magad

1. Mi az `extern` direktíva szerepe, és mikor van rá szükség?
2. Miért kell `and rsp, -16` (vagy `push rbp`) a `printf` hívása előtt?
3. Mit jelent a `-no-pie` linker flag, és mikor szükséges?
4. Az `xor eax, eax` nullázza a **teljes** RAX-ot is? Miért?
5. Mi a különbség az `SSE` és `AVX` utasításkészlet között egy mondatban?
6. Mi a `godbolt.org` és mire tudod használni DevOps munkádban?

---

*Gratulálunk! Végigjártad az x86-64 Assembly 10 napos tanfolyamát.*

*Mellékletek: [Utasítás referencia](../mellekletek/utasitas-referencia.md) · [Syscall referencia](../mellekletek/syscall-referencia.md)*
