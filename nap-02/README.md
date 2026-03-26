# 2. nap: Első program – NASM szintaxis, szekciók, Hello World, fordítás és linkelés

## Mit tanulunk ma?

Ma megírjuk az első futtatható assembly programunkat. Megismerjük a NASM assembler szintaxisát, a szekciók fogalmát, és végigmegyünk a teljes fordítási folyamaton: `.asm` forráskódtól az ELF futtatható fájlig. A nap végén képes leszel önállóan megírni, lefordítani és futtatni egy "Hello, World!" programot – és pontosan érteni fogod, hogy az egyes sorok mit csinálnak.

**Előfeltétel:** 1. nap anyaga (CPU architektúra, regiszterek alapjai, számrendszerek)

---

## 1. Fejlesztői környezet beállítása

### Szükséges csomagok telepítése

```bash
sudo apt update
sudo apt install nasm binutils gdb strace
```

| Csomag     | Mire kell?                                           |
|------------|------------------------------------------------------|
| `nasm`     | NASM assembler – az `.asm` fájlt `.o` object fájllá fordítja |
| `binutils` | GNU linker (`ld`), `objdump`, `nm` és más eszközök  |
| `gdb`      | GNU debugger – assembly szintű debuggoláshoz         |
| `strace`   | Syscall-ok nyomon követése futás közben (opcionális) |

### Verzió ellenőrzése

```bash
nasm --version
# nasm version 2.15.05 compiled on ...

ld --version
# GNU ld (GNU Binutils for Ubuntu) 2.38

gdb --version
# GNU gdb (Ubuntu 12.1-...) ...
```

> **💡 Tipp:** Ha `nasm --version` parancsra `command not found` üzenetet kapsz, a telepítés sikertelen volt. Ellenőrizd az `apt` kimenetét hibák után.

---

## 2. NASM szintaxis alapjai

### Intel szintaxis vs AT&T szintaxis

Az x86 assembly-nek két fő "tájszólása" létezik:

| Tulajdonság        | Intel szintaxis (NASM)            | AT&T szintaxis (GAS/gcc)          |
|--------------------|-----------------------------------|-----------------------------------|
| Assembler          | NASM, MASM, FASM                  | GAS (`as`), gcc alapértelmezett   |
| Operandus sorrend  | `mov dst, src` (cél, forrás)      | `movq src, dst` (forrás, cél!)    |
| Memória hivatkozás | `[rax]`                           | `(%rax)`                          |
| Regiszter prefix   | nincs (`rax`)                     | `%` jel (`%rax`)                  |
| Immediáte prefix   | nincs (`42`)                      | `$` jel (`$42`)                   |
| Utasítás suffix    | nincs (`mov`)                     | méret jelöli (`movq`, `movl`)     |

**Mi az Intel szintaxist (NASM) használjuk** – ez olvashatóbb, és a dokumentációk többsége (Intel manuálok) is ezt használja.

> **💡 Tipp:** Java analógia: ez olyan, mintha a Java-t kétféleképpen lehetne írni – az egyik `x = y + z;`, a másik `z y + x =;`. Ugyanaz a gépi kód keletkezik, csak a forrás néz ki másképp.

### Utasítás felépítése

Minden NASM sor a következő struktúrát követi:

```
[label:]    utasítás    [operandus1, operandus2]    [; komment]
```

Példák:

```nasm
start:      mov     rax, 60         ; az rax regiszterbe töltjük a 60-as számot
            syscall                 ; kernelhívás végrehajtása
loop_end:                           ; csak egy label, utasítás nélkül
```

- **Label (címke):** opcionális, kettősponttal zárul. Szimbolikus név egy memóriacímre.
- **Utasítás:** a tényleges CPU-utasítás (`mov`, `add`, `syscall`, stb.)
- **Operandusok:** az utasítás argumentumai (0, 1 vagy 2 db)
- **Komment:** pontosvessző után, a fordító figyelmen kívül hagyja

### Szekciók (.text, .data, .bss)

Egy assembly program logikailag szekciókra van osztva. Ezek az ELF fájlban különböző szegmensekké válnak, különböző jogosultságokkal:

```
+--------------------+----------------------------------------+
| Szekció            | Tartalom                               |
+--------------------+----------------------------------------+
| .text              | Futtatható kód (read-only + exec)      |
| .data              | Inicializált adatok (read-write)       |
| .bss               | Inicializálatlan adatok (read-write)   |
+--------------------+----------------------------------------+
```

**Java analógia:**
- `.text` ≈ a bytekód a `.class` fájlban (a JVM futtatja)
- `.data` ≈ a konstans pool (`static final` értékek)
- `.bss` ≈ statikus változók, amelyek nullára inicializálódnak JVM induláskor

```nasm
section .data
    message db "Hello, World!", 10   ; inicializált adat: string literal

section .bss
    buffer resb 64                   ; 64 bájt fenntartva, de nem inicializálva

section .text
    global _start                    ; az _start szimbólum legyen publikus (linker igényli)

_start:
    ; ... kód ide kerül
```

#### .bss adattípusok

A `.bss` szekcióban `res*` direktívákkal foglalunk helyet:

| Direktíva | Méret       | C megfelelő    |
|-----------|-------------|----------------|
| `resb N`  | N × 1 bájt  | `char buf[N]`  |
| `resw N`  | N × 2 bájt  | `short buf[N]` |
| `resd N`  | N × 4 bájt  | `int buf[N]`   |
| `resq N`  | N × 8 bájt  | `long buf[N]`  |

#### .data adattípusok

A `.data` szekcióban `d*` direktívákkal definiálunk inicializált adatokat:

| Direktíva | Méret       | Példa                          |
|-----------|-------------|--------------------------------|
| `db`      | 1 bájt      | `db 0x41` (= 'A')             |
| `dw`      | 2 bájt      | `dw 1000`                     |
| `dd`      | 4 bájt      | `dd 0xDEADBEEF`               |
| `dq`      | 8 bájt      | `dq 1234567890123`            |
| `db`      | string      | `db "Hello", 10, 0`           |

### A `global _start` direktíva

```nasm
global _start
```

Ez a sor azt mondja a linkernek: az `_start` szimbólum legyen publikusan látható az object fájlból. A `ld` linker alapértelmezetten az `_start` szimbólumot keresi belépési pontként.

> **⚠️ Figyelem:** Ha kihagyod a `global _start` sort, a linker ezt a hibát dobja: `ld: warning: cannot find entry symbol _start`. A program esetleg futhat, de a viselkedése meghatározatlan.

**Java analógia:** A `global _start` megfelel annak, mintha a `public static void main(String[] args)` metódust kell megjelölni – ez az a pont, ahol a program végrehajtása kezdődik.

---

## 3. Az első program: kilépés, de szépen

Mielőtt "Hello, World!"-öt írnánk, nézzük meg a legegyszerűbb lehetséges programot: az, amelyik azonnal kilép.

### exit.asm

```nasm
; exit.asm – A lehető legegyszerűbb x86-64 assembly program
; Csinál semmit, de szabályosan kilép.

section .text
    global _start

_start:
    mov     rax, 60         ; syscall szám: exit (60)
    mov     rdi, 0          ; exit kód: 0 (siker)
    syscall                 ; kernelhívás: kilépés
```

### Soronkénti magyarázat

| Sor | Magyarázat |
|-----|------------|
| `section .text` | Megkezdjük a kód szekciót |
| `global _start` | Az `_start` szimbólum legyen látható a linker számára |
| `_start:` | Label: ide mutat a program belépési pontja |
| `mov rax, 60` | Az `rax` regiszterbe töltjük a 60-as számot (ez az `exit` syscall száma Linux x86-64-en) |
| `mov rdi, 0` | Az `rdi` regiszterbe töltjük a 0-t (ez lesz az exit kód, a program visszatérési értéke) |
| `syscall` | Kernel megszakítás: átadjuk a vezérlést a Linux kernelnek. A kernel az `rax` értéke alapján tudja, melyik syscall-t kell végrehajtani |

### Fordítás és futtatás

```bash
# 1. Assembler: .asm → .o (object fájl)
nasm -f elf64 -g -F dwarf exit.asm -o exit.o

# 2. Linker: .o → futtatható
ld exit.o -o exit

# 3. Futtatás
./exit

# 4. Exit kód ellenőrzése (mindig azonnal az exit után!)
echo $?
# Kimenet: 0
```

