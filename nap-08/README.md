# 8. NAP – Stringkezelés és adatstruktúrák

## Mit tanulunk ma?

Az x86-64 processzor rendelkezik egy egész utasításkészlettel, amelyet kifejezetten sztring (byte-sorozat) műveletek hatékony elvégzésére terveztek. Ma megismerjük ezeket az utasításokat, a `rep` prefixet, a null-terminált C-stílusú stringek kezelését, majd megtanuljuk, hogyan szimulálhatunk C-struktúrákat és hogyan dolgozhatunk tömbökkel Assembly szinten.

**Java párhuzam:** Ahogyan a Java `String` a JVM-ben egy `char[]` tömb + hosszmező a heap-en, az Assembly stringek egyszerűen egymást követő byte-ok a memóriában – nulla byte-tal lezárva (C-stílus) vagy explicit hosszal nyilvántartva.

---

## 1. String utasítások

Az x86-64 string utasítások mindig két speciális regisztert használnak:
- **RSI** (*source index*) – forrás cím
- **RDI** (*destination index*) – cél cím
- **RCX** – számláló (rep prefixnél)
- **AL/AX/EAX/RAX** – adat (lods/stos utasításoknál)

Minden egyes végrehajtás után az RSI és/vagy RDI automatikusan léptet – előre vagy hátra, a **direction flag** (DF) állásától függően.

### Direction Flag (DF)

```nasm
cld     ; Clear Direction Flag – előre haladás (RSI/RDI növekszik)
std     ; Set Direction Flag   – hátra haladás (RSI/RDI csökken)
```

> **💡 Tipp:** A Linux ABI szerint a függvények belépéskor `cld` állapotot (DF=0) feltételeznek. Mindig állítsd vissza `cld`-vel, ha `std`-t használtál!

---

### `movsb/movsw/movsd/movsq` – String másolás (RSI → RDI)

Átmásol 1/2/4/8 byte-ot `[RSI]`-ból `[RDI]`-be, majd lépteti mindkét regisztert.

| Utasítás | Méret | RSI/RDI változás (DF=0) |
|----------|-------|--------------------------|
| `movsb`  | 1 B   | +1                       |
| `movsw`  | 2 B   | +2                       |
| `movsd`  | 4 B   | +4                       |
| `movsq`  | 8 B   | +8                       |

```nasm
; Egyetlen byte másolása:
; [RDI] ← [RSI],  RSI++, RDI++
movsb
```

---

### `stosb/stosw/stosd/stosq` – Memória feltöltés (AL/AX/EAX/RAX → RDI)

Beírja AL/AX/EAX/RAX értékét `[RDI]`-be, majd lépteti RDI-t.

```nasm
mov  al,  0
stosb       ; [RDI] ← 0, RDI++
```

Tipikus felhasználás: memóriaterület nullázása (`memset` ekvivalens).

---

### `lodsb/lodsw/lodsd/lodsq` – Betöltés stringből (RSI → AL/AX/EAX/RAX)

Betölti `[RSI]` tartalmát AL/AX/EAX/RAX-ba, majd lépteti RSI-t.

```nasm
lodsb       ; AL ← [RSI], RSI++
```

---

### `cmpsb/cmpsw` – String összehasonlítás

Összehasonlítja `[RSI]` és `[RDI]` tartalmát (kivonás flag-ekre), majd lépteti mindkét regisztert. Nem tárol eredményt, csak a flag-eket állítja.

```nasm
cmpsb       ; [RSI] - [RDI], flag-ek frissítése, RSI++, RDI++
```

---

### `scasb` – Keresés stringben

Összehasonlítja AL értékét `[RDI]`-vel, majd lépteti RDI-t.

```nasm
mov al, 'A'
scasb       ; AL - [RDI], flag-ek, RDI++
```

---

## 2. REP prefix

A `rep` prefix megismétli az utána álló string utasítást RCX-szer (RCX-et csökkentve minden lépésnél).

### `rep movsb` – RCX-szer másolás

```nasm
mov rcx, 16    ; 16 byte másolása
rep movsb      ; 16x: [RDI] ← [RSI], RSI++, RDI++, RCX--
```

### `repne scasb` – Keresés nulláig (vagy egyezésig)

