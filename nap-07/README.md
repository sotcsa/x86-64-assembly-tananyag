# 7. nap – Syscall-ok: I/O és fájlkezelés

## Mit tanulunk ma?

Ma megismerjük a **syscall** mechanizmust — azt a kaput, amelyen keresztül a programunk az operációs rendszer kernelével kommunikál. Megtanuljuk, hogyan írunk szöveget a terminálra, hogyan olvasunk a billentyűzetről, és hogyan kezelünk fájlokat. Ez az a pont, ahol az Assembly programok "élővé" válnak és valódi I/O-t végeznek.

**Java párhuzam:** Javában a `System.out.println()` mögött JNI hívások, natív kódtárak, és végül kernel rendszerhívások sorozata áll. Assembly-ben mindezeket te hívod meg közvetlenül — egy `syscall` utasítással átlépünk a user space-ből a kernel space-be.

---

## Tartalom

1. [Mi az a syscall?](#1-mi-az-a-syscall)
2. [Legfontosabb syscall-ok táblázata](#2-legfontosabb-syscall-ok-táblázata)
3. [Standard I/O](#3-standard-io)
4. [Fájlkezelés](#4-fájlkezelés)
5. [Hibakezelés](#5-hibakezelés)
6. [Teljes példaprogram: Echo](#6-teljes-példaprogram-echo)
7. [Teljes példaprogram: Fájlmásoló](#7-teljes-példaprogram-fájlmásoló)
8. [Gyakorlatok](#8-gyakorlatok)
9. [Ellenőrizd magad](#9-ellenőrizd-magad)

---

## 1. Mi az a syscall?

### 1.1 User space vs. Kernel space

A Linux két elkülönített védelmi szinten működik:

```
┌─────────────────────────────────────┐
│          USER SPACE                 │
│  A programod itt fut                │
│  (korlátozott jogosultságok)        │
├─────────────────────────────────────┤
│       ── syscall határ ──           │
├─────────────────────────────────────┤
│          KERNEL SPACE               │
│  Linux kernel (teljhatalmú)         │
│  Hardver vezérlők                   │
│  Fájlrendszer, hálózat, memória     │
└─────────────────────────────────────┘
```

A user space programok **nem** férnek hozzá közvetlenül a hardverhez. Ha pl. fájlt kell írni, meg kell kérni a kernelt: "Kerlek, írd be ezt a bytes-t a fájlba." Ez a kérés a **syscall** (rendszerhívás).

### 1.2 A `syscall` utasítás működése

A `syscall` utasítás x86-64-en:
1. CPU-t kernelmódba kapcsolja (ring 0)
2. A kernel megkapja a vezérlést
3. A kernel elvégzi a kért műveletet
4. Visszatér user módba, az eredménnyel a `rax` regiszterben

```nasm
; Séma: hogyan hívunk meg egy syscallt
mov rax, <syscall_szám>    ; melyik syscallt akarjuk?
mov rdi, <1. argumentum>   ; 1. paraméter
mov rsi, <2. argumentum>   ; 2. paraméter
mov rdx, <3. argumentum>   ; 3. paraméter
; r10, r8, r9 = 4-6. paraméter (ritkán kell)
syscall                    ; ugrás kernelbe
; Visszatérés után: rax = eredmény (vagy negatív hibakód)
```

> **💡 Tipp:** A syscall konvenció **különbözik** a System V AMD64 ABI függvényhívási konvenciótól! A 4. argumentum **r10** (nem rcx), és maga a `syscall` utasítás felülírja rcx-t és r11-t.

### 1.3 Syscall konvenció összefoglalása

| Regiszter | Szerep |
|-----------|--------|
| `rax` | Syscall szám (bemenet) + visszatérési érték (kimenet) |
| `rdi` | 1. argumentum |
| `rsi` | 2. argumentum |
| `rdx` | 3. argumentum |
| `r10` | 4. argumentum |
| `r8`  | 5. argumentum |
| `r9`  | 6. argumentum |

**A syscall felülírja:** `rcx`, `r11` (és természetesen `rax`)
**Megőrzi:** `rbx`, `rbp`, `r12`–`r15`, `rsp`

### 1.4 Visszatérési érték és hibakezelés

```nasm
syscall
; rax tartalmazza:
;   >= 0  → sikeres, a visszatérési érték (pl. fájl descriptor, bájtok száma)
;   < 0   → hiba, értéke: -errno  (pl. -2 = ENOENT = "No such file")
```

**Java párhuzam:**
```java
// Java-ban: FileNotFoundException dobódik
// Assembly-ben: rax = -2 (ENOENT), te ellenőrzöd
FileOutputStream f = new FileOutputStream("nem_letezik.txt"); // throws
```

---

## 2. Legfontosabb syscall-ok táblázata

Az összes Linux syscall szám megtalálható: `/usr/include/asm/unistd_64.h`

| Szám | Név | Leírás | rdi | rsi | rdx |
|------|-----|--------|-----|-----|-----|
| 0 | `sys_read` | Adatot olvas | fd | puffer cím | max bájtok |
| 1 | `sys_write` | Adatot ír | fd | adat cím | bájtok száma |
| 2 | `sys_open` | Fájlt nyit | fájlnév cím | flags | mode |
| 3 | `sys_close` | Fájlt zár | fd | — | — |
| 9 | `sys_mmap` | Memóriát foglal | addr | length | prot |
| 12 | `sys_brk` | Heap határt állít | addr | — | — |
| 22 | `sys_pipe` | Pipe-ot hoz létre | pipefd cím | — | — |
| 32 | `sys_dup` | FD másolás | fd | — | — |
| 57 | `sys_fork` | Folyamatot klónoz | — | — | — |
| 59 | `sys_execve` | Programot futtat | fájlnév | argv | envp |
| 60 | `sys_exit` | Programot lezár | exit kód | — | — |
| 87 | `sys_unlink` | Fájlt töröl | fájlnév cím | — | — |

> **💡 Tipp:** Ellenőrizd a számokat: `grep -E "^#define __NR_write" /usr/include/asm/unistd_64.h`

### Fájlmegnyitási flag-ek (`sys_open` rsi argumentuma)

Ezek a konstansok `/usr/include/fcntl.h`-ban találhatók:

| Flag | Érték (hex) | Leírás |
|------|-------------|--------|
| `O_RDONLY` | `0x000` | Csak olvasás |
| `O_WRONLY` | `0x001` | Csak írás |
| `O_RDWR`   | `0x002` | Olvasás és írás |
| `O_CREAT`  | `0x040` | Létrehozás ha nem létezik |
| `O_TRUNC`  | `0x200` | Fájl tartalmának törlése megnyitáskor |
| `O_APPEND` | `0x400` | Hozzáfűzés a végéhez |

Kombinálás: bitenkénti OR: `O_WRONLY | O_CREAT | O_TRUNC` = `0x001 | 0x040 | 0x200` = `0x241`

```nasm
; Konstansok definiálása
O_RDONLY    EQU 0
O_WRONLY    EQU 1
O_RDWR      EQU 2
O_CREAT     EQU 0x40
O_TRUNC     EQU 0x200
O_APPEND    EQU 0x400
```

---

## 3. Standard I/O

### 3.1 Standard fájldescriptorok

Linux minden processnek három előre megnyitott fájldescriptora van:

| FD | Neve | Leírás |
|----|------|--------|
| 0 | `stdin` | Billentyűzet bemenet |
| 1 | `stdout` | Normál kimenet (terminál) |
| 2 | `stderr` | Hibaüzenet kimenet |

```nasm
STDIN   EQU 0
STDOUT  EQU 1
STDERR  EQU 2
```

### 3.2 Szöveg kiírása stdout-ra (`sys_write`)

```nasm
; sys_write(fd=1, buf=üzenet, count=hossz)
mov rax, 1              ; syscall szám: write
mov rdi, STDOUT         ; fd = 1
mov rsi, uzenet         ; puffer cím
mov rdx, uzenet_hossz   ; bájtok száma
syscall
```

A `write` visszatérési értéke: ténylegesen kiírt bájtok száma (vagy negatív hiba).

### 3.3 Szöveg olvasása stdin-ről (`sys_read`)

```nasm
; sys_read(fd=0, buf=puffer, count=max_meret)
mov rax, 0              ; syscall szám: read
mov rdi, STDIN          ; fd = 0
mov rsi, puffer         ; hova kerüljön az adat
mov rdx, MAX_MERET      ; maximum bájtok száma
syscall
; rax = ténylegesen olvasott bájtok száma (0 = EOF, negatív = hiba)
```

> **⚠️ Figyelem:** Az `Enter` leütése is a pufferbe kerül (`\n` = 0x0A bájt). Ha stringet vársz és a newline-t le akarod kezelni, ellenőrizd az utolsó karaktert.

### 3.4 Interaktív program

Az alábbi program bekér egy nevet és visszaköszön:

```nasm
; udvozles.asm
; Interaktív program: bekéri a nevet, kiírja az üdvözlést
; Fordítás:
;   nasm -f elf64 -g -F dwarf udvozles.asm -o udvozles.o
;   ld udvozles.o -o udvozles
;   ./udvozles

section .data
    kerdes      db "Hogy hivnak? ", 0
    kerdes_h    EQU $ - kerdes - 1       ; hossz a null terminator nelkul

    udvozles_e  db "Szia, "
    udvozles_eh EQU $ - udvozles_e

section .bss
    nev         resb 64         ; max 64 karakter a névnek
    nev_h       resq 1          ; ide tároljuk a ténylegesen olvasott hosszt

section .text
global _start

_start:
    ; 1. Kérdés kiírása
    mov rax, 1
    mov rdi, 1
    mov rsi, kerdes
    mov rdx, kerdes_h
    syscall

    ; 2. Név beolvasása
    mov rax, 0
    mov rdi, 0
    mov rsi, nev
    mov rdx, 64
    syscall
    mov [nev_h], rax        ; eltároljuk a beolvasott bájtok számát

    ; 3. "Szia, " kiírása
    mov rax, 1
    mov rdi, 1
    mov rsi, udvozles_e
    mov rdx, udvozles_eh
    syscall

    ; 4. A név kiírása (amit az Enter-rel együtt olvastunk be)
    mov rax, 1
    mov rdi, 1
    mov rsi, nev
    mov rdx, [nev_h]        ; a beolvasott hossz (tartalmazza a \n-t is)
    syscall

    ; 5. Kilépés
    mov rax, 60
    xor rdi, rdi
    syscall
```

---

## 4. Fájlkezelés

### 4.1 Fájl megnyitása: `sys_open` (syscall 2)

```nasm
; sys_open(pathname, flags, mode)
mov rax, 2              ; syscall szám: open
mov rdi, fajlnev        ; null-terminált string: a fájl neve
mov rsi, flags          ; O_RDONLY / O_WRONLY | O_CREAT | ... stb.
mov rdx, mode           ; jogosultságok létrehozáskor (pl. 0o644)
syscall
; rax = fájl descriptor (≥ 0), vagy negatív hibakód
```

**Mode értékek (oktálisan):**

| Mód | Oktál | Leírás |
|-----|-------|--------|
| Tulajdonos olvashat + írhat | `0o644` | `-rw-r--r--` |
| Tulajdonos mindenre jogosult | `0o755` | `-rwxr-xr-x` |
| Csak tulajdonos olvashat | `0o600` | `-rw-------` |

```nasm
; Oktál konstansok NASM-ban: 0o prefix vagy q prefix
; NASM-ban az oktál jelölés: 0644q  vagy  0o644
MODE_644    EQU 0o644    ; -rw-r--r--
```

### 4.2 Fájlba írás: `sys_write` (syscall 1)

Pontosan ugyanaz, mint stdout-ra írás, csak az `rdi` más (nem 1, hanem az open által visszaadott fd):

```nasm
; sys_write(fd, buf, count)
mov rax, 1
mov rdi, [fajl_fd]      ; az open által visszaadott fd
mov rsi, adat
mov rdx, adat_hossz
syscall
```

### 4.3 Fájlból olvasás: `sys_read` (syscall 0)

```nasm
; sys_read(fd, buf, count)
mov rax, 0
mov rdi, [fajl_fd]
mov rsi, puffer
mov rdx, MAX_MERET
syscall
; rax = olvasott bájtok, 0 = EOF
```

### 4.4 Fájl bezárása: `sys_close` (syscall 3)

```nasm
; sys_close(fd)
mov rax, 3
mov rdi, [fajl_fd]
syscall
```

> **⚠️ Figyelem:** Mindig zárd be a fájlt! A nem bezárt fájl descriptor kiszivárog, és a kernel limit (`/proc/sys/fs/file-max`) elérésekor "Too many open files" hibát kapsz — Assembly-ben nincs automatikus erőforrás-felszabadítás (nincs `try-with-resources`, nincs GC).

### 4.5 Teljes példa: fájl létrehozása és szöveg beírása

```nasm
; fajl_iras.asm
; Létrehoz egy fájlt és beleír egy üzenetet
; Fordítás:
;   nasm -f elf64 -g -F dwarf fajl_iras.asm -o fajl_iras.o
;   ld fajl_iras.o -o fajl_iras
;   ./fajl_iras
;   cat kimenet.txt

section .data
    fajlnev     db "kimenet.txt", 0
    uzenet      db "Hello, fajlrendszer!", 10    ; 10 = '\n'
    uzenet_h    EQU $ - uzenet

    ; Flag-ek: O_WRONLY | O_CREAT | O_TRUNC
    O_WRONLY    EQU 1
    O_CREAT     EQU 0x40
    O_TRUNC     EQU 0x200
    FLAGS       EQU O_WRONLY | O_CREAT | O_TRUNC

    MODE        EQU 0o644

    hiba_msg    db "Hiba: nem sikerult megnyitni a fajlt!", 10
    hiba_h      EQU $ - hiba_msg

section .bss
    fd          resq 1          ; fájl descriptor

section .text
global _start

_start:
    ; --- 1. Fájl megnyitása/létrehozása ---
    mov rax, 2              ; sys_open
    mov rdi, fajlnev        ; "kimenet.txt"
    mov rsi, FLAGS          ; O_WRONLY | O_CREAT | O_TRUNC
    mov rdx, MODE           ; 0644
    syscall

    ; Hibakezelés: ha rax < 0, hiba történt
    test rax, rax
    js .hiba                ; jump if sign (negatív → hiba)

    mov [fd], rax           ; fd mentése

    ; --- 2. Írás a fájlba ---
    mov rax, 1              ; sys_write
    mov rdi, [fd]           ; a fájl fd-je
    mov rsi, uzenet
    mov rdx, uzenet_h
    syscall

    ; --- 3. Fájl bezárása ---
    mov rax, 3              ; sys_close
    mov rdi, [fd]
    syscall

    ; --- 4. Sikeres kilépés ---
    mov rax, 60
    xor rdi, rdi
    syscall

.hiba:
    ; Hibaüzenet stdout-ra
    mov rax, 1
    mov rdi, 2              ; stderr (fd=2)
    mov rsi, hiba_msg
    mov rdx, hiba_h
    syscall

    mov rax, 60
    mov rdi, 1              ; exit(1) = hiba
    syscall
```

---

## 5. Hibakezelés

### 5.1 Negatív visszatérési érték ellenőrzése

A syscall-ok hiba esetén negatív értékkel térnek vissza. A negatív érték: `-errno` (ahol errno egy pozitív hibakód szám).

```nasm
    syscall
    ; Ellenőrzési módszerek:

    ; 1. test rax, rax + js (ha negatív előjelű)
    test rax, rax
    js .hiba_cimke

    ; 2. cmp explicit értékkel
    cmp rax, 0
    jl .hiba_cimke          ; jl = jump if less (signed)

    ; 3. Nagy negatív számok (hiba ha rax > 0xFFFFFFFFFFFFFF80)
    ; A hibakódok -1-től -4095-ig terjednek Linux-on
    cmp rax, -4096
    jb .hiba_cimke          ; jb = jump if below (unsigned)
```

> **💡 Tipp:** A `-errno` ellenőrzésre a `cmp rax, -4095; ja .ok` minta is használható — Linux garantálja, hogy a syscall visszatérési értékek [-4095, -1] tartományba esnek hiba esetén.

### 5.2 Fontos errno értékek

| errno | Érték | Konstans | Leírás |
|-------|-------|----------|--------|
| EPERM | 1 | | Nincs jogosultság |
| ENOENT | 2 | | Fájl/könyvtár nem létezik |
| EACCES | 13 | | Hozzáférés megtagadva |
| EEXIST | 17 | | Fájl már létezik |
| EINVAL | 22 | | Érvénytelen argumentum |
| EMFILE | 24 | | Túl sok nyitott fájl (processz limit) |
| ENOSPC | 28 | | Nincs elég hely az eszközön |
| EBADF | 9 | | Érvénytelen file descriptor |

### 5.3 strace – syscall-ok nyomon követése

A `strace` eszközzel megnézheted, milyen syscall-okat hív a programod:

```bash
# Összes syscall kilistázása
strace ./programod

# Csak write és read syscall-ok
strace -e trace=write,read ./programod

# Kimenet fájlba mentve
strace -o trace.txt ./programod

# Futó process vizsgálata PID alapján
strace -p 1234
```

Példa strace kimenet:

```
execve("./echo_prog", ["./echo_prog"], ...) = 0
read(0, "Hello\n", 4096)               = 6
write(1, "Hello\n", 6)                 = 6
exit(0)                                = ?
+++ exited with 0 +++
```

Ez megmutatja az összes rendszerhívást argumentumaikkal és visszatérési értékeikkel.

> **💡 Tipp:** Ha a programod "lefagy" vagy váratlanul viselkedik, `strace` segítségével láthatod, melyik syscall-nál akad el, és mi a hibaüzenet.

---

## 6. Teljes példaprogram: Echo

Ez a program beolvas a stdin-ről (maximum 4096 bájt / iteráció), majd kiírja stdout-ra — pontosan mint a Unix `cat` parancs bemeneti üzemmódban.

```nasm
; echo_prog.asm
; stdin-ről olvas, stdout-ra ír, amíg EOF-ot nem kap
; Fordítás:
;   nasm -f elf64 -g -F dwarf echo_prog.asm -o echo_prog.o
;   ld echo_prog.o -o echo_prog
;   ./echo_prog           (gépelj valamit, majd Ctrl+D = EOF)
;   echo "Hello World" | ./echo_prog   (pipe-ból is működik)

section .bss
    puffer      resb 4096       ; 4 KB puffer

section .text
global _start

_start:
.olvas_ciklus:
    ; --- stdin olvasása ---
    mov rax, 0              ; sys_read
    mov rdi, 0              ; stdin (fd=0)
    mov rsi, puffer
    mov rdx, 4096           ; max 4096 bájt
    syscall

    ; Ellenőrzés: 0 = EOF (Ctrl+D), negatív = hiba
    test rax, rax
    jle .vege               ; ha rax <= 0: kilépünk

    ; --- stdout kiírása ---
    ; rax = ténylegesen olvasott bájtok száma
    push rax                ; elmentjük a bájtszámot

    mov rdx, rax            ; count = olvasott bájtok
    mov rax, 1              ; sys_write
    mov rdi, 1              ; stdout (fd=1)
    mov rsi, puffer
    syscall

    ; Hibakezelés: ha write hiba
    test rax, rax
    js .iras_hiba

    pop rax                 ; (a push-olt értéket kidobjuk, már nem kell)
    jmp .olvas_ciklus       ; következő iteráció

.iras_hiba:
    pop rax                 ; stack egyensúly helyreállítása

.vege:
    mov rax, 60             ; sys_exit
    xor rdi, rdi            ; exit(0)
    syscall

; Tesztelés:
;   printf "Teszt szoveg\nMasodik sor\n" | ./echo_prog
; Várható kimenet:
;   Teszt szoveg
;   Masodik sor
```

---

## 7. Teljes példaprogram: Fájlmásoló

Ez a program parancssori argumentumok nélkül, két hardkódolt fájlnévvel dolgozik — beolvassa a `forrás.txt`-t és átírja `cel.txt`-be. (Egy igazi `cp` klón parancssori argumentumokat kezelne — ez a 10. nap témája.)

```nasm
; fajl_masolo.asm
; Bemásolja a "forras.txt" fájlt "cel.txt"-be
; Fordítás:
;   nasm -f elf64 -g -F dwarf fajl_masolo.asm -o fajl_masolo.o
;   ld fajl_masolo.o -o fajl_masolo
;   echo "Ez a forras tartalma!" > forras.txt
;   ./fajl_masolo
;   cat cel.txt

section .data
    forras_nev  db "forras.txt", 0
    cel_nev     db "cel.txt", 0

    ; Open flag-ek
    O_RDONLY    EQU 0
    O_WRONLY    EQU 1
    O_CREAT     EQU 0x40
    O_TRUNC     EQU 0x200
    WR_FLAGS    EQU O_WRONLY | O_CREAT | O_TRUNC
    MODE_644    EQU 0o644

    ; Hibaüzenetek
    hiba_forras db "Hiba: nem nyithato meg forras.txt", 10
    hiba_forras_h EQU $ - hiba_forras
    hiba_cel    db "Hiba: nem nyithato meg cel.txt", 10
    hiba_cel_h  EQU $ - hiba_cel

section .bss
    forras_fd   resq 1          ; forrás fájl descriptor
    cel_fd      resq 1          ; cél fájl descriptor
    puffer      resb 4096       ; másolási puffer

section .text
global _start

_start:
    ; ====== 1. Forrás fájl megnyitása (olvasásra) ======
    mov rax, 2
    mov rdi, forras_nev
    mov rsi, O_RDONLY
    xor rdx, rdx                ; mode = 0 (nem létrehozunk fájlt)
    syscall

    test rax, rax
    js .hiba_forras_megnyit

    mov [forras_fd], rax

    ; ====== 2. Cél fájl megnyitása/létrehozása (írásra) ======
    mov rax, 2
    mov rdi, cel_nev
    mov rsi, WR_FLAGS
    mov rdx, MODE_644
    syscall

    test rax, rax
    js .hiba_cel_megnyit

    mov [cel_fd], rax

    ; ====== 3. Másolási ciklus ======
.masolasi_ciklus:
    ; Olvasás a forrásból
    mov rax, 0                  ; sys_read
    mov rdi, [forras_fd]
    mov rsi, puffer
    mov rdx, 4096
    syscall

    ; 0 = EOF, negatív = hiba, pozitív = olvasott bájtok
    test rax, rax
    jz .masolasi_vege           ; EOF: befejezés
    js .hiba_olvad               ; negatív: hiba

    ; Megőrizzük az olvasott bájtok számát
    mov r12, rax                ; r12: callee-saved, biztonságos

    ; Írás a célba
    mov rax, 1                  ; sys_write
    mov rdi, [cel_fd]
    mov rsi, puffer
    mov rdx, r12                ; pontosan annyi bájt, amennyit olvastunk
    syscall

    test rax, rax
    js .hiba_iras

    jmp .masolasi_ciklus        ; következő blokk

    ; ====== 4. Fájlok bezárása ======
.masolasi_vege:
    mov rax, 3
    mov rdi, [forras_fd]
    syscall

    mov rax, 3
    mov rdi, [cel_fd]
    syscall

    ; Sikeres kilépés
    mov rax, 60
    xor rdi, rdi
    syscall

    ; ====== Hibakezelők ======
.hiba_forras_megnyit:
    mov rax, 1
    mov rdi, 2
    mov rsi, hiba_forras
    mov rdx, hiba_forras_h
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

.hiba_cel_megnyit:
    ; Forrást be kell zárni mielőtt kilépünk
    mov rax, 3
    mov rdi, [forras_fd]
    syscall

    mov rax, 1
    mov rdi, 2
    mov rsi, hiba_cel
    mov rdx, hiba_cel_h
    syscall
    mov rax, 60
    mov rdi, 1
    syscall

.hiba_olvad:
.hiba_iras:
    ; Mindkét fájlt bezárjuk
    mov rax, 3
    mov rdi, [forras_fd]
    syscall
    mov rax, 3
    mov rdi, [cel_fd]
    syscall
    mov rax, 60
    mov rdi, 1
    syscall
```

**Tesztelés:**

```bash
nasm -f elf64 -g -F dwarf fajl_masolo.asm -o fajl_masolo.o
ld fajl_masolo.o -o fajl_masolo

echo "Ez a forras tartalma!" > forras.txt
./fajl_masolo
cat cel.txt     # kimenet: Ez a forras tartalma!

# strace-szel ellenőrizve:
strace ./fajl_masolo
```

---

## 8. Gyakorlatok

### 1. feladat – Számkiírás
Írj programot, amely kiírja az 1-től 5-ig terjedő számokat (szövegként, pl. "1\n2\n3\n4\n5\n") stdout-ra. Tipp: a számjegyből karakter lesz, ha hozzáadsz `'0'`-t (ASCII 48).

### 2. feladat – Fájl szöveges elemzése
Olvasd be a `forras.txt` tartalmát, és számold meg, hány newline karakter (`\n` = 10) van benne. Írd ki a sorok számát exit kódként (`echo $?`).

### 3. feladat – Hibakód ellenőrzés
Módosítsd a `fajl_iras.asm`-t úgy, hogy ha az `open` syscall hiba miatt negatív értékkel tér vissza, az exit kód legyen a negatív errno abszolút értéke (pl. `ENOENT` esetén `exit(2)`).

### 4. feladat – Fájl hozzáfűzés
Módosítsd a `fajl_iras.asm`-t: nyisd meg a fájlt `O_WRONLY | O_CREAT | O_APPEND` flagekkel, és futtasd le háromszor egymás után. A `cat kimenet.txt` eredménye az üzenet háromszor egymás után legyen.

### 5. feladat – strace elemzés
Fordítsd le az `echo_prog.asm`-t, és futtasd `strace`-szel:
```bash
echo "Hello strace" | strace ./echo_prog 2>trace.txt
cat trace.txt
```
Azonosítsd a kimenetben a `read` és `write` syscall-okat, azok argumentumait és visszatérési értékeit!

---

## 9. Ellenőrizd magad

1. **Melyik regiszterbe** kell tenni a syscall számát, és melyikbe kerül a visszatérési érték?

2. **Mi a különbség** a syscall konvenció és a System V AMD64 ABI konvenció között? (Különösen: melyik regiszter adja a 4. argumentumot?)

3. **Mit jelent** ha egy syscall negatív értékkel tér vissza? Hogyan ellenőrzöd hiba esetén a konkrét errno értéket?

4. **Sorolj fel** három `open` flag-et, és magyarázd el mindegyik hatását!

5. **Miért kell mindig** bezárni a fájlt (`sys_close`)? Mi történik ha nem teszed meg?

6. **Hogyan tudod** megnézni futás közben, hogy pontosan milyen syscall-okat hív a programod? Írd le a parancsot és a tipikus kimenet formátumát!

---

*Előző fejezet: [6. nap – Memóriakezelés és címzési módok](../nap-06/README.md)*
*Következő fejezet: [8. nap – Stringek és adatstruktúrák](../nap-08/README.md)*
