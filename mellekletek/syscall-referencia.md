# Linux x86-64 Syscall Referencia

A tananyagban használt és a napi Assembly fejlesztéshez legfontosabb Linux syscall-ok részletes leírással, paraméterekkel és teljes kódpéldákkal.

---

## Syscall hívási konvenció

```
RAX  = syscall szám
RDI  = 1. argumentum
RSI  = 2. argumentum
RDX  = 3. argumentum
R10  = 4. argumentum (fontos: C-ben RCX lenne, de syscall-nál R10!)
R8   = 5. argumentum
R9   = 6. argumentum

syscall              ; kernel hívás

RAX  = visszatérési érték
       >= 0: siker
       < 0:  hiba (errno = -RAX, pl. -2 = ENOENT)
```

> **⚠️ Figyelem:** A `syscall` utasítás felülírja **RCX** és **R11** regisztereket (kernel használja). Ezeket menteni kell, ha szükség van rájuk a hívás után.

---

## Syscall táblázat

### Alap I/O syscall-ok

| Szám | Neve | Prototípus | Leírás |
|------|------|-----------|--------|
| 0 | `read` | `ssize_t read(int fd, void *buf, size_t count)` | Adatok olvasása file descriptorról |
| 1 | `write` | `ssize_t write(int fd, const void *buf, size_t count)` | Adatok írása file descriptorra |
| 2 | `open` | `int open(const char *path, int flags, mode_t mode)` | Fájl megnyitása / létrehozása |
| 3 | `close` | `int close(int fd)` | File descriptor lezárása |
| 19 | `readv` | `ssize_t readv(int fd, const struct iovec *iov, int iovcnt)` | Vektorizált olvasás |
| 20 | `writev` | `ssize_t writev(int fd, const struct iovec *iov, int iovcnt)` | Vektorizált írás |

### Folyamat kezelés

| Szám | Neve | Prototípus | Leírás |
|------|------|-----------|--------|
| 60 | `exit` | `void exit(int status)` | Folyamat leállítása (egyetlen szál) |
| 231 | `exit_group` | `void exit_group(int status)` | Összes szál leállítása |
| 57 | `fork` | `pid_t fork(void)` | Folyamat másolása |
| 59 | `execve` | `int execve(const char *path, char *const argv[], char *const envp[])` | Új program futtatása |
| 39 | `getpid` | `pid_t getpid(void)` | Aktuális folyamat ID-je |
| 61 | `wait4` | `pid_t wait4(pid_t pid, int *wstatus, int options, ...)` | Gyerekfolyamat bevárása |

### Memória kezelés

| Szám | Neve | Prototípus | Leírás |
|------|------|-----------|--------|
| 9 | `mmap` | `void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t off)` | Memória leképezés |
| 11 | `munmap` | `int munmap(void *addr, size_t len)` | Leképezés megszüntetése |
| 12 | `brk` | `void *brk(void *addr)` | Heap határának módosítása |
| 10 | `mprotect` | `int mprotect(void *addr, size_t len, int prot)` | Memóriavédelem módosítása |
| 25 | `mremap` | `void *mremap(void *old, size_t old_sz, size_t new_sz, int flags)` | Leképezés átméretezése |

### Fájlrendszer műveletek

| Szám | Neve | Prototípus | Leírás |
|------|------|-----------|--------|
| 4 | `stat` | `int stat(const char *path, struct stat *buf)` | Fájl metaadatok lekérése |
| 5 | `fstat` | `int fstat(int fd, struct stat *buf)` | fd alapján metaadatok |
| 6 | `lstat` | `int lstat(const char *path, struct stat *buf)` | Symlink metaadatok |
| 8 | `lseek` | `off_t lseek(int fd, off_t offset, int whence)` | Fájlpozíció módosítása |
| 85 | `creat` | `int creat(const char *path, mode_t mode)` | Fájl létrehozása |
| 86 | `link` | `int link(const char *old, const char *new)` | Hard link létrehozása |
| 87 | `unlink` | `int unlink(const char *path)` | Fájl törlése |
| 82 | `rename` | `int rename(const char *old, const char *new)` | Fájl átnevezése |
| 80 | `chdir` | `int chdir(const char *path)` | Munkakönyvtár váltás |
| 83 | `mkdir` | `int mkdir(const char *path, mode_t mode)` | Könyvtár létrehozása |
| 84 | `rmdir` | `int rmdir(const char *path)` | Könyvtár törlése |
| 78 | `getdents` | `int getdents(uint fd, struct linux_dirent *buf, uint cnt)` | Könyvtár bejegyzések olvasása |

### Hálózat