- `repne` = *repeat while not equal* – addig ismétli, amíg nem talál egyezést, vagy RCX = 0
- Klasszikus `strlen` alap: addig lépteti RDI-t, amíg `AL == [RDI]` nem teljesül

```nasm
repne scasb    ; RDI++, RCX-- amíg [RDI] != AL és RCX > 0
```

### `repe cmpsb` – Összehasonlítás eltérésig

- `repe` = *repeat while equal* – addig ismétli, amíg egyeznek, vagy RCX = 0

```nasm
repe cmpsb     ; RSI++, RDI++, RCX-- amíg [RSI] == [RDI] és RCX > 0
```

> **⚠️ Figyelem:** A `rep` prefix nem minden string utasítással kombinálható. Használható: `rep movsb/w/d/q`, `rep stosb/w/d/q`, `rep lodsb/w/d/q`, `repe/repne cmpsb/w`, `repe/repne scasb`.

---

## 3. Null-terminated stringek (C-stílus)

A Linux rendszerhívások és a C könyvtár null-terminált stringeket használ: a string véget ér az első `0x00` (nulla) byte-nál. A NASM-ban a string konstansokban automatikusan **nem** kerül nulla lezáró – azt expliciten kell megadni.

```nasm
section .data
    szoveg  db  "Hello, World!", 0   ; null-terminált string
    szoveg2 db  "Hello", 10, 0       ; '\n' + null terminátor
```

---

### `strlen` implementáció

```nasm
; ============================================================
; strlen.asm – String hossz számítás (null-terminált string)
; Fordítás:
;   nasm -f elf64 -g -F dwarf strlen.asm -o strlen.o
;   ld strlen.o -o strlen
;   ./strlen && echo $?
; ============================================================

section .data
    uzenet  db  "Hello, Assembly!", 0
    fmt_len db  "Hossz: ", 0

section .bss
    buf     resb 4      ; számjegyeknek

section .text
    global _start

; -------------------------------------------------------
; strlen(rdi) -> rax
; Bemenet:  RDI = string kezdőcíme (null-terminált)
; Kimenet:  RAX = string hossza (null byte nélkül)
; -------------------------------------------------------
strlen:
    push    rdi             ; RDI mentése (callee-saved nem, de rögtön visszaállítjuk)
    xor     rax, rax        ; RAX = 0 (byte számláló)
.loop:
    cmp     byte [rdi], 0   ; elértük a null terminátort?
    je      .done
    inc     rdi             ; következő karakter
    inc     rax             ; számláló++
    jmp     .loop
.done:
    pop     rdi
    ret

; -------------------------------------------------------
; Alternatív strlen – scasb/repne használatával
; Bemenet:  RDI = string kezdőcíme
; Kimenet:  RAX = hossz
; -------------------------------------------------------
strlen_fast:
    xor     rcx, rcx
    not     rcx             ; RCX = 0xFFFFFFFFFFFFFFFF (max hossz)
    xor     al,  al         ; AL = 0 (keresett karakter: null byte)
    cld                     ; DF = 0 (előre haladás)
    repne   scasb           ; keres null byte-ot [RDI]-ben, RDI++, RCX--
    ; RCX most: 0xFFFFFFFF... - 1 - hossz
    not     rcx             ; bitenkénti negálás
    dec     rcx             ; rcx = hossz (null byte nélkül)
    mov     rax, rcx
    ret

_start:
    ; strlen hívása
    lea     rdi, [rel uzenet]
    call    strlen
    ; RAX tartalmazza a hosszt (16)

    ; Visszatérés a hosszsal (exit code)
    mov     rdi, rax
    mov     rax, 60
    syscall
```

Fordítás és futtatás:
```bash
nasm -f elf64 -g -F dwarf strlen.asm -o strlen.o
ld strlen.o -o strlen
./strlen
echo $?    ; 16-ot kell kiírnia ("Hello, Assembly!" = 16 karakter)
```

---

### `strcpy` implementáció

