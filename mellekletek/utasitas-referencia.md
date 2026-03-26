# Utasítás Referencia – x86-64 Assembly (NASM)

Ez a dokumentum a tananyagban szereplő összes utasítást összefoglalja csoportosított táblázatokban.  
**Konvenciók:** `r` = regiszter, `m` = memória, `imm` = közvetlen érték, `r/m` = regiszter vagy memória.

---

## 1. Adatmozgatás (Data Transfer)

| Utasítás | Szintaxis | Leírás | Flag-hatás | Példa |
|----------|-----------|--------|------------|-------|
| `mov` | `mov r/m, r/m/imm` | Értékét másolja a forrásból a célba | Nincs | `mov rax, 42` |
| `movzx` | `movzx r, r/m` | Másolás nullával kiterjesztve (kisebb → nagyobb) | Nincs | `movzx rax, bl` |
| `movsx` | `movsx r, r/m` | Másolás előjellel kiterjesztve | Nincs | `movsx rax, ebx` |
| `movsxd` | `movsxd r64, r/m32` | 32-bit → 64-bit előjeles kiterjesztés | Nincs | `movsxd rax, ecx` |
| `lea` | `lea r, [mem_expr]` | Cím számítása (nem olvassa a memóriát!) | Nincs | `lea rax, [rbx+rcx*4+8]` |
| `xchg` | `xchg r/m, r/m` | Két operandus értékének felcserélése | Nincs | `xchg rax, rbx` |
| `push` | `push r/m/imm` | RSP -= 8, [RSP] ← érték | Nincs | `push rax` |
| `pop` | `pop r/m` | érték ← [RSP], RSP += 8 | Nincs | `pop rbx` |
| `pushfq` | `pushfq` | RFLAGS regiszter stack-re | Nincs | `pushfq` |
| `popfq` | `popfq` | Stack-ről RFLAGS visszaállítása | Összes | `popfq` |
| `cbw` | `cbw` | AL → AX előjeles kiterjesztés | Nincs | `cbw` |
| `cwde` | `cwde` | AX → EAX előjeles kiterjesztés | Nincs | `cwde` |
| `cdqe` | `cdqe` | EAX → RAX előjeles kiterjesztés | Nincs | `cdqe` |
| `cqo` | `cqo` | RAX → RDX:RAX előjeles kiterjesztés | Nincs | `cqo` |

---

## 2. Aritmetika (Arithmetic)

| Utasítás | Szintaxis | Leírás | Flag-hatás | Példa |
|----------|-----------|--------|------------|-------|
| `add` | `add r/m, r/m/imm` | Összeadás | CF, OF, SF, ZF, PF, AF | `add rax, rbx` |
| `sub` | `sub r/m, r/m/imm` | Kivonás | CF, OF, SF, ZF, PF, AF | `sub rax, 5` |
| `adc` | `adc r/m, r/m/imm` | Összeadás carry-vel (CF is hozzáadva) | CF, OF, SF, ZF, PF, AF | `adc rdx, 0` |
| `sbb` | `sbb r/m, r/m/imm` | Kivonás borrowal (CF is levonva) | CF, OF, SF, ZF, PF, AF | `sbb rax, rbx` |
| `inc` | `inc r/m` | Növelés 1-gyel (CF nem változik!) | OF, SF, ZF, PF, AF | `inc rcx` |
| `dec` | `dec r/m` | Csökkentés 1-gyel (CF nem változik!) | OF, SF, ZF, PF, AF | `dec rcx` |
| `neg` | `neg r/m` | Kettes komplemens negálás (0 - operandus) | CF, OF, SF, ZF, PF, AF | `neg rax` |
| `mul` | `mul r/m` | Előjel nélküli szorzás: RDX:RAX ← RAX × r/m | CF, OF (többi undef.) | `mul rbx` |
| `imul` | `imul r/m` | Előjeles szorzás: RDX:RAX ← RAX × r/m | CF, OF | `imul rbx` |
| `imul` | `imul r, r/m` | Kétoperandusú: r ← r × r/m | CF, OF | `imul rax, rbx` |
| `imul` | `imul r, r/m, imm` | Háromoperandusú: r ← r/m × imm | CF, OF | `imul rax, rbx, 5` |
| `div` | `div r/m` | Előjel nélküli osztás: RAX ← RDX:RAX / r/m, RDX ← maradék | Undef. | `div rcx` |
| `idiv` | `idiv r/m` | Előjeles osztás: RAX ← RDX:RAX / r/m, RDX ← maradék | Undef. | `idiv rcx` |