| Szám | Neve | Prototípus | Leírás |
|------|------|-----------|--------|
| 41 | `socket` | `int socket(int domain, int type, int protocol)` | Socket létrehozása |
| 42 | `connect` | `int connect(int fd, const struct sockaddr *addr, socklen_t len)` | Kapcsolat kezdeményezése |
| `43` | `accept` | `int accept(int fd, struct sockaddr *addr, socklen_t *len)` | Beérkező kapcsolat fogadása |
| 44 | `sendto` | `ssize_t sendto(int fd, const void *buf, size_t len, int flags, ...)` | Adat küldése |
| 45 | `recvfrom` | `ssize_t recvfrom(int fd, void *buf, size_t len, int flags, ...)` | Adat fogadása |
| 49 | `bind` | `int bind(int fd, const struct sockaddr *addr, socklen_t len)` | Cím kötés |
| 50 | `listen` | `int listen(int fd, int backlog)` | Figyelő mód |

---

## Részletes leírások és kódpéldák

---

### `read` (0)

**Prototípus:** `ssize_t read(int fd, void *buf, size_t count)`

**Paraméterek:**
- `rdi` = file descriptor (0 = stdin)
- `rsi` = puffer cím (ahova olvasunk)
- `rdx` = maximum byte-ok száma

**Visszatérés:** `rax` = ténylegesen olvasott byte-ok (0 = EOF, negatív = hiba)

```nasm
; ============================================================
; read_demo.asm – Beolvasás stdin-ről
; Fordítás:
;   nasm -f elf64 read_demo.asm -o read_demo.o
;   ld read_demo.o -o read_demo
;   echo "Hello" | ./read_demo
;   echo $?    ; olvasott byte-ok száma
; ============================================================
section .bss
    puffer  resb 256        ; olvasási puffer

section .text
    global _start

_start:
    ; read(0, puffer, 255)
    mov     rax, 0          ; sys_read
    mov     rdi, 0          ; stdin (fd=0)
    lea     rsi, [rel puffer]
    mov     rdx, 255
    syscall                 ; RAX = olvasott byte-ok

    ; write(1, puffer, rax) – visszaírjuk amit olvastunk
    mov     rdx, rax        ; byte-ok száma
    mov     rax, 1          ; sys_write
    mov     rdi, 1          ; stdout
    lea     rsi, [rel puffer]
    syscall

    ; exit(0)
    xor     rdi, rdi
    mov     rax, 60
    syscall
```

**Fontosabb `open` flag-ek (nem standard; lásd `open` syscall):**
- `O_RDONLY = 0` – csak olvasás
- `O_WRONLY = 1` – csak írás
- `O_RDWR = 2` – olvasás + írás

---

### `write` (1)

**Prototípus:** `ssize_t write(int fd, const void *buf, size_t count)`

**Paraméterek:**
- `rdi` = file descriptor (1 = stdout, 2 = stderr)
- `rsi` = puffer cím (mit írunk)
- `rdx` = byte-ok száma

**Visszatérés:** `rax` = kiírt byte-ok (negatív = hiba)

```nasm
; ============================================================
; write_demo.asm – stdout és stderr írás
; Fordítás:
;   nasm -f elf64 write_demo.asm -o write_demo.o
;   ld write_demo.o -o write_demo
;   ./write_demo
; ============================================================
section .data
    uzenet      db  "Hello, stdout!", 10    ; '\n' = 10
    UZENET_LEN  equ $ - uzenet
    hiba_msg    db  "Hiba tortent!", 10
    HIBA_LEN    equ $ - hiba_msg

section .text
    global _start

_start:
    ; stdout-ra írás
    mov     rax, 1
    mov     rdi, 1                  ; stdout
    lea     rsi, [rel uzenet]
    mov     rdx, UZENET_LEN
    syscall

    ; stderr-re írás
    mov     rax, 1
    mov     rdi, 2                  ; stderr
    lea     rsi, [rel hiba_msg]
    mov     rdx, HIBA_LEN
    syscall

    ; exit(0)
    xor     rdi, rdi
    mov     rax, 60
    syscall
```

> **💡 Tipp:** `$ - label` a NASM-ban kiszámolja a string hosszát fordítási időben. `$` az aktuális pozíció, a különbség adja a hosszt.

---

### `open` (2)

**Prototípus:** `int open(const char *path, int flags, mode_t mode)`

**Paraméterek:**
- `rdi` = fájl elérési út (null-terminált string cím)
- `rsi` = megnyitási flag-ek (kombinálhatók OR-ral)
- `rdx` = mód (csak létrehozásnál, pl. `0644`)