> **💡 Tipp:** Az `echo $?` a legutoljára futtatott parancs visszatérési értékét írja ki. Shell konvenció: 0 = siker, 1-255 = hiba vagy egyéb jelzés.

### Kísérletezés: különböző exit kódok

Módosítsd a `mov rdi, 0` sort különböző értékekre és ellenőrizd:

```bash
# exit kód = 42
mov rdi, 42

# Fordítás és futtatás után:
echo $?
# Kimenet: 42
```

> **⚠️ Figyelem:** Az exit kód értéke 0-255 között van (8 bites értékként kezeli a kernel). Ha 256-ot adsz meg, az `echo $?` kimenete 0 lesz (moduláris aritmetika: 256 mod 256 = 0).

---

## 4. Hello, World!

### hello.asm – teljes kód

```nasm
; hello.asm – "Hello, World!" kiírása Linux x86-64 assembly-ben

section .data
    message db "Hello, World!", 10  ; string + newline (10 = '\n' ASCII kódja)
    len     equ $ - message         ; a string hossza bájtban

section .text
    global _start

_start:
    ; --- write syscall ---
    mov     rax, 1          ; syscall szám: write (1)
    mov     rdi, 1          ; fájl descriptor: 1 = stdout
    mov     rsi, message    ; pointer: hol kezdődik a string a memóriában
    mov     rdx, len        ; hány bájtot írjunk ki
    syscall                 ; kernelhívás: kiírás

    ; --- exit syscall ---
    mov     rax, 60         ; syscall szám: exit (60)
    mov     rdi, 0          ; exit kód: 0 (siker)
    syscall                 ; kernelhívás: kilépés
```

### Soronkénti magyarázat

#### A .data szekció

```nasm
message db "Hello, World!", 10
```

- `message` – label: szimbolikus neve ennek a memóriacímnek
- `db` – "define byte" direktíva: bájtsorozatot definiál
- `"Hello, World!"` – a karakterek ASCII kódjai egymás után a memóriában
- `10` – az újsor karakter (`\n`) ASCII kódja, vesszővel elválasztva a stringtől

A memóriában ez így néz ki (hexadecimálisan):
```
48 65 6C 6C 6F 2C 20 57 6F 72 6C 64 21 0A
H  e  l  l  o  ,     W  o  r  l  d  !  \n
```

```nasm
len equ $ - message
```

- `len` – szimbolikus konstans neve
- `equ` – "equate": konstanst definiál (nem foglal memóriát, csak egy szám)
- `$` – speciális NASM szimbólum: az **aktuális pozíció** (current position counter) értéke, azaz a `$` utáni bájt címe
- `$ - message` – a különbség: az aktuális pozíció mínusz a `message` label címe = a string hossza bájtban

> **💡 Tipp:** `equ` direktíva Java analógiája: `static final int LEN = 14;` – fordítási időben ismert konstans. A különbség az, hogy itt a fordító automatikusan kiszámolja a hosszt a string definiálása alapján.

> **⚠️ Figyelem:** Ne feledd a `10`-et (újsor) a string végén! Nélküle a "Hello, World!" szöveg nem kap saját sort, hanem összeolvad a shell prompttal.

#### A write syscall

Linux x86-64-en a `write` rendszerhívás szignaturája C-ben:
```c
ssize_t write(int fd, const void *buf, size_t count);
```

Assembly-ben a syscall konvenció szerint:

| Regiszter | Szerepe              | Értékünk     |
|-----------|----------------------|--------------|
| `rax`     | Syscall szám         | 1 (write)    |
| `rdi`     | 1. argumentum: `fd`  | 1 (stdout)   |
| `rsi`     | 2. argumentum: `buf` | `message` (pointer) |
| `rdx`     | 3. argumentum: `count` | `len` (14) |

**Java analógia:**
```java
// Java:
System.out.write("Hello, World!\n".getBytes());

// Assembly syscall szinten ugyanez történik:
// write(1, &message, 14)
// A JVM is ezt hívja a háttérben (JNI → libc → syscall)
```