> **⚠️ Figyelem:** `div` és `idiv` előtt mindig állítsd be RDX-et: `xor rdx, rdx` (előjel nélkülinél) vagy `cqo` (előjeles, 64-bit esetén).

---

## 3. Logikai műveletek (Logical)

| Utasítás | Szintaxis | Leírás | Flag-hatás | Példa |
|----------|-----------|--------|------------|-------|
| `and` | `and r/m, r/m/imm` | Bitenkénti logikai ÉS | CF=0, OF=0, SF, ZF, PF | `and rax, 0xFF` |
| `or` | `or r/m, r/m/imm` | Bitenkénti logikai VAGY | CF=0, OF=0, SF, ZF, PF | `or rax, rbx` |
| `xor` | `xor r/m, r/m/imm` | Bitenkénti kizáró VAGY | CF=0, OF=0, SF, ZF, PF | `xor rax, rax` |
| `not` | `not r/m` | Bitenkénti komplemens (nem állít flag-et!) | Nincs | `not rax` |
| `test` | `test r/m, r/m/imm` | AND flag-ekre, eredmény eldobva | CF=0, OF=0, SF, ZF, PF | `test rax, rax` |
| `cmp` | `cmp r/m, r/m/imm` | SUB flag-ekre, eredmény eldobva | CF, OF, SF, ZF, PF, AF | `cmp rax, 5` |

### Bitléptető utasítások

| Utasítás | Szintaxis | Leírás | Flag-hatás | Példa |
|----------|-----------|--------|------------|-------|
| `shl` | `shl r/m, imm/CL` | Bal logikai léptetés (× 2^n) | CF, OF, ZF, SF | `shl rax, 3` |
| `shr` | `shr r/m, imm/CL` | Jobb logikai léptetés (÷ 2^n, előjel nélküli) | CF, OF, ZF, SF | `shr rax, 1` |
| `sar` | `sar r/m, imm/CL` | Jobb aritmetikai léptetés (előjel megőrző) | CF, OF, ZF, SF | `sar rax, 2` |
| `rol` | `rol r/m, imm/CL` | Balra forgatás (rotate left) | CF, OF | `rol al, 1` |
| `ror` | `ror r/m, imm/CL` | Jobbra forgatás | CF, OF | `ror al, 4` |
| `rcl` | `rcl r/m, imm/CL` | Balra forgatás CF-en át | CF, OF | `rcl rax, 1` |
| `rcr` | `rcr r/m, imm/CL` | Jobbra forgatás CF-en át | CF, OF | `rcr rax, 1` |
| `bsf` | `bsf r, r/m` | Bit Scan Forward: legalacsonyabb 1-es bit indexe | ZF | `bsf rax, rbx` |
| `bsr` | `bsr r, r/m` | Bit Scan Reverse: legmagasabb 1-es bit indexe | ZF | `bsr rax, rbx` |
| `bt` | `bt r/m, r/imm` | Bit Test: az N-edik bit értéke CF-be | CF | `bt rax, 3` |
| `bts` | `bts r/m, r/imm` | Bit Test and Set | CF | `bts rax, 5` |
| `btr` | `btr r/m, r/imm` | Bit Test and Reset | CF | `btr rax, 5` |

---

## 4. Ugró utasítások (Control Flow)

### Feltétel nélküli ugró