**Visszatérés:** `rax` = file descriptor (>=0 siker, negatív hiba)

**Fontosabb flag-ek:**

| Konstans | Érték (oktális/hex) | Leírás |
|----------|---------------------|--------|
| `O_RDONLY` | `0` | Csak olvasás |
| `O_WRONLY` | `1` | Csak írás |
| `O_RDWR` | `2` | Olvasás + írás |
| `O_CREAT` | `0o100` = `0x40` | Létrehozás ha nem létezik |
| `O_TRUNC` | `0o1000` = `0x200` | Csonkítás (meglévő tartalom törlése) |
| `O_APPEND` | `0o2000` = `0x400` | Hozzáfűzés |
| `O_NONBLOCK` | `0o4000` = `0x800` | Nem blokkoló mód |

```nasm
; ============================================================
; open_demo.asm – Fájl megnyitása és olvasása
; Fordítás:
;   nasm -f elf64 open_demo.asm -o open_demo.o
;   ld open_demo.o -o open_demo
;   echo "teszt" > /tmp/teszt.txt
;   ./open_demo
; ============================================================
section .data
    fajlnev     db  "/tmp/teszt.txt", 0

section .bss
    puffer      resb 256

section .text
    global _start

_start:
    ; fd = open("/tmp/teszt.txt", O_RDONLY)
    mov     rax, 2          ; sys_open
    lea     rdi, [rel fajlnev]
    mov     rsi, 0          ; O_RDONLY
    mov     rdx, 0          ; mode (irreleváns olvasásnál)
    syscall

    ; Hiba ellenőrzés: ha rax < 0, hiba
    test    rax, rax
    js      .hiba
    mov     rbx, rax        ; fd mentése

    ; read(fd, puffer, 255)
    mov     rax, 0
    mov     rdi, rbx
    lea     rsi, [rel puffer]
    mov     rdx, 255
    syscall
    mov     r12, rax        ; olvasott byte-ok mentése

    ; write(1, puffer, r12)
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [rel puffer]
    mov     rdx, r12
    syscall

    ; close(fd)
    mov     rax, 3
    mov     rdi, rbx
    syscall

    xor     rdi, rdi
    mov     rax, 60
    syscall

.hiba:
    ; exit(-rax) a hibakóddal
    neg     rax
    mov     rdi, rax
    mov     rax, 60
    syscall
```

---

### `close` (3)

**Prototípus:** `int close(int fd)`

**Paraméterek:** `rdi` = file descriptor

**Visszatérés:** `rax` = 0 siker, negatív hiba

```nasm
; close(fd)
mov     rax, 3
mov     rdi, rbx    ; fd
syscall
; Ellenőrzés: rax = 0 ha sikeres
```

---

### `exit` (60)

**Prototípus:** `void exit(int status)`

**Paraméterek:** `rdi` = kilépési kód (0-255)

**Visszatérés:** Nem tér vissza.

```nasm
; exit(0) – sikeres kilépés
xor     rdi, rdi
mov     rax, 60
syscall

; exit(1) – hibás kilépés
mov     rdi, 1
mov     rax, 60
syscall
```

> **💡 Tipp:** Az `exit` (60) csak az aktuális szálat állítja le. Többszálú programoknál `exit_group` (231) kell az összes szál leállításához – ez a C `exit()` könyvtárfüggvény által használt syscall.

---

### `mmap` (9)

**Prototípus:**
```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

**Paraméterek:**
- `rdi` = kért cím (0 = kernel döntsön)
- `rsi` = hossz (byte-okban, oldalhatárra kerekítve)
- `rdx` = védelmi flag-ek
- `r10` = leképezés flag-ek (**NEM rcx!**)
- `r8` = file descriptor (-1 = anonymous)
- `r9` = file offset

**Fontosabb flag-ek:**

| `prot` értékek | Leírás |
|---------------|--------|
| `PROT_NONE = 0` | Nincs hozzáférés |
| `PROT_READ = 1` | Olvasható |
| `PROT_WRITE = 2` | Írható |
| `PROT_EXEC = 4` | Futtatható |
| `PROT_READ\|PROT_WRITE = 3` | Olvasható + írható |

| `flags` értékek | Leírás |
|----------------|--------|
| `MAP_SHARED = 1` | Megosztott leképezés (fájlba ír) |
| `MAP_PRIVATE = 2` | Privát leképezés (COW) |
| `MAP_ANONYMOUS = 0x20` | Nem fájl alapú (fd=-1) |
| `MAP_FIXED = 0x10` | Pontosan `addr`-re képezze |

```nasm
; ============================================================
; mmap_demo.asm – Dinamikus memóriafoglalás mmap-pal
; Fordítás:
;   nasm -f elf64 mmap_demo.asm -o mmap_demo.o
;   ld mmap_demo.o -o mmap_demo
;   ./mmap_demo
;   echo $?    ; 42-t kell mutatni
; ============================================================
section .text
    global _start

