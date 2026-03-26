# 9. NAP – Debugging, optimalizálás és a valós világ

## Mit tanulunk ma?

Eddig megtanultunk programokat írni – ma megtanuljuk őket *megérteni*, mikor nem működnek. A GDB debuggerrel belülről nézzük meg a programunkat, megismerjük a leggyakoribb hibaüzeneteket és okaikat, áttekintjük az alapvető teljesítmény-szempontokat, és megnézzük, hogyan jelenik meg az Assembly a valós DevOps/fejlesztési munkában.

**Java párhuzam:** A GDB Assembly debuggolás olyan, mint a JVM remote debugging + JMX egyszerre, csak sokkal közelebb a hardverhez. A `stepi` (step instruction) az egész JIT-compiled bytecode helyett egyetlen gépi utasítást hajt végre.

---

## 1. GDB debugger Assembly-hez

### Fordítás debug infóval

A debug szimbólumok (DWARF) elengedhetetlenek ahhoz, hogy a GDB soronként navigálhasson az assembly forráson:

```bash
nasm -f elf64 -g -F dwarf program.asm -o program.o
ld program.o -o program
```

A `-g` flag debug szimbólumokat generál, a `-F dwarf` megadja a formátumot (GDB DWARF-ot vár).

> **💡 Tipp:** Ha C könyvtárakat is linkelsz (`gcc`-vel), add hozzá a `--debug` flaget is a GDB-nek.

---

### GDB indítása

```bash
gdb ./program
```

Ez betölti a programot, de még nem futtatja. A GDB prompt: `(gdb)`.

---

### Legfontosabb GDB parancsok

#### Futtatás és navigáció

| Parancs              | Rövidítés | Leírás                                          |
|----------------------|-----------|-------------------------------------------------|
| `break _start`       | `b`       | Breakpoint az `_start` label-nél                |
| `break *0x401000`    | `b *cím`  | Breakpoint konkrét memóriacímen                 |
| `run`                | `r`       | Program indítása (breakpoint-ig fut)            |
| `stepi`              | `si`      | Egy utasítás végrehajtása (belép hívásba)       |
| `nexti`              | `ni`      | Egy utasítás végrehajtása (nem lép be hívásba)  |
| `continue`           | `c`       | Futtatás következő breakpoint-ig               |
| `finish`             | `fin`     | Aktuális függvény befejezése                    |
| `quit`               | `q`       | Kilépés GDB-ből                                 |

#### Regiszterek vizsgálata

```gdb
info registers          ; összes regiszter decimálisan
info registers rax rbx  ; csak rax és rbx
print $rax              ; rax decimálisan
print/x $rax            ; rax hexadecimálisan
print/d $rax            ; rax decimálisan (signed)
print/t $rax            ; rax bináris formában
```

#### Memória vizsgálata

```gdb
x/10x $rsp              ; 10 hex szó a stack tetejéről
x/10xb $rsi             ; 10 byte hex-ben a [RSI] től
x/5xg $rdi              ; 5 db 8-byte-os (giant) érték
x/s $rdi                ; null-terminált string kiírása
x/10i $rip              ; következő 10 utasítás disassembly
```

A `x` (examine) formátuma: `x/Nfs <cím>`
- `N` = darabszám
- `f` = formátum: `x`(hex), `d`(dec), `s`(string), `i`(instruction), `b`(byte szintű hex)
- `s` = méret: `b`(byte), `h`(halfword=2B), `w`(word=4B), `g`(giant=8B)

#### Disassembly

```gdb
disassemble             ; aktuális függvény
disassemble _start      ; az _start függvény
disassemble /m          ; forráskóddal vegyítve (ha van debug info)
```

#### TUI mód (Text User Interface)

```gdb
layout asm              ; assembly ablak megnyitása
layout regs             ; regiszter ablak + assembly
layout src              ; forrás ablak (ha van debug info)
Ctrl+x 2               ; váltás több ablak között
Ctrl+x a               ; TUI mód ki/be
```

---

### Breakpoint-ok és watchpoint-ok

```gdb
break _start            ; label-re
break *0x401020         ; konkrét cím
break 42                ; forráskód 42. sora
info breakpoints        ; aktív breakpoint-ok listája
delete 1                ; 1-es breakpoint törlése
disable 2               ; letiltás (nem törlés)

watch [rax]             ; watchpoint: ha rax értéke változik, megáll
watch *0x601000         ; memória watchpoint: ha a cím tartalma változik
info watchpoints        ; aktív watchpoint-ok
```

---

### Teljes debuggolási walkthrough

Vegyük az alábbi egyszerű programot:

```nasm
; debug_demo.asm
section .data
    tomb    dd  10, 20, 30, 40, 50
    MERET   equ 5

section .text
    global _start

osszeges:
    xor     eax, eax
    xor     rcx, rcx
.loop:
    cmp     rcx, rsi
    jge     .done
    add     eax, [rdi + rcx * 4]
    inc     rcx
    jmp     .loop
.done:
    ret

_start:
    lea     rdi, [rel tomb]
    mov     rsi, MERET
    call    osszeges
    movzx   rdi, eax
    mov     rax, 60
    syscall
```

```bash
nasm -f elf64 -g -F dwarf debug_demo.asm -o debug_demo.o
ld debug_demo.o -o debug_demo
gdb ./debug_demo
```

GDB munkamenet:

```gdb
(gdb) break _start
Breakpoint 1 at 0x401030: file debug_demo.asm, line 20.

(gdb) run
Breakpoint 1, _start () at debug_demo.asm:20

(gdb) layout regs
# Megjelenik a regiszter panel felül, assembly alul

(gdb) si          ; lea rdi, [rel tomb] végrehajtása
(gdb) si          ; mov rsi, 5

(gdb) print/x $rdi    ; Tomb címe
(gdb) print $rsi      ; 5

(gdb) si          ; call osszeges

(gdb) break osszeges.loop    ; breakpoint a ciklusba
(gdb) continue

(gdb) print $rcx          ; i értéke
(gdb) print $eax          ; aktuális összeg
(gdb) x/5d $rdi           ; tomb 5 eleme decimálisan

(gdb) continue    ; következő iteráció
(gdb) print $eax          ; összeg növekedett?

(gdb) finish      ; osszeges függvény vége
(gdb) print $eax          ; végeredmény: 150
```

---

## 2. Egyéb eszközök

### `objdump -d` – Disassembly

Az összefordított ELF bináris disassembly-jét mutatja – debugger nélkül.

```bash
objdump -d ./program
objdump -d -M intel ./program    # Intel szintaxis (nem AT&T)
objdump -d --no-show-raw-insn ./program   # gépi kód nélkül
```

Kimeneti példa:
```
0000000000401000 <_start>:
  401000:   48 8d 3d 09 00 00 00    lea    rdi,[rip+0x9]
  401007:   48 c7 c0 01 00 00 00    mov    rax,0x1
  40100e:   0f 05                   syscall
```

> **💡 Tipp:** Ha egy core dump-ból akarod visszafejteni, mi futott, `objdump -d` az első lépés.

---

### `strace` – Syscall nyomkövetés

Megmutatja, pontosan mely syscall-okat hívja a program és milyen argumentumokkal.

```bash
strace ./program
strace -e trace=write ./program    # csak write syscall-ok
strace -o kimenet.txt ./program    # fájlba mentés
```

Példa kimenet:
```
execve("./program", ["./program"], 0x... /* 20 vars */) = 0
write(1, "Hello, World!\n", 14)         = 14
exit(0)                                  = ?
+++ exited with 0 +++
```

**Java párhuzam:** A `strace` olyan, mint a JVM JNI/native hívások nyomkövetése – látod az összes OS-szintű interakciót.

---

### `ltrace` – Library call nyomkövetés

Ha C könyvtárakat linkelünk (`gcc`), az `ltrace` megmutatja a libc hívásokat:

```bash
ltrace ./program_with_libc
```

---

### `readelf -a` – ELF fejléc vizsgálat

```bash
readelf -h ./program     # ELF fejléc (architektúra, entry point, stb.)
readelf -S ./program     # section-ök (.text, .data, .bss mérete)
readelf -s ./program     # szimbólumtábla
readelf -a ./program     # összes info
```

Hasznos, ha tudni akarod: pontosan hol van az `_start` a binárisban, vagy mekkora a `.bss` szekció.

---

### `hexdump -C` – Bináris tartalom

```bash
hexdump -C ./program | head -20
hexdump -C adatfajl.bin
```

Kimenet:
```
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  02 00 3e 00 01 00 00 00  00 10 40 00 00 00 00 00  |..>.......@.....|
```

Az első 4 byte (`7f 45 4c 46` = `\x7fELF`) az ELF magic number – így azonosítja a Linux a futtatható fájlokat.

---

## 3. Gyakori hibák és hibaüzenetek

### Segmentation Fault (SIGSEGV)

**Mi ez?** A program olyan memóriacímre próbált hozzáférni, amihez nincs joga (nem leképezett, read-only, vagy kernel-terület).

**Leggyakoribb okok Assembly-ben:**