| Utasítás | Szintaxis | Leírás | Példa |
|----------|-----------|--------|-------|
| `jmp` | `jmp label/r/m` | Feltétel nélküli ugrás | `jmp .loop` |
| `call` | `call label/r/m` | Hívás: ret. cím stack-re, majd ugrás | `call my_func` |
| `ret` | `ret` | Visszatérés: RIP ← [RSP], RSP += 8 | `ret` |
| `ret N` | `ret imm16` | Visszatérés + RSP += N | `ret 8` |
| `loop` | `loop label` | RCX--; ha RCX ≠ 0: ugrás | `loop .body` |
| `loope` | `loope label` | RCX--; ha RCX ≠ 0 és ZF=1: ugrás | `loope .eq_loop` |
| `loopne` | `loopne label` | RCX--; ha RCX ≠ 0 és ZF=0: ugrás | `loopne .ne_loop` |

### Feltételes ugrók

#### Előjeles összehasonlítás utáni (cmp a, b → a ?? b)

| Utasítás | Alias | Feltétel | Leírás |
|----------|-------|----------|--------|
| `je` | `jz` | ZF=1 | Egyenlő / nulla |
| `jne` | `jnz` | ZF=0 | Nem egyenlő / nem nulla |
| `jl` | `jnge` | SF ≠ OF | Kisebb (előjeles) |
| `jle` | `jng` | ZF=1 vagy SF≠OF | Kisebb-egyenlő (előjeles) |
| `jg` | `jnle` | ZF=0 és SF=OF | Nagyobb (előjeles) |
| `jge` | `jnl` | SF=OF | Nagyobb-egyenlő (előjeles) |

#### Előjel nélküli összehasonlítás utáni

| Utasítás | Alias | Feltétel | Leírás |
|----------|-------|----------|--------|
| `jb` | `jnae`, `jc` | CF=1 | Kisebb (előjel nélküli, below) |
| `jbe` | `jna` | CF=1 vagy ZF=1 | Kisebb-egyenlő (előjel nélküli) |
| `ja` | `jnbe` | CF=0 és ZF=0 | Nagyobb (előjel nélküli, above) |
| `jae` | `jnb`, `jnc` | CF=0 | Nagyobb-egyenlő (előjel nélküli) |

#### Flag alapú ugrók

| Utasítás | Feltétel | Leírás |
|----------|----------|--------|
| `js` | SF=1 | Negatív (sign flag set) |
| `jns` | SF=0 | Nem negatív |
| `jo` | OF=1 | Overflow |
| `jno` | OF=0 | Nincs overflow |
| `jp` / `jpe` | PF=1 | Páros paritás |
| `jnp` / `jpo` | PF=0 | Páratlan paritás |
| `jcxz` | CX=0 | CX nulla (16-bit) |
| `jecxz` | ECX=0 | ECX nulla (32-bit) |
| `jrcxz` | RCX=0 | RCX nulla (64-bit) |

### Feltételes mozgatás (CMOVcc)

| Utasítás | Feltétel | Leírás | Példa |
|----------|----------|--------|-------|
| `cmove` | ZF=1 | Mozgatás ha egyenlő | `cmove rax, rbx` |
| `cmovne` | ZF=0 | Mozgatás ha nem egyenlő | `cmovne rax, rbx` |
| `cmovl` | SF≠OF | Mozgatás ha kisebb | `cmovl rax, rbx` |
| `cmovg` | ZF=0, SF=OF | Mozgatás ha nagyobb | `cmovg rax, rbx` |
| `cmovs` | SF=1 | Mozgatás ha negatív | `cmovs rax, rbx` |

---

## 5. Stack és függvényhívás (Stack & Call)