```nasm
; ============================================================
; strcpy.asm – String másolás (null-terminált)
; Fordítás:
;   nasm -f elf64 -g -F dwarf strcpy.asm -o strcpy.o
;   ld strcpy.o -o strcpy
;   ./strcpy
; ============================================================

section .data
    forras  db  "Hello, World!", 0
    newline db  10

section .bss
    cel     resb 32     ; cél puffer (elég nagy)

section .text
    global _start

; -------------------------------------------------------
; strcpy(rdi=cel, rsi=forras) -> rax=cel
; -------------------------------------------------------
strcpy:
    push    rdi         ; eredeti cél mentése
    cld                 ; DF=0, előre haladás
.loop:
    lodsb               ; AL ← [RSI], RSI++
    stosb               ; [RDI] ← AL, RDI++
    test    al, al      ; null byte volt?
    jnz     .loop       ; nem: folytassuk
    pop     rax         ; RAX = eredeti cél cím (visszatérési érték)
    ret

_start:
    lea     rdi, [rel cel]
    lea     rsi, [rel forras]
    call    strcpy

    ; Írjuk ki a másolt stringet
    mov     rax, 1          ; sys_write
    mov     rdi, 1          ; stdout
    lea     rsi, [rel cel]
    mov     rdx, 13         ; "Hello, World!" = 13 karakter
    syscall

    ; Newline
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel newline]
    mov     rdx, 1
    syscall

    ; Kilépés
    xor     rdi, rdi
    mov     rax, 60
    syscall
```

---

## 4. Struktúrák szimulálása

A C `struct` a memóriában egymás után elhelyezett mezők gyűjteménye. Assembly-ben mi magunk kezeljük az offszetteket.

**Java párhuzam:** Egy Java objektum a heap-en szintén egymás utáni mezők – csak a JVM elrejti előlünk az offseteket. Assembly-ben mi vagyunk a JVM.

### Kézi offset kezelés EQU-val

```nasm
; Point struktúra: { int32 x; int32 y; }
POINT_X     equ 0   ; x mező offsetje: 0 byte
POINT_Y     equ 4   ; y mező offsetje: 4 byte
POINT_SIZE  equ 8   ; teljes méret
```

Használat:
```nasm
section .bss
    pont    resb POINT_SIZE     ; Point struktúra helye

; Mezők beállítása:
    mov     dword [pont + POINT_X], 10
    mov     dword [pont + POINT_Y], 20

; Olvasás:
    mov     eax, [pont + POINT_X]   ; eax = 10
```

### NASM `struc`/`endstruc` direktíva

A NASM beépített direktívával formálisabban is definiálhatunk struktúrát:

```nasm
struc Point
    .x:     resd 1      ; int32 x (4 byte)
    .y:     resd 1      ; int32 y (4 byte)
endstruc

; Méret: Point_size (automatikusan definiálva)
; Mezők: Point.x = 0, Point.y = 4
```

### Teljes példa: Point struktúra

```nasm
; ============================================================
; struct_point.asm – Point struktúra szimulálása
; Fordítás:
;   nasm -f elf64 -g -F dwarf struct_point.asm -o struct_point.o
;   ld struct_point.o -o struct_point
;   ./struct_point
;   echo $?    ; 30-at kell mutatni (10+20)
; ============================================================

section .text
    global _start

; Struktúra definíció (offszet konstansok)
POINT_X     equ 0
POINT_Y     equ 4
POINT_SIZE  equ 8

; -------------------------------------------------------
; point_sum(rdi = *pont) -> eax = x + y
; -------------------------------------------------------
point_sum:
    mov     eax, [rdi + POINT_X]    ; eax = pont->x
    add     eax, [rdi + POINT_Y]    ; eax += pont->y
    ret

_start:
    ; Struktúra a stack-en (8 byte)
    sub     rsp, 8              ; helyet foglalunk

    ; pont.x = 10
    mov     dword [rsp + POINT_X], 10
    ; pont.y = 20
    mov     dword [rsp + POINT_Y], 20

    ; Függvény hívása
    mov     rdi, rsp            ; RDI = &pont
    call    point_sum           ; EAX = 30

    ; Kilépés az összeggel
    movzx   rdi, eax
    add     rsp, 8              ; stack visszaállítás
    mov     rax, 60
    syscall
```

---

### Tömbök struktúrákból (struct array)

```nasm
; Tömb három Point struktúrából
section .bss
    pontok  resb 3 * POINT_SIZE     ; Point pontok[3]

; pontok[1].x elérése:
;   base + index * méret + mező_offset
;   pontok + 1 * 8 + 0
mov     eax, [pontok + 1 * POINT_SIZE + POINT_X]
```

---

## 5. Tömbök kezelése

### Tömb deklarálása

