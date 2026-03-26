# x86-64 Assembly Tananyag

> **10 napos tanfolyam senior Java/DevOps mérnököknek, nulláról indulva**

Ez a tananyag x86-64 Assembly programozást tanít NASM assemblerrel, Linux környezetben. A célközönség tapasztalt szoftverfejlesztő, aki ismeri a magas szintű nyelveket (Java, Kotlin), de még nem találkozott Assembly-vel. A fejezetek Java párhuzamokkal, gyakorlatokkal és teljes, futtatható kódpéldákkal dolgoznak.

---

## Előfeltételek

- Linux (Ubuntu/Debian ajánlott, WSL2 is működik)
- Alapvető terminál/shell ismeretek
- Telepítendő csomagok:

```bash
sudo apt update
sudo apt install nasm binutils gdb strace
```

---

## Tananyag felépítése

| Nap | Téma | Fájl |
|-----|-------|------|
| **1** | Alapok – CPU architektúra, regiszterek, számrendszerek | [`nap-01/README.md`](nap-01/README.md) |
| **2** | Első program – NASM szintaxis, Hello World, fordítás | [`nap-02/README.md`](nap-02/README.md) |
| **3** | Adatmozgatás és aritmetika – MOV, ADD, MUL, flag-ek | [`nap-03/README.md`](nap-03/README.md) |
| **4** | Vezérlési szerkezetek – ugró utasítások, ciklusok, CMOVcc | [`nap-04/README.md`](nap-04/README.md) |
| **5** | Stack és függvények – push/pop, call/ret, calling convention | [`nap-05/README.md`](nap-05/README.md) |
| **6** | Memóriakezelés – címzési módok, LEA, mutatók, endianness | [`nap-06/README.md`](nap-06/README.md) |
| **7** | Syscall-ok – I/O, fájlkezelés, hibakezelés | [`nap-07/README.md`](nap-07/README.md) |
| **8** | Stringkezelés és adatstruktúrák – REP, struktúrák, tömbök | [`nap-08/README.md`](nap-08/README.md) |
| **9** | Debugging és optimalizálás – GDB, strace, teljesítmény | [`nap-09/README.md`](nap-09/README.md) |
| **10** | Záró projekt és továbblépés – C interop, szám→szöveg konverter | [`nap-10/README.md`](nap-10/README.md) |

## Mellékletek

| Dokumentum | Fájl |
|-----------|------|
| Utasítás referencia (cheat sheet) | [`mellekletek/utasitas-referencia.md`](mellekletek/utasitas-referencia.md) |
| Linux syscall referencia | [`mellekletek/syscall-referencia.md`](mellekletek/syscall-referencia.md) |

---

## Hogyan használd

1. **Napi ~1 óra** – olvasd el a fejezetet, próbáld ki a kódpéldákat
2. **Gépelj, ne másolj** – az Assembly-t gépelve tanulod meg igazán
3. **Kísérletezz** – módosítsd a példákat, nézd meg mi történik
4. **Használd a GDB-t** – a 9. naptól, de már korábban is érdemes kipróbálni
5. **Olddd meg a gyakorlatokat** – a fejezetek végén találsz feladatokat megoldásokkal

## Gyors fordítási sablon

```bash
# Fordítás (debug infóval)
nasm -f elf64 -g -F dwarf program.asm -o program.o

# Linkelés
ld program.o -o program

# Futtatás
./program

# Visszatérési érték ellenőrzése
echo $?

# Syscall nyomkövetés
strace ./program
```

## Makefile sablon

```makefile
ASM = nasm
ASMFLAGS = -f elf64 -g -F dwarf
LD = ld

%: %.asm
	$(ASM) $(ASMFLAGS) $< -o $*.o
	$(LD) $*.o -o $@
	rm -f $*.o

clean:
	rm -f *.o
```

---

## Hasznos linkek

- [NASM dokumentáció](https://www.nasm.us/doc/)
- [NASM Tutorial (LMU)](https://cs.lmu.edu/~ray/notes/nasmtutorial/)
- [x86-64 regiszter referencia (UAF)](https://www.cs.uaf.edu/2017/fall/cs301/reference/x86_64.html)
- [Linux syscall táblázat](https://filippo.io/linux-syscall-table/)
- [x64 Cheat Sheet (Brown CS)](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
- [System V AMD64 ABI specifikáció](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
- [Compiler Explorer (Godbolt)](https://godbolt.org/) – C kód Assembly kimenetének vizsgálata
- [x64 Roadmap (GitHub)](https://github.com/yds12/x64-roadmap) – további gyakorlatok

---

*Készült 2026 márciusában. Architektúra: x86-64 | Assembler: NASM | OS: Linux*