| Utasítás | Szintaxis | Leírás | Flag-hatás | Példa |
|----------|-----------|--------|------------|-------|
| `push` | `push r/m/imm` | RSP -= 8; [RSP] ← érték | Nincs | `push rbp` |
| `pop` | `pop r/m` | érték ← [RSP]; RSP += 8 | Nincs | `pop rbp` |
| `call` | `call label/r/m` | push RIP; jmp cél | Nincs | `call printf` |
| `ret` | `ret [imm]` | pop RIP; [RSP += imm] | Nincs | `ret` |
| `enter` | `enter imm16, 0` | push rbp; mov rbp,rsp; sub rsp,imm | Nincs | `enter 16, 0` |
| `leave` | `leave` | mov rsp,rbp; pop rbp | Nincs | `leave` |
| `pushfq` | `pushfq` | Push RFLAGS | Nincs | `pushfq` |
| `popfq` | `popfq` | Pop RFLAGS | Összes | `popfq` |

---

## 6. String utasítások (String Operations)

### Alap string utasítások

| Utasítás | Forrás → Cél | Méret | Lépés (DF=0) | Leírás | Példa |
|----------|-------------|-------|--------------|--------|-------|
| `movsb` | [RSI] → [RDI] | 1 B | RSI++, RDI++ | Byte másolás | `movsb` |
| `movsw` | [RSI] → [RDI] | 2 B | RSI+=2, RDI+=2 | Word másolás | `movsw` |
| `movsd` | [RSI] → [RDI] | 4 B | RSI+=4, RDI+=4 | Dword másolás | `movsd` |
| `movsq` | [RSI] → [RDI] | 8 B | RSI+=8, RDI+=8 | Qword másolás | `movsq` |
| `stosb` | AL → [RDI] | 1 B | RDI++ | Byte tárolás | `stosb` |
| `stosw` | AX → [RDI] | 2 B | RDI+=2 | Word tárolás | `stosw` |
| `stosd` | EAX → [RDI] | 4 B | RDI+=4 | Dword tárolás | `stosd` |
| `stosq` | RAX → [RDI] | 8 B | RDI+=8 | Qword tárolás | `stosq` |
| `lodsb` | [RSI] → AL | 1 B | RSI++ | Byte betöltés | `lodsb` |
| `lodsw` | [RSI] → AX | 2 B | RSI+=2 | Word betöltés | `lodsw` |
| `lodsd` | [RSI] → EAX | 4 B | RSI+=4 | Dword betöltés | `lodsd` |
| `lodsq` | [RSI] → RAX | 8 B | RSI+=8 | Qword betöltés | `lodsq` |
| `scasb` | AL vs [RDI] | 1 B | RDI++ | Byte keresés (flag) | `scasb` |
| `scasw` | AX vs [RDI] | 2 B | RDI+=2 | Word keresés | `scasw` |
| `cmpsb` | [RSI] vs [RDI] | 1 B | RSI++, RDI++ | Byte összehasonlítás (flag) | `cmpsb` |
| `cmpsw` | [RSI] vs [RDI] | 2 B | RSI+=2, RDI+=2 | Word összehasonlítás | `cmpsw` |

### Direction Flag és REP prefix

| Utasítás | Leírás | Példa |
|----------|--------|-------|
| `cld` | DF ← 0 (előre: RSI/RDI növekszik) | `cld` |
| `std` | DF ← 1 (hátra: RSI/RDI csökken) | `std` |
| `rep X` | X végrehajtása RCX-szer (RCX--) | `rep movsb` |
| `repe X` | X amíg ZF=1 és RCX>0 | `repe cmpsb` |
| `repz X` | `repe` alias | `repz cmpsb` |
| `repne X` | X amíg ZF=0 és RCX>0 | `repne scasb` |
| `repnz X` | `repne` alias | `repnz scasb` |

---

## 7. Rendszer utasítások (System)