```nasm
section .data
    ; Inicializált tömb (int32)
    tomb    dd  10, 20, 30, 40, 50
    TOMB_MERET  equ 5

section .bss
    ; Nem inicializált tömb (int64, 10 elem)
    nagy_tomb   resq 10
```

### Elem elérése indexeléssel

Az x86-64 `[base + index * scale + offset]` címzési módot támogat, ahol `scale` = 1, 2, 4 vagy 8.

```nasm
; tomb[i] elérése (int32, 4 byte elemenként):
; EAX = tomb[2]
mov     eax, [tomb + 2 * 4]     ; közvetlen offset (ha i ismert)

; Regiszterrel (ha i változó):
mov     rbx, 2                   ; i = 2
mov     eax, [tomb + rbx * 4]   ; EAX = tomb[i]
```

### Tömb bejárása ciklussal

```nasm
; Tömb kiírása (minden elem):
    xor     rbx, rbx                ; i = 0
.loop:
    cmp     rbx, TOMB_MERET
    jge     .done
    mov     eax, [tomb + rbx * 4]  ; EAX = tomb[i]
    ; ... feldolgozás ...
    inc     rbx
    jmp     .loop
.done:
```

---

### Teljes példa 1: Tömb elemeinek összegzése

```nasm
; ============================================================
; tomb_osszeg.asm – int32 tömb elemeinek összegzése
; Fordítás:
;   nasm -f elf64 -g -F dwarf tomb_osszeg.asm -o tomb_osszeg.o
;   ld tomb_osszeg.o -o tomb_osszeg
;   ./tomb_osszeg
;   echo $?    ; 150-et kell mutatni (10+20+30+40+50)
; ============================================================

section .data
    tomb        dd  10, 20, 30, 40, 50
    TOMB_MERET  equ 5

section .text
    global _start

; -------------------------------------------------------
; tomb_sum(rdi=*tomb, rsi=meret) -> eax = összeg
; -------------------------------------------------------
tomb_sum:
    xor     eax, eax            ; összeg = 0
    xor     rcx, rcx            ; i = 0
.loop:
    cmp     rcx, rsi            ; i < meret?
    jge     .done
    add     eax, [rdi + rcx * 4]    ; összeg += tomb[i]
    inc     rcx
    jmp     .loop
.done:
    ret

_start:
    lea     rdi, [rel tomb]
    mov     rsi, TOMB_MERET
    call    tomb_sum            ; EAX = 150

    ; Kilépés az összeggel
    movzx   rdi, eax
    mov     rax, 60
    syscall
```

---

### Teljes példa 2: Maximum keresés tömbben

```nasm
; ============================================================
; tomb_max.asm – Maximum keresés int32 tömbben
; Fordítás:
;   nasm -f elf64 -g -F dwarf tomb_max.asm -o tomb_max.o
;   ld tomb_max.o -o tomb_max
;   ./tomb_max
;   echo $?    ; 99-et kell mutatni
; ============================================================

section .data
    tomb        dd  42, 17, 99, 3, 56, 81, 7
    TOMB_MERET  equ 7

section .text
    global _start

; -------------------------------------------------------
; tomb_max(rdi=*tomb, rsi=meret) -> eax = maximum
; Feltétel: meret >= 1
; -------------------------------------------------------
tomb_max:
    mov     eax, [rdi]          ; max = tomb[0]
    mov     rcx, 1              ; i = 1
.loop:
    cmp     rcx, rsi            ; i < meret?
    jge     .done
    mov     edx, [rdi + rcx * 4]    ; edx = tomb[i]
    cmp     edx, eax
    jle     .no_update          ; tomb[i] <= max? ugrunk
    mov     eax, edx            ; max = tomb[i]
.no_update:
    inc     rcx
    jmp     .loop
.done:
    ret

_start:
    lea     rdi, [rel tomb]
    mov     rsi, TOMB_MERET
    call    tomb_max            ; EAX = 99

    movzx   rdi, eax
    mov     rax, 60
    syscall
```

---

### Teljes példa 3: `rep movsb` alapú memcpy