| Ok | Példa | Megoldás |
|----|-------|----------|
| Nulla pointer dereferencia | `mov rax, [0]` | Ellenőrizd a pointer értékét |
| Stack overflow (végtelen rekurzió) | `call fn` → `call fn` → … | Ellenőrizd a rekurzió leállási feltételét |
| Nem allokált `.bss` terület | `resb` hiánya | `resb N` a szükséges mérettel |
| RSP nem 16-byte-ra igazított `call` előtt | `call printf` igazítatlan stack-kel | `and rsp, -16` a hívás előtt |

```gdb
# GDB segfault diagnosztika:
(gdb) run
Program received signal SIGSEGV, Segmentation fault.
(gdb) info registers    # melyik regiszter mutat érvénytelen helyre?
(gdb) x/i $rip          # melyik utasításnál történt?
(gdb) bt                # backtrace – hívási lánc
```

---

### Bus Error (SIGBUS)

**Mi ez?** Igazítatlan memória hozzáférés – pl. egy 4-byte-os értéket páratlan memóriacímről olvasunk.

```nasm
; HIBÁS: rdi = 0x601001 (páratlan cím), de dword-öt olvasunk
mov eax, dword [rdi]    ; Bus error, ha rdi nem 4-re igazított!

; HELYES: igazított cím
mov eax, dword [rdi]    ; rdi = 0x601000 (4-re igazított)
```

> **⚠️ Figyelem:** Az x86-64 általában megengedi az igazítatlan hozzáférést (teljesítményveszteséggel), de egyes SSE/AVX utasítások szigorúan megkövetelik az igazítást.

---

### Rossz syscall szám

Ha rossz syscall számot adsz meg (pl. Linuxon x86-ból hozod át a syscall számot), a kernel `ENOSYS` (-38) hibát ad vissza RAX-ban:

```nasm
mov rax, 4      ; HELYTELEN! Linux x86-64-en write = 1, nem 4
mov rdi, 1
syscall         ; RAX = -38 (ENOSYS)
```

> **💡 Tipp:** Mindig ellenőrizd a syscall számokat: `grep -r 'write' /usr/include/asm/unistd_64.h` vagy a syscall referencia mellékletben (lásd: `mellekletek/syscall-referencia.md`).

---

### Stack alignment probléma

Az x86-64 System V ABI megköveteli, hogy `call` utasítás végrehajtásakor az RSP **16 byte-ra legyen igazítva**. A `call` maga 8 byte-ot tolja a stackre (visszatérési cím), tehát a függvény belépésekor RSP értéke 8-cal osztható, de 16-tal nem.

```nasm
; HELYTELEN: printf() SIGSEGV-t adhat
_start:
    ; RSP itt 16-ra igazított (_start-nál RSP = ... 0x...0 végű)
    ; ha call printf-t hívunk közvetlenül, OK
    ; DE ha push-oltunk valamit, vagy sub rsp, 8-at csináltunk:
    sub rsp, 8     ; RSP már nem 16-ra igazított!
    call printf    ; CRASH

; HELYES:
_start:
    sub rsp, 8
    ; Igazítás: RSP most 8-as osztójú, de nem 16-os
    ; Megoldás: legyen az összes allokáció 16-os többszöröse
    sub rsp, 16    ; 16 byte: RSP marad 16-ra igazított
    call printf
    add rsp, 16
```

---

### Hiányzó null terminátor

```nasm
; HIBÁS: nincs null terminátor, strlen végtelen ciklusba esik
uzenet  db  "Hello"         ; NULL BYTE HIÁNYZIK!

; HELYES:
uzenet  db  "Hello", 0      ; null-terminált
; vagy:
uzenet  db  "Hello"
        db  0
```

---

## 4. Teljesítmény alapok

### Pipeline és branch prediction

A modern CPU-k utasítás folyamatot (pipeline) használnak: egyszerre több utasítás különböző végrehajtási fázisban van. Ha egy feltételes ugrás eredménye bizonytalan, a CPU "megjósol" (branch prediction) – ha rosszul jósolt, a pipeline kiürül (pipeline flush), ami ~15-20 ciklus veszteséget okoz.

**Tanulság:** Kerüld az adatfüggő elágazásokat belső ciklusokban, ahol lehetséges. Részesítsd előnyben a `cmov` (conditional move) használatát:

```nasm
; Feltételes ugrás (potenciálisan lassabb misprediction esetén):
cmp eax, ebx
jl  .kisebb
mov ecx, ebx
jmp .vege
.kisebb:
mov ecx, eax
.vege:

; cmovl – feltételes mozgatás (elágazás nélkül, mindig gyors):
cmp eax, ebx
mov ecx, ebx
cmovl ecx, eax    ; ha eax < ebx: ecx = eax, különben marad ebx
```