| Utasítás | Szintaxis | Leírás | Megjegyzés |
|----------|-----------|--------|------------|
| `syscall` | `syscall` | Linux kernel hívás (RAX = szám) | RAX, RCX, R11 módosul |
| `int 0x80` | `int 0x80` | Legacy 32-bit syscall interface | **Ne használd 64-bit programokban!** |
| `nop` | `nop` | Nem csinál semmit (1 ciklus) | Padding, timing |
| `hlt` | `hlt` | CPU megállítás (kernel mód) | Ring 0 szükséges |
| `cpuid` | `cpuid` | CPU info lekérdezés (EAX bemenet) | EAX,EBX,ECX,EDX módosul |
| `rdtsc` | `rdtsc` | Time Stamp Counter olvasás | EDX:EAX = TSC érték |
| `rdtscp` | `rdtscp` | RDTSC + processzor ID | EDX:EAX=TSC, ECX=ProcID |
| `xacquire` | prefix | Hardware Lock Elision hint | Tranzakcionális memória |
| `mfence` | `mfence` | Memória barrier (teljes) | Store/load nem sorrendcserélhető |
| `sfence` | `sfence` | Store fence | Store-ok sorrendben |
| `lfence` | `lfence` | Load fence | Load-ok sorrendben |

---

## 8. Speciális / Hasznos utasítások

| Utasítás | Szintaxis | Leírás | Példa |
|----------|-----------|--------|-------|
| `xor reg, reg` | `xor rax, rax` | Hatékony nullázás (gyorsabb mint `mov r,0`) | `xor eax, eax` |
| `lea` (aritmetika) | `lea rax, [rbx+rcx*2]` | Gyors összeadás/szorzás regisztereken | `lea rax, [rbx+rbx*2]` |
| `cdq` | `cdq` | EAX → EDX:EAX kiterjesztés (32-bit div előtt) | `cdq` |
| `cqo` | `cqo` | RAX → RDX:RAX kiterjesztés (64-bit div előtt) | `cqo` |
| `bswap` | `bswap r32/r64` | Byte-sorrend megfordítása (endian csere) | `bswap eax` |
| `popcnt` | `popcnt r, r/m` | 1-es bitek megszámlálása | `popcnt rax, rbx` |
| `lzcnt` | `lzcnt r, r/m` | Vezető nullák számlálása | `lzcnt rax, rbx` |
| `tzcnt` | `tzcnt r, r/m` | Záró nullák számlálása | `tzcnt rax, rbx` |
| `setcc` | `sete al` | Flag alapján AL = 0 vagy 1 | `sete al` |
| `stc` | `stc` | CF ← 1 | `stc` |
| `clc` | `clc` | CF ← 0 | `clc` |
| `cmc` | `cmc` | CF invertálása | `cmc` |
| `lahf` | `lahf` | AH ← FLAGS (alsó 8 bit) | `lahf` |
| `sahf` | `sahf` | FLAGS ← AH (alsó 8 bit) | `sahf` |

---

## 9. NASM direktívák

| Direktíva | Szintaxis | Leírás | Példa |
|-----------|-----------|--------|-------|
| `db` | `db expr,...` | Byte(ok) definiálása (.data) | `db 0x41, "Hello", 0` |
| `dw` | `dw expr,...` | Word (2 byte) definiálása | `dw 1000, 2000` |
| `dd` | `dd expr,...` | Dword (4 byte) | `dd 3.14, -1` |
| `dq` | `dq expr,...` | Qword (8 byte) | `dq 0xDEADBEEF` |
| `dt` | `dt expr` | 10 byte (extended precision float) | `dt 3.14159` |
| `resb` | `resb N` | N byte foglalása (.bss, nullázva) | `resb 64` |
| `resw` | `resw N` | N×2 byte | `resw 10` |
| `resd` | `resd N` | N×4 byte | `resd 10` |
| `resq` | `resq N` | N×8 byte | `resq 10` |
| `equ` | `name equ expr` | Névvel ellátott konstans | `SIZE equ 100` |
| `%define` | `%define name val` | Preprocesszor makró | `%define NULL 0` |
| `%macro` | `%macro name narg` | Többsoros makró definíció | `%macro PUSH_ALL 0` |
| `global` | `global sym` | Szimbólum exportálás (linker látja) | `global _start` |
| `extern` | `extern sym` | Külső szimbólum deklarálása | `extern printf` |
| `section` | `section .name` | Szekció váltás | `section .data` |
| `align` | `align N` | Igazítás N byte-ra | `align 16` |
| `times` | `times N db 0` | Utasítás/adat ismétlése | `times 64 db 0` |
| `struc` | `struc name` | Struktúra definíció kezdete | `struc Point` |
| `endstruc` | `endstruc` | Struktúra definíció vége | `endstruc` |
| `istruc` | `istruc name` | Struktúra példány kezdete | `istruc Point` |
| `iend` | `iend` | Struktúra példány vége | `iend` |
| `at` | `at field, db val` | Mező értéke struktúra példányban | `at Point.x, dd 10` |
| `bits` | `bits 16/32/64` | CPU mód beállítása | `bits 64` |
| `use64` | `use64` | 64-bites mód (BITS 64 alias) | `use64` |