_start:
    ; void *p = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
    ;                MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
    xor     rdi, rdi            ; addr = NULL
    mov     rsi, 4096           ; length = 4096 (1 lap)
    mov     rdx, 3              ; PROT_READ | PROT_WRITE
    mov     r10, 0x22           ; MAP_PRIVATE | MAP_ANONYMOUS (0x2 | 0x20)
    mov     r8, -1              ; fd = -1 (anonymous)
    xor     r9, r9              ; offset = 0
    mov     rax, 9              ; sys_mmap
    syscall

    ; RAX = foglalt memória pointer (vagy negatív = hiba)
    test    rax, rax
    js      .hiba

    ; Adjunk valamit a foglalt területre
    mov     qword [rax], 42

    ; Olvassuk vissza
    mov     rbx, [rax]          ; rbx = 42

    ; munmap(p, 4096)
    mov     rdi, rax
    mov     rsi, 4096
    mov     rax, 11             ; sys_munmap
    syscall

    ; exit(42)
    mov     rdi, rbx
    mov     rax, 60
    syscall

.hiba:
    mov     rdi, 1
    mov     rax, 60
    syscall
```

---

### `brk` (12)

**Prototípus:** `void *brk(void *addr)`

A program adatszegmensének végét (heap határát) állítja be. Egyszerű heap-foglaláshoz használható.

**Paraméterek:** `rdi` = új cím (0 = aktuális határ lekérdezése)

**Visszatérés:** `rax` = tényleges új cím

```nasm
; ============================================================
; brk_demo.asm – Heap foglalás brk-val
; Fordítás:
;   nasm -f elf64 brk_demo.asm -o brk_demo.o
;   ld brk_demo.o -o brk_demo
;   ./brk_demo
; ============================================================
section .text
    global _start

_start:
    ; Aktuális heap vég lekérdezése
    xor     rdi, rdi
    mov     rax, 12         ; sys_brk
    syscall
    mov     rbx, rax        ; rbx = jelenlegi heap határ

    ; Foglalj 4096 byte-ot (heap növelése)
    mov     rdi, rbx
    add     rdi, 4096
    mov     rax, 12
    syscall
    ; RAX = új heap határ

    ; Most [rbx] .. [rbx+4095] használható
    mov     qword [rbx], 99

    ; Ellenőrzés
    mov     rdi, [rbx]      ; rdi = 99
    mov     rax, 60
    syscall
```

> **💡 Tipp:** Valós programban a `mmap` + `MAP_ANONYMOUS` rugalmasabb, mint a `brk`. A C `malloc` a `brk`-t és/vagy `mmap`-et használja belül.

---

### `lseek` (8)

**Prototípus:** `off_t lseek(int fd, off_t offset, int whence)`

**Paraméterek:**
- `rdi` = file descriptor
- `rsi` = offset
- `rdx` = whence (SEEK_SET=0, SEEK_CUR=1, SEEK_END=2)

**Visszatérés:** `rax` = új file pozíció

```nasm
; Fájl végére ugrás (méret lekérdezése)
mov     rax, 8
mov     rdi, rbx    ; fd
xor     rsi, rsi    ; offset = 0
mov     rdx, 2      ; SEEK_END
syscall             ; RAX = fájlméret

; Visszaugrás az elejére
mov     rax, 8
mov     rdi, rbx
xor     rsi, rsi    ; offset = 0
xor     rdx, rdx    ; SEEK_SET
syscall
```

---

### Hibaüzenetek (errno értékek)

Ha egy syscall hibával tér vissza, `rax` negatív érték lesz (`rax = -errno`).

| errno | Konstans | Leírás |
|-------|----------|--------|
| 1 | `EPERM` | Nem engedélyezett |
| 2 | `ENOENT` | Fájl / könyvtár nem létezik |
| 9 | `EBADF` | Érvénytelen file descriptor |
| 12 | `ENOMEM` | Nincs elég memória |
| 13 | `EACCES` | Hozzáférés megtagadva |
| 14 | `EFAULT` | Rossz cím (érvénytelen pointer) |
| 16 | `EBUSY` | Erőforrás foglalt |
| 17 | `EEXIST` | Fájl már létezik |
| 19 | `ENODEV` | Nincs ilyen eszköz |
| 20 | `ENOTDIR` | Nem könyvtár |
| 21 | `EISDIR` | Könyvtár (nem fájl) |
| 22 | `EINVAL` | Érvénytelen argumentum |
| 28 | `ENOSPC` | Nincs hely az eszközön |
| 32 | `EPIPE` | Törött pipe |
| 38 | `ENOSYS` | Syscall nem létezik (**rossz syscall szám!**) |

```nasm
; Syscall hiba ellenőrzési minta:
    syscall
    test    rax, rax
    js      .hiba           ; rax < 0: hiba