#### A két syscall sorrendje

Az utasítások **szekvenciálisan** hajtódnak végre, fentről lefelé:
1. Először a `write` syscall fut le → kiírja a szöveget
2. Utána az `exit` syscall fut le → kilép a program

### Fordítás és futtatás

```bash
# Assembler
nasm -f elf64 -g -F dwarf hello.asm -o hello.o

# Linker
ld hello.o -o hello

# Futtatás
./hello
# Kimenet: Hello, World!

# Exit kód
echo $?
# Kimenet: 0
```

### Ellenőrzés strace-szel

A `strace` megmutatja, milyen syscall-okat hajt végre a program futás közben:

```bash
strace ./hello
```

Várható kimenet (rövidítve):

```
execve("./hello", ["./hello"], ...) = 0
write(1, "Hello, World!\n", 14)     = 14
exit(0)                              = ?
+++ exited with 0 +++
```

> **💡 Tipp:** Figyeld meg: pontosan két syscall hívódik (a kernel által észlelt `execve` az OS-től jön). A `write` visszatérési értéke 14 – ennyi bájtot írt ki sikeresen. Ez megegyezik a `len` értékével.

---

## 5. A fordítási folyamat részletesen

### A három fázis

```
hello.asm
    │
    │  nasm -f elf64 -g -F dwarf hello.asm -o hello.o
    ▼
hello.o   (ELF relocatable object file)
    │
    │  ld hello.o -o hello
    ▼
hello     (ELF executable)
    │
    │  ./hello
    ▼
Futó folyamat (process)
```

### Mi az ELF formátum?

Az **ELF** (Executable and Linkable Format) a Linux futtatható fájlok szabványos formátuma. Tartalmaz:
- **ELF header:** fájltípus, architektúra, belépési pont címe
- **Program headers:** hogyan kell betölteni a memóriába (szegmensek)
- **Section headers:** szekciók metaadatai (`.text`, `.data`, `.bss`, stb.)
- **Szimbolumtábla:** label-ek neve és címe (debugging-hoz)

```bash
# ELF header megtekintése
readelf -h hello

# Szekciók listája
readelf -S hello

# Szimbólumtábla
readelf -s hello
```

### A NASM parancs részletezve

```bash
nasm -f elf64 -g -F dwarf hello.asm -o hello.o
```

| Kapcsoló     | Jelentés                                                        |
|--------------|-----------------------------------------------------------------|
| `-f elf64`   | Output formátum: 64-bites ELF object fájl                      |
| `-g`         | Debug információ generálása (szükséges GDB-hez)                 |
| `-F dwarf`   | Debug formátum: DWARF (a GDB ezt érti)                         |
| `hello.asm`  | Forrásfájl neve                                                 |
| `-o hello.o` | Output fájl neve (konvenció: `.o` kiterjesztés)                 |

### A linker (ld) feladata

Az `ld` linker:
1. Beolvassa a `.o` object fájl(oka)t
2. Feloldja a szimbolikus hivatkozásokat (pl. `message` → tényleges memóriacím)
3. Összerakja a végleges ELF futtatható fájlt
4. Beállítja a belépési pontot (`_start` címe)

```bash
ld hello.o -o hello
```

> **⚠️ Figyelem:** Ha C standard library-t használnál (`printf`, `malloc` stb.), akkor `ld` helyett `gcc` linkelést kell használni: `gcc -nostartfiles hello.o -o hello`. Mi most pure assembly-t írunk, ezért elegendő a sima `ld`.

### Gépi kód megtekintése: objdump

```bash
objdump -d hello
```

Várható kimenet:

```
hello:     file format elf64-x86-64

Disassembly of section .text:

0000000000401000 <_start>:
  401000:  b8 01 00 00 00    mov    eax,0x1
  401005:  bf 01 00 00 00    mov    edi,0x1
  40100a:  48 be 00 20 40 00 00 00 00 00   movabs rsi,0x402000
  401014:  ba 0e 00 00 00    mov    edx,0xe
  401019:  0f 05             syscall
  40101b:  b8 3c 00 00 00    mov    eax,0x3c
  401020:  bf 00 00 00 00    mov    edi,0x0
  401025:  0f 05             syscall
```