---

## 10. Flag-ek referenciája (RFLAGS)

| Bit | Név | Jel | Leírás |
|-----|-----|-----|--------|
| 0 | Carry Flag | CF | Átvitel/kölcsönzés, előjel nélküli túlcsordulás |
| 2 | Parity Flag | PF | Páros számú 1-es bit az eredményben |
| 4 | Auxiliary Carry | AF | BCD számtanhoz (4. bitből átvitel) |
| 6 | Zero Flag | ZF | Eredmény nulla |
| 7 | Sign Flag | SF | Eredmény negatív (MSB=1) |
| 8 | Trap Flag | TF | Léptetéses futtatás (debugger) |
| 9 | Interrupt Enable | IF | Megszakítások engedélyezése |
| 10 | Direction Flag | DF | String utasítások iránya (0=előre, 1=hátra) |
| 11 | Overflow Flag | OF | Előjeles túlcsordulás |

---

## 11. Méretjelölők és méret-szuffixek

| NASM jelölés | AT&T szuffixum | Méret | Leírás |
|-------------|---------------|-------|--------|
| `byte [mem]` | `b` | 1 B | 8-bit |
| `word [mem]` | `w` | 2 B | 16-bit |
| `dword [mem]` | `l` | 4 B | 32-bit |
| `qword [mem]` | `q` | 8 B | 64-bit |

```nasm
; Példák a méretjelölők használatára:
mov     byte [rdi], 0       ; 1 byte írása
mov     word [rdi], 0       ; 2 byte írása
mov     dword [rdi], 0      ; 4 byte írása
mov     qword [rdi], 0      ; 8 byte írása
```

---

## 12. Calling Convention gyorsreferencia (System V AMD64 ABI)

### Argumentum regiszterek (sorrendben)

| Sorrend | Egész/Pointer | Lebegőpontos |
|---------|---------------|--------------|
| 1. arg  | `rdi`         | `xmm0`       |
| 2. arg  | `rsi`         | `xmm1`       |
| 3. arg  | `rdx`         | `xmm2`       |
| 4. arg  | `rcx`         | `xmm3`       |
| 5. arg  | `r8`          | `xmm4`       |
| 6. arg  | `r9`          | `xmm5`       |
| 7+. arg | Stack (jobbról balra) | Stack |

### Callee-saved vs Caller-saved regiszterek

| Kategória | Regiszterek |
|-----------|-------------|
| **Callee-saved** (hívott ment) | `rbx`, `rbp`, `r12`, `r13`, `r14`, `r15`, `rsp` |
| **Caller-saved** (hívó ment) | `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`, `r9`, `r10`, `r11` |

### Visszatérési érték

| Típus | Regiszter |
|-------|-----------|
| 64-bit egész / pointer | `rax` |
| 128-bit egész | `rdx:rax` |
| float/double | `xmm0` |

### Stack igazítás

- A `call` utasítás végrehajtásakor **RSP + 8** legyen 16-tal osztható
- Azaz a `call`-t megelőzően RSP 16-ra igazított legyen
- Függvény belépésekor RSP mod 16 = 8 (a visszatérési cím miatt)
- `printf`-et és más vararg C függvényeket mindig igazított stack-kel hívd!

---

*Lásd még: [Syscall referencia](./syscall-referencia.md)*