; Hiba kezelésénél az errno:
.hiba:
    neg     rax             ; rax = errno (pozitív)
    ; pl. rax=2 → ENOENT (nem létező fájl)
```

---

## Standard file descriptorok

| fd | Neve | Leírás |
|----|------|--------|
| 0 | `STDIN_FILENO` | Standard bemenet |
| 1 | `STDOUT_FILENO` | Standard kimenet |
| 2 | `STDERR_FILENO` | Standard hibakimenet |

---

## Teljes syscall táblázat (Linux x86-64, legfontosabb 60)

A teljes lista a kernel forrásában: `/usr/include/asm/unistd_64.h` vagy `ausyscall --dump`

```bash
# Syscall számok lekérdezése Linux rendszeren:
ausyscall --dump | sort -n | head -30
# vagy:
grep -r 'define __NR_' /usr/include/asm/unistd_64.h | sort -t' ' -k3 -n | head -30
```

| Szám | Neve | Szám | Neve | Szám | Neve |
|------|------|------|------|------|------|
| 0 | read | 20 | writev | 40 | sendfile |
| 1 | write | 21 | access | 41 | socket |
| 2 | open | 22 | pipe | 42 | connect |
| 3 | close | 23 | select | 43 | accept |
| 4 | stat | 24 | sched_yield | 44 | sendto |
| 5 | fstat | 25 | mremap | 45 | recvfrom |
| 6 | lstat | 26 | msync | 46 | sendmsg |
| 7 | poll | 27 | mincore | 47 | recvmsg |
| 8 | lseek | 28 | madvise | 48 | shutdown |
| 9 | mmap | 29 | shmget | 49 | bind |
| 10 | mprotect | 30 | shmat | 50 | listen |
| 11 | munmap | 31 | shmctl | 51 | getsockname |
| 12 | brk | 32 | dup | 57 | fork |
| 13 | rt_sigaction | 33 | dup2 | 59 | execve |
| 14 | rt_sigprocmask | 34 | pause | 60 | exit |
| 15 | rt_sigreturn | 35 | nanosleep | 61 | wait4 |
| 16 | ioctl | 36 | getitimer | 62 | kill |
| 17 | pread64 | 37 | alarm | 63 | uname |
| 18 | pwrite64 | 38 | setitimer | 231 | exit_group |
| 19 | readv | 39 | getpid | 232 | epoll_wait |

---

## Gyors kódtempletek

### stdout-ra szöveg kiírás

```nasm
section .data
    msg     db  "Szöveg", 10    ; \n végén
    MSGLEN  equ $ - msg

section .text
    mov     rax, 1          ; sys_write
    mov     rdi, 1          ; stdout
    lea     rsi, [rel msg]
    mov     rdx, MSGLEN
    syscall
```

### stdin-ről olvasás

```nasm
section .bss
    buf     resb 1024

section .text
    mov     rax, 0          ; sys_read
    mov     rdi, 0          ; stdin
    lea     rsi, [rel buf]
    mov     rdx, 1023       ; max 1023 byte (1 null terminátornak)
    syscall
    ; RAX = tényleges olvasott byte
```

### Fájl teljes olvasása

```nasm
; 1. Megnyitás
    mov     rax, 2
    lea     rdi, [rel fajlnev]
    xor     rsi, rsi        ; O_RDONLY
    xor     rdx, rdx
    syscall
    mov     r12, rax        ; fd mentés

; 2. Olvasás
    mov     rax, 0
    mov     rdi, r12
    lea     rsi, [rel puffer]
    mov     rdx, PUFFER_MERET
    syscall
    mov     r13, rax        ; olvasott hossz

; 3. Lezárás
    mov     rax, 3
    mov     rdi, r12
    syscall
```

### Program kilépési kód

```nasm
; Sikeres kilépés:
    xor     rdi, rdi        ; 0
    mov     rax, 60
    syscall

; Hiba kilépés:
    mov     rdi, 1          ; nem nulla
    mov     rax, 60
    syscall
```

---

*Lásd még: [Utasítás referencia](./utasitas-referencia.md)*