---

### Cache-barát kód

A CPU cache (L1: ~4ns, L2: ~12ns, L3: ~40ns, RAM: ~100ns) óriási szerepet játszik. Az adatok sequential (egymás utáni) elérése a leggyorsabb – ez a **spatial locality**.

```nasm
; CACHE-BARÁT: tömb szekvenciális bejárása (minden elem egymás után)
xor rcx, rcx
.loop:
    mov eax, [tomb + rcx * 4]   ; szekvenciális hozzáférés ✓
    inc rcx
    cmp rcx, 1000
    jl  .loop

; CACHE-ROSSZ: véletlenszerű ugrások nagy tömbben
; (pl. láncolt listák bejárása pointer-követéssel)
```

---

### Felesleges memória hozzáférések kerülése

Regiszter >> Cache >> RAM. Ha egy értéket többször is használsz, tárold regiszterben:

```nasm
; LASSABB: minden iterációban memóriából olvas
.loop:
    add eax, [szamlalo]     ; memória olvasás minden ciklusban
    inc ecx
    jl  .loop

; GYORSABB: regiszterben tartjuk a számlálót
    mov edx, [szamlalo]     ; egyszer betöltjük
.loop:
    add eax, edx            ; regiszter + regiszter: 1 ciklus
    inc ecx
    jl  .loop
    mov [szamlalo], edx     ; végén visszaírjuk
```

---

### `xor reg, reg` vs. `mov reg, 0`

Nullázáshoz az `xor` hatékonyabb:

```nasm
xor eax, eax    ; 2 byte gépi kód, nincs adatfüggőség, regiszter rename
mov eax, 0      ; 5 byte gépi kód, közvetlen érték betöltés
```

A `xor eax, eax` kód 3 byte-tal rövidebb, és a processzor külön optimalizálja (zero idiom recognition).

> **💡 Tipp:** `xor eax, eax` automatikusan nullázza a **teljes RAX**-ot (64 bit) is – a 32-bites műveletek felső 32 bitje mindig törlődik x86-64-en.

---

### `lea` vs. `add` + `imul`

A `lea` (Load Effective Address) nem csak cím számításra jó – gyors összeadásra és szorzásra is használható:

```nasm
; Cél: rax = rbx * 3
imul rax, rbx, 3    ; szorzás (több ciklus)
; vs.
lea  rax, [rbx + rbx * 2]   ; rax = rbx + 2*rbx = 3*rbx (1 ciklus)

; Cél: rax = rbx + rcx + 10
add  rax, rbx
add  rax, rcx
add  rax, 10        ; 3 utasítás
; vs.
lea  rax, [rbx + rcx + 10]  ; 1 utasítás
```

---

## 5. Assembly a valós életben (DevOps szemmel)

### Core dump olvasás

Ha egy program segfault-tal crashel, a kernel core dump fájlt hagyhat hátra (ha engedélyezett):

```bash
ulimit -c unlimited         # core dump engedélyezése
./program                   # ha crashel: core fájl keletkezik
gdb ./program core          # post-mortem debugging

# GDB-ben:
(gdb) bt                    # backtrace: hol crashelt?
(gdb) info registers        # regiszter állapot a crash pillanatában
(gdb) x/20x $rsp            # stack tartalma
```

**DevOps kontextus:** Kubernetes crash-oló pod-oknál a `/proc/<pid>/maps` és core dump combo azonos módon használható – `kubectl exec` + `gdb` + core dump.

---

### JVM JIT compiler outputjának olvasása

**Java párhuzam:** A `javap -c` az. amivel bytecode-ot nézel, de az Assembly szintű JIT kódhoz más eszköz kell:

```bash
# JVM JIT output (HotSpot PrintAssembly):
java -XX:+PrintAssembly -XX:+UnlockDiagnosticVMOptions MyClass 2>&1 | head -100
```

A kimenetben ugyanazok az x86-64 utasítások szerepelnek, amelyeket már ismersz: `mov`, `add`, `cmp`, `jne` – csak generált kódban.

```bash
# javap bytecode összehasonlítás:
javap -c MyClass.class      # JVM bytecode (magas szintű)
# vs.
objdump -d ./my_native_lib.so   # natív x86-64 (alacsony szintű)
```

---

### Biztonsági sebezhetőségek megértése – Buffer Overflow alapelv

A buffer overflow az egyik legklasszikusabb biztonsági sebezhetőség, és Assembly szinten érthetjük meg igazán:

```nasm
; SEBEZHETŐ: fix méretű puffer, de korlátlan adat olvasás
section .bss
    puffer  resb 16         ; 16 byte puffer

section .text
; sys_read hívás 256 byte-ot olvas egy 16 byte-os pufferbe:
    mov     rax, 0          ; sys_read
    mov     rdi, 0          ; stdin
    lea     rsi, [puffer]   ; puffer cím
    mov     rdx, 256        ; HIBA: 256 > 16, stack/heap felülírás!
    syscall
```

Mi történik? Az extra adatok felülírják a stack-et – beleértve a **return address**-t. Ha a támadó irányítja az inputot, a program tetszőleges kódot futtathat.

**Modern védelem:** Stack canary (`-fstack-protector`), ASLR, NX bit (No-Execute) – ezek ismerete elengedhetetlen DevOps/biztonsági szempontból.

---

### Compiler Explorer (godbolt.org) bemutatása

A [Compiler Explorer](https://godbolt.org) webes eszközzel C/C++/Rust kódot fordíthatsz valós időben Assembly-vé, böngészőből – anélkül, hogy helyi eszközt kellene telepíteni.

```
Bal oldal: C forrás
    int add(int a, int b) { return a + b; }

Jobb oldal: x86-64 Assembly (gcc -O2):
    add(int, int):
        lea eax, [rdi+rsi]
        ret
```

**Hogyan használd?**
1. Menj a [godbolt.org](https://godbolt.org)-ra
2. Válaszd a `x86-64 gcc` compilert
3. Kísérletezz az optimalizálási szintekkel: `-O0` (debug), `-O2` (production), `-O3` (agresszív)
4. Nézd meg, hogyan változik az Assembly kód

> **💡 Tipp:** DevOps-ban hasznos lehet megérteni, miért lassabb egy `.jar` native library nélkül – a Compiler Explorer megmutatja, hogy egy egyszerű Java-ban megírt ciklus mennyi gépi utasítássá válik vs. az optimalizált C változat.

---

## Összefoglalás: Debugging checklist

Mielőtt GDB-be nyúlsz, ellenőrizd:

1. **Szegmentálási hiba?** → Ellenőrizd a pointereket, stack igazítást, `.bss` méreteket
2. **Rossz eredmény?** → `info registers` minden kritikus ponton, `print/x` értékek
3. **Program lefagy?** → Végtelen ciklus? `Ctrl+C` GDB-ben, majd `bt`
4. **Syscall hibák?** → `strace ./program` – RAX negatív értéke errno

---

## Gyakorlatok

### 1. GDB sesszion
Vedd a 8. nap `tomb_max.asm` programját, fordítsd debug szimbólumokkal, és debuggold végig GDB-vel. Helyezz breakpoint-ot a ciklus elejére, és minden iterációban nézd meg `$rcx`, `$eax` és `$edx` értékét. Dokumentáld a megfigyeléseket.

### 2. strace elemzés
Futtasd `strace`-szel a 7. nap fájlkezelő programját. Listázd ki, pontosan milyen syscall-okat hívott és milyen argumentumokkal. Hasonlítsd össze a forrással.

### 3. Szándékos hiba debug
Módosítsd a `tomb_sum.asm` programot úgy, hogy az `MERET` értéke helyett eggyel nagyobb számot adsz meg (`TOMB_MERET + 1`). Futtasd, figyeld meg a jelenséget, majd debuggold GDB-vel hogy mi történik az utolsó iterációban.

### 4. objdump vizsgálat
Fordítsd le a `hello_world.asm` programot, majd futtasd `objdump -d -M intel ./hello_world` parancsot. Azonosítsd a `.text` szekció boundary-jait és az `_start` belépési pontot.

### 5. Teljesítmény összehasonlítás
Írj két `strlen` implementációt: egy naiv byte-onkénti verziót és egy `repne scasb` alapú verziót. Mérd meg mindkettő futási idejét 1 000 000 karakteres string-en (`time ./program`). Melyik gyorsabb és miért?

---

## Ellenőrizd magad

1. Mi a különbség a `stepi` és `nexti` GDB parancs között?
2. Mit jelent az `x/10xg $rsp` parancs: mit ír ki és milyen formában?
3. Miért okoz segmentation fault-ot, ha `mov rax, [0]`-t futtatunk?
4. Mi a stack alignment követelmény x86-64 System V ABI-ban, és miért fontosabb, ha `printf`-et hívunk?
5. Miért hatékonyabb `xor eax, eax` a `mov eax, 0`-nál?
6. Mi az a `strace` és mikor használod szívesebben, mint a GDB-t?

---

*Következő fejezet: **10. NAP – Záró projekt és továbblépés** →*