```nasm
; ============================================================
; memcpy_rep.asm – rep movsb alapú memóriamásolás
; Fordítás:
;   nasm -f elf64 -g -F dwarf memcpy_rep.asm -o memcpy_rep.o
;   ld memcpy_rep.o -o memcpy_rep
;   ./memcpy_rep
; ============================================================

section .data
    forras  db  "ABCDEFGHIJ", 0
    newline db  10

section .bss
    cel     resb 16

section .text
    global _start

; -------------------------------------------------------
; my_memcpy(rdi=cel, rsi=forras, rdx=meret)
; -------------------------------------------------------
my_memcpy:
    mov     rcx, rdx        ; RCX = byte-ok száma
    cld                     ; DF = 0 (előre)
    rep     movsb           ; RCX-szer: [RDI] ← [RSI], RDI++, RSI++
    ret

_start:
    lea     rdi, [rel cel]
    lea     rsi, [rel forras]
    mov     rdx, 10         ; 10 byte másolása
    call    my_memcpy

    ; Kiírás
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel cel]
    mov     rdx, 10
    syscall

    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel newline]
    mov     rdx, 1
    syscall

    xor     rdi, rdi
    mov     rax, 60
    syscall
```

---

## Összefoglalás

| Utasítás      | Forrás    | Cél       | Méret  | Leírás                          |
|---------------|-----------|-----------|--------|---------------------------------|
| `movsb`       | [RSI]     | [RDI]     | 1 B    | Másolás, RSI++, RDI++           |
| `movsq`       | [RSI]     | [RDI]     | 8 B    | Másolás, RSI+=8, RDI+=8         |
| `stosb`       | AL        | [RDI]     | 1 B    | Tárolás, RDI++                  |
| `lodsb`       | [RSI]     | AL        | 1 B    | Betöltés, RSI++                 |
| `scasb`       | AL        | [RDI]     | 1 B    | Összehasonlítás, RDI++          |
| `cmpsb`       | [RSI]     | [RDI]     | 1 B    | Összehasonlítás, RSI++, RDI++   |
| `rep movsb`   | [RSI]     | [RDI]     | RCX B  | RCX-szeres másolás              |
| `repne scasb` | AL        | [RDI]     | —      | Keresés, megáll egyezéskor      |
| `repe cmpsb`  | [RSI]     | [RDI]     | —      | Összehasonlítás, megáll eltérésnél |

---

## Gyakorlatok

### 1. String összehasonlítás
Írj egy `strcmp` függvényt, amely összehasonlít két null-terminált stringet (RDI és RSI), és visszaad:
- 0-t, ha egyenlők
- pozitívat, ha az első nagyobb
- negatívat, ha a második nagyobb

*Tipp: Használd `repe cmpsb`-t, majd vizsgáld meg a flag-eket.*

### 2. Memória feltöltés
Írj egy `memset` függvényt: `memset(rdi=cel, rsi=ertek, rdx=meret)` – tölts fel `rdx` byte-ot `rsi` értékkel, `rep stosb` használatával.

### 3. Struktúra tömb
Definiálj egy `Student` struktúrát két mezővel: `kor` (int32) és `pontszam` (int32). Hozz létre 3 elemű tömbje, töltsd fel adatokkal, majd keresd meg a legjobb pontszámú diákot.

### 4. String keresés
Írj egy `strchr` függvényt: `strchr(rdi=string, rsi=karakter)` – keresse meg az első előfordulást, és adja vissza a cím pointerét (vagy 0-t, ha nem találja).

*Tipp: Mozgasd RSI-ből a karaktert AL-ba, majd `repne scasb`.*

### 5. Tömb fordítása
Írj egy függvényt, amely egy int32 tömb elemeit visszafelé rendezi a helyben (in-place reverse). Két pointert használj (eleje és vége), és cseréld fel az elemeket, amíg össze nem érnek.

---

## Ellenőrizd magad

1. Mit csinál a `cld` utasítás, és miért fontos string műveleteknél?
2. Mi a különbség a `rep` és a `repne` prefix között?
3. Hogyan oldja meg a `strlen_fast` implementáció, hogy nem kell előre tudni a string hosszát?
4. Ha egy `Point` struktúra `{ int32 x; int32 y; int32 z; }`, mi az `y` mező offsetje?
5. `[tomb + rbx * 4]` – miért 4 a szorzó, ha int32 elemeket tárolunk?
6. Mi történik, ha elfelejtjük a null terminátort a string végére tenni, és `repne scasb`-vel keressük?

---

*Következő fejezet: **9. NAP – Debugging, optimalizálás és a valós világ** →*