> **💡 Tipp:** Figyeld meg a `0f 05` bájtokat – ez a `syscall` utasítás gépi kódja. A `b8 01 00 00 00` a `mov eax, 1` kódja (little-endian: az 1-es szám bájtjai fordított sorrendben).

### Makefile az assembly projektekhez

Egy egyszerű `Makefile` megkíméli a gépelési hibáktól:

```makefile
# Makefile – x86-64 assembly projekthez

ASM    = nasm
ASMFLAGS = -f elf64 -g -F dwarf
LD     = ld

# Alapértelmezett cél: minden program lefordítása
all: hello exit

# Általános szabály: .asm → .o
%.o: %.asm
	$(ASM) $(ASMFLAGS) $< -o $@

# Általános szabály: .o → futtatható
%: %.o
	$(LD) $< -o $@

# Takarítás
clean:
	rm -f *.o hello exit

.PHONY: all clean
```

Használat:

```bash
make          # Lefordít mindent
make hello    # Csak a hello programot fordítja
make clean    # Törli a generált fájlokat
./hello
```

> **⚠️ Figyelem:** A Makefile-ban az egyes szabályok **TAB karakterrel** kezdődnek, nem szóközökkel! Ez a make szintaxis szigorú követelménye. Ha szóközt használsz, hibaüzenetet kapsz: `Makefile:10: *** missing separator. Stop.`

---

## 6. Második program: két szöveg kiírása

Ez a program bemutatja, hogy az utasítások **sorban, egymás után** hajtódnak végre:

### two_lines.asm

```nasm
; two_lines.asm – Két különböző szöveg kiírása egymás után

section .data
    msg1    db "Első sor", 10           ; 10 = newline
    len1    equ $ - msg1

    msg2    db "Második sor", 10
    len2    equ $ - msg2

section .text
    global _start

_start:
    ; Első szöveg kiírása
    mov     rax, 1          ; write syscall
    mov     rdi, 1          ; stdout
    mov     rsi, msg1       ; pointer az első stringre
    mov     rdx, len1       ; az első string hossza
    syscall

    ; Második szöveg kiírása
    mov     rax, 1          ; write syscall (újra be kell állítani!)
    mov     rdi, 1          ; stdout
    mov     rsi, msg2       ; pointer a második stringre
    mov     rdx, len2       ; a második string hossza
    syscall

    ; Kilépés
    mov     rax, 60         ; exit syscall
    mov     rdi, 0          ; exit kód: 0
    syscall
```

### Fordítás és futtatás

```bash
nasm -f elf64 -g -F dwarf two_lines.asm -o two_lines.o
ld two_lines.o -o two_lines
./two_lines
```

Kimenet:
```
Első sor
Második sor
```

> **💡 Tipp:** Figyeld meg, hogy a második `write` syscall előtt újra be kell állítani az `rax`-t `1`-re! Az első `syscall` utasítás után az `rax` értéke megváltozhat (a kernel a visszatérési értéket ide írja). Assembly-ben nem létezik automatikus állapot-megőrzés a syscall-ok között – minden regiszter beállítása a te felelősséged.

> **⚠️ Figyelem:** Ha UTF-8 karaktereket (ékezetes betűket) használsz a stringekben, a bájthossz eltér a karakterhossztól! Magyar ékezetes betűk UTF-8-ban 2 bájtot foglalnak. Az `equ $ - msg1` trükk a bájtszámot számítja, nem a karakterszámot – de ez pontosan az, amire a `write` syscallnak szüksége van.

---

## 7. Gyakorlatok

### 1. feladat: Goodbye, World!

Írj programot, amely kiírja: `Goodbye, World!`

**Lépések:**
1. Másold a `hello.asm`-et `goodbye.asm` névre
2. Módosítsd a stringet
3. Frissítsd a `len` számítást (automatikusan helyes lesz az `equ $ - message` trükk miatt!)
4. Fordítsd le és futtasd

### 2. feladat: Exit kód kísérlet

Módosítsd az `exit.asm` programot:
- Exit kód legyen 42
- Ellenőrizd: `echo $?`
- Mi történik, ha 256-ot adsz meg? Mi az `echo $?` kimenete?
- Mi történik 255-tel? 1-gyel?

### 3. feladat: Saját neved kiírása

Írj programot, amely a nevedet írja ki! Ha például a neved "Kovács János", a kimenet legyen:
```
Kovács János
```

> **⚠️ Figyelem:** Ha ékezetes betűket használsz, a fájlt UTF-8 kódolásban mentsd el. A string hossza bájtokban lesz mérve, de az `equ $ - label` ezt automatikusan kezeli.

### 4. feladat: objdump vizsgálat

Fordítsd le a `hello.asm` programot és vizsgáld meg a gépi kódot:

```bash
objdump -d hello
```

- Hány bájt a `syscall` utasítás gépi kódja?
- Mi a `mov rax, 1` gépi kódja hexadecimálisan?
- Melyik memóriacímen kezdődik az `_start`?

### 5. feladat: strace elemzés

Futtasd a `two_lines` programot strace-szel:

```bash
strace ./two_lines
```

- Hány `write` syscall hívódik meg?
- Mit mutat a `write` syscall visszatérési értéke?
- Mi az `exit` syscall visszatérési értéke? Miért?

---

### Megoldás vázlat

<details>
<summary>1. feladat – Goodbye, World! (kattints a megoldásért)</summary>

```nasm
section .data
    message db "Goodbye, World!", 10
    len     equ $ - message

section .text
    global _start

_start:
    mov     rax, 1
    mov     rdi, 1
    mov     rsi, message
    mov     rdx, len
    syscall

    mov     rax, 60
    mov     rdi, 0
    syscall
```

```bash
nasm -f elf64 -g -F dwarf goodbye.asm -o goodbye.o
ld goodbye.o -o goodbye
./goodbye
```

</details>

<details>
<summary>2. feladat – Exit kód (kattints a megoldásért)</summary>

```nasm
section .text
    global _start

_start:
    mov     rax, 60
    mov     rdi, 42         ; ← ezt változtasd
    syscall
```

- 256 → `echo $?` kimenete: **0** (256 mod 256 = 0)
- 255 → `echo $?` kimenete: **255**
- 257 → `echo $?` kimenete: **1**

</details>

---

## 8. Ellenőrizd magad

1. **Mi a különbség az Intel és az AT&T szintaxis között?** Hozz fel legalább két konkrét példát (operandus sorrend, memória hivatkozás).

2. **Mit jelent a `global _start` direktíva?** Mi történik, ha kihagyod?

3. **Mit számít ki a `len equ $ - message` kifejezés?** Miért hasznos ez a trükk, és mi a `$` szimbólum jelentése NASM-ban?

4. **Sorold fel a `write` syscall argumentumait x86-64 Linuxon!** Melyik regiszterbe kerül a fájl descriptor, a buffer pointere és a bájthossz?

5. **Mi a különbség a `.data` és a `.bss` szekció között?** Melyikben használsz `db`-t, és melyikben `resb`-t?

6. **Mi történik, ha az `exit` syscall előtt elfelejtesz kiírni?** Miért nem jelenik meg semmi a képernyőn – mit kellene módosítani?

---

## Összefoglalás

Ma elvégeztük az assembly fejlesztés leglényegesebb alapját:

| Fogalom | Összefoglalás |
|---------|--------------|
| NASM szintaxis | Intel szintaxis: `mov cél, forrás`, szekciók, label-ek |
| .text / .data / .bss | Kód, inicializált adat, nem inicializált adat szekciók |
| `global _start` | A linker belépési pontja; kötelező |
| `db`, `equ`, `$` | Adatdefiníció, konstans, aktuális pozíció |
| write syscall (1) | `rax=1, rdi=fd, rsi=ptr, rdx=len` |
| exit syscall (60) | `rax=60, rdi=exit_code` |
| Fordítás | `nasm -f elf64` → `ld` → futtatható ELF |
| objdump / strace | Gépi kód és syscall-ok vizsgálata |

**Holnap (3. nap):** Adatmozgatás és aritmetika – `mov`, `add`, `sub`, `mul`, `div`, flag-ek és az aritmetikai műveletek mélyebb megértése.
