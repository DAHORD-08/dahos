# DAHORD Computer
## Cahier des charges, feuille de route et documentation technique

> Projet personnel de longue durée — implémentation en C d'un ordinateur virtuel complet, de son jeu d'instructions, de son assembleur et, progressivement, d'un système d'exploitation exécuté sur cette machine virtuelle.

---

# 1. Vision du projet

## 1.1 Objectif général

Construire en C une machine informatique virtuelle complète appelée provisoirement **DAHORD Computer**.

Le projet doit progressivement reproduire plusieurs couches d'un ordinateur réel :

```text
Applications
     ↓
DAHORD OS
     ↓
API système / appels système
     ↓
Gestion des processus / mémoire / fichiers
     ↓
DAHORD CPU + périphériques virtuels
     ↓
DAHORD ISA (jeu d'instructions)
     ↓
RAM / stockage virtuels
     ↓
Programme C exécuté sur la machine hôte
```

Le projet ne nécessite pas d'assembleur x86, ARM ou RISC-V réel.

L'assembleur utilisé par le projet sera un **assembleur personnalisé**, destiné uniquement à la machine virtuelle.

Le cœur du projet est donc :

1. concevoir une architecture de CPU ;
2. implémenter ce CPU en C ;
3. définir une mémoire virtuelle ;
4. définir un jeu d'instructions ;
5. créer un format binaire ;
6. créer un assembleur ;
7. exécuter des programmes sur la machine virtuelle ;
8. construire un système d'exploitation pour cette machine ;
9. ajouter des périphériques virtuels ;
10. ajouter des outils de développement et de débogage.

---

# 2. Pourquoi ce projet existe

Le projet n'a pas pour but de créer un énième langage généraliste.

Le but est de comprendre et d'implémenter les mécanismes fondamentaux qui composent une machine informatique :

- représentation binaire ;
- architecture CPU ;
- registres ;
- ALU ;
- mémoire ;
- pile ;
- instructions ;
- adressage ;
- appels de fonctions ;
- interruptions simulées ;
- processus ;
- ordonnanceur ;
- système de fichiers ;
- entrées/sorties ;
- pilotes virtuels ;
- communication entre programmes ;
- débogage ;
- éventuellement réseau.

Le projet constitue donc un laboratoire informatique personnel.

---

# 3. Contraintes

## 3.1 Langage principal

Le cœur du projet est développé en **C**.

C est utilisé pour :

- CPU ;
- mémoire ;
- assembleur ;
- format binaire ;
- machine virtuelle ;
- système de fichiers ;
- processus ;
- scheduler ;
- kernel ;
- outils de diagnostic.

## 3.2 Assembleur

Aucun assembleur matériel n'est nécessaire dans la version principale.

Il n'est pas nécessaire d'écrire du :

- x86 assembly ;
- x86-64 assembly ;
- ARM assembly ;
- RISC-V assembly.

Le mot « assembleur » désigne ici un programme C qui traduit la syntaxe textuelle de DAHORD ASM en instructions binaires destinées au DAHORD CPU.

## 3.3 Bibliothèques externes

La première version doit privilégier la bibliothèque standard du C et les mécanismes de l'OS hôte.

Des bibliothèques peuvent être ajoutées plus tard.

SDL2 peut notamment servir à créer une interface graphique de débogage, mais elle ne doit pas être une dépendance du cœur du CPU.

---

# 4. Architecture globale du dépôt

Organisation initiale proposée :

```text
dahord-computer/
│
├── README.md
├── LICENSE
├── Makefile
├── docs/
│   ├── architecture.md
│   ├── isa.md
│   ├── memory.md
│   ├── binary-format.md
│   ├── assembler.md
│   ├── cpu.md
│   ├── os.md
│   └── development.md
│
├── include/
│   ├── cpu.h
│   ├── memory.h
│   ├── instruction.h
│   ├── vm.h
│   ├── assembler.h
│   └── types.h
│
├── src/
│   ├── cpu.c
│   ├── memory.c
│   ├── instruction.c
│   ├── vm.c
│   ├── assembler.c
│   └── main.c
│
├── assembler/
│   ├── lexer.c
│   ├── parser.c
│   ├── encoder.c
│   └── assembler.c
│
├── os/
│   ├── kernel/
│   ├── shell/
│   ├── process/
│   ├── memory/
│   ├── filesystem/
│   └── drivers/
│
├── programs/
│   ├── hello.asm
│   ├── calculator.asm
│   └── tests.asm
│
├── tests/
│   ├── test_cpu.c
│   ├── test_memory.c
│   ├── test_assembler.c
│   └── test_filesystem.c
│
└── tools/
    ├── disassembler.c
    ├── debugger.c
    └── hexdump.c
```

Cette structure pourra évoluer.

---

# 5. Étape 0 — Maîtriser les bases C nécessaires

Avant le CPU, les notions suivantes doivent être suffisamment comprises :

- types ;
- `struct` ;
- `enum` ;
- tableaux ;
- pointeurs ;
- pointeurs vers structures ;
- chaînes C ;
- `size_t` ;
- types à largeur fixe ;
- fichiers ;
- bitwise operators ;
- allocation dynamique ;
- durée de vie des objets ;
- `malloc` ;
- `calloc` ;
- `realloc` ;
- `free` ;
- erreurs d'allocation ;
- segmentation faults ;
- comportement indéfini.

Référence générale :

https://en.cppreference.com/w/c

La documentation de `malloc`, `realloc` et `free` est particulièrement importante :

https://en.cppreference.com/w/c/memory/malloc

---

# 6. Allocation mémoire

## 6.1 Principe

```c
int *numbers = malloc(10 * sizeof(*numbers));

if (numbers == NULL) {
    return 1;
}
```

Puis :

```c
free(numbers);
```

Règle fondamentale :

> Toute mémoire allouée dynamiquement doit avoir une responsabilité claire de libération.

## 6.2 `realloc`

Exemple :

```c
int *tmp = realloc(numbers, 20 * sizeof(*numbers));

if (tmp == NULL) {
    free(numbers);
    return 1;
}

numbers = tmp;
```

Attention : ne jamais écraser directement un pointeur important avec `realloc` sans réfléchir au cas d'échec.

## 6.3 Objectif pédagogique

Avant le CPU, implémenter :

- tableau dynamique ;
- chaîne dynamique ;
- buffer dynamique ;
- liste chaînée ;
- table de hachage simple.

Ces structures seront réutilisées dans le projet.

---

# 7. Types numériques

Utiliser `<stdint.h>` pour les données de la machine virtuelle :

```c
#include <stdint.h>

uint8_t  byte;
uint16_t word16;
uint32_t word32;
uint64_t word64;
```

Pour DAHORD Computer, une première architecture **32 bits** est recommandée.

Cela signifie notamment :

- registres généraux de 32 bits ;
- adresses de 32 bits ;
- valeurs principales de 32 bits.

La RAM virtuelle n'a pas besoin de faire réellement 4 Gio au départ.

Exemple :

```c
#define RAM_SIZE (16 * 1024 * 1024)

uint8_t *ram;
```

---

# 8. Étape 1 — Définir l'architecture du CPU

Une architecture initiale possible :

```text
R0
R1
R2
R3
R4
R5
R6
R7

PC
SP
FLAGS
```

## Registres

### R0-R7

Registres généraux de 32 bits.

### PC — Program Counter

Adresse de la prochaine instruction.

### SP — Stack Pointer

Adresse du sommet de la pile.

### FLAGS

Registre contenant différents indicateurs :

```text
Z = Zero
N = Negative
C = Carry
V = Overflow
```

Le nombre exact de flags pourra évoluer.

---

# 9. ALU

L'ALU virtuelle exécute les opérations arithmétiques et logiques.

Opérations initiales :

```text
ADD
SUB
MUL
DIV
AND
OR
XOR
NOT
SHL
SHR
```

Exemple :

```text
ADD R0, R1
```

signifie :

```text
R0 = R0 + R1
```

Une implémentation simple pourrait commencer par :

```c
uint32_t alu_add(uint32_t a, uint32_t b);
uint32_t alu_sub(uint32_t a, uint32_t b);
uint32_t alu_and(uint32_t a, uint32_t b);
uint32_t alu_or(uint32_t a, uint32_t b);
uint32_t alu_xor(uint32_t a, uint32_t b);
```

---

# 10. Cycle d'exécution CPU

Le cœur du CPU doit suivre un cycle de type :

```text
FETCH
  ↓
DECODE
  ↓
EXECUTE
  ↓
UPDATE
  ↓
FETCH
```

Exemple :

```c
while (cpu->running) {
    uint32_t instruction = memory_fetch(...);

    decode_instruction(...);

    execute_instruction(...);
}
```

Version conceptuelle :

```c
void cpu_step(CPU *cpu)
{
    Instruction instruction = fetch(cpu);
    decode(&instruction);
    execute(cpu, &instruction);
}
```

Cette séparation est importante pour faciliter le débogage.

---

# 11. Définition de l'ISA

ISA = Instruction Set Architecture.

L'ISA décrit ce que le CPU sait faire.

Une première ISA pourrait contenir :

## Transfert

```text
MOV
LOAD
STORE
PUSH
POP
```

## Arithmétique

```text
ADD
SUB
MUL
DIV
INC
DEC
```

## Logique

```text
AND
OR
XOR
NOT
SHL
SHR
```

## Comparaison

```text
CMP
TEST
```

## Contrôle

```text
JMP
JZ
JNZ
JG
JL
CALL
RET
```

## Système

```text
HALT
NOP
SYSCALL
```

## Entrées/sorties

```text
IN
OUT
```

Le jeu d'instructions doit rester petit au début.

---

# 12. Format d'une instruction

Une première possibilité :

```text
┌──────────┬──────────┬──────────┬──────────┐
│ OPCODE   │ OPERAND1 │ OPERAND2 │ OPERAND3 │
│  8 bits  │  8 bits  │  8 bits  │  8 bits  │
└──────────┴──────────┴──────────┴──────────┘
```

Soit une instruction de 32 bits.

Exemple fictif :

```text
ADD R0, R1
```

pourrait devenir :

```text
[opcode ADD][register R0][register R1][unused]
```

Les détails devront être définis précisément dans `docs/isa.md`.

---

# 13. Opcode

Chaque instruction possède un identifiant numérique.

Exemple :

```c
typedef enum {
    OP_NOP   = 0x00,
    OP_MOV   = 0x01,
    OP_LOAD  = 0x02,
    OP_STORE = 0x03,
    OP_ADD   = 0x10,
    OP_SUB   = 0x11,
    OP_MUL   = 0x12,
    OP_DIV   = 0x13,
    OP_JMP   = 0x20,
    OP_JZ    = 0x21,
    OP_CALL  = 0x30,
    OP_RET   = 0x31,
    OP_HALT  = 0xFF
} Opcode;
```

Ces valeurs sont des exemples et ne constituent pas une spécification définitive.

---

# 14. Mémoire virtuelle

La mémoire du DAHORD Computer est un tableau de bytes.

```c
typedef struct {
    uint8_t *data;
    size_t size;
} Memory;
```

Fonctions :

```c
uint8_t  memory_read8(Memory *memory, uint32_t address);
uint16_t memory_read16(Memory *memory, uint32_t address);
uint32_t memory_read32(Memory *memory, uint32_t address);

void memory_write8(Memory *memory, uint32_t address, uint8_t value);
void memory_write16(Memory *memory, uint32_t address, uint16_t value);
void memory_write32(Memory *memory, uint32_t address, uint32_t value);
```

Toujours vérifier les limites :

```text
address + size <= memory->size
```

Sinon :

```text
MEMORY_FAULT
```

---

# 15. Endianness

Le format binaire doit définir explicitement l'ordre des octets.

Exemple little-endian :

```text
uint32_t 0x12345678

78 56 34 12
```

Il ne faut jamais laisser ce comportement implicite dans le format de fichiers.

---

# 16. Stack

La machine doit disposer d'une pile.

Exemple conceptuel :

```text
HIGH ADDRESS
┌───────────────┐
│ return addr   │
├───────────────┤
│ local data    │
├───────────────┤
│ saved R0      │
├───────────────┤
│ saved R1      │
└───────────────┘
LOW ADDRESS
```

`SP` indique le sommet.

Instructions :

```text
PUSH R0
POP R0
CALL function
RET
```

La pile sera indispensable pour :

- appels de fonctions ;
- variables locales ;
- sauvegarde de registres ;
- processus.

---

# 17. Premier programme machine

Objectif :

```text
MOV R0, 40
MOV R1, 2
ADD R0, R1
HALT
```

Résultat attendu :

```text
R0 = 42
```

Avant toute notion d'OS, ce test doit fonctionner.

---

# 18. Premier programme assembleur

Syntaxe proposée :

```asm
MOV R0, 40
MOV R1, 2
ADD R0, R1
HALT
```

Commande :

```bash
dahord-as hello.asm -o hello.bin
```

Puis :

```bash
dahord-run hello.bin
```

Résultat :

```text
Program terminated.

R0 = 42
```

---

# 19. Assembleur personnalisé

L'assembleur est un programme C.

Pipeline :

```text
source.asm
    ↓
lexer
    ↓
tokens
    ↓
parser
    ↓
instructions structurées
    ↓
encodeur
    ↓
programme binaire
```

## Lexer

Exemple :

```text
MOV R0, 42
```

devient :

```text
IDENTIFIER("MOV")
REGISTER("R0")
COMMA
NUMBER(42)
```

## Parser

Le parser transforme les tokens en représentation structurée.

Exemple :

```c
typedef struct {
    Opcode opcode;
    Operand operands[3];
    size_t operand_count;
} Instruction;
```

## Encoder

L'encodeur transforme cette structure en bytes.

---

# 20. Labels

L'assembleur devra éventuellement supporter :

```asm
start:
    MOV R0, 0
    JMP loop

loop:
    ADD R0, 1
    JMP loop
```

Le problème est qu'un label n'est pas directement une adresse.

Il faudra donc gérer une table de symboles :

```text
start -> 0x0000
loop  -> 0x0008
```

Deux méthodes possibles :

1. deux passes ;
2. relocation plus sophistiquée.

La méthode à deux passes est recommandée au début.

---

# 21. Désassembleur

Créer l'outil inverse :

```bash
dahord-dis hello.bin
```

Résultat :

```asm
0000  MOV R0, 40
0004  MOV R1, 2
0008  ADD R0, R1
000C  HALT
```

Cela deviendra extrêmement utile pour le débogage.

---

# 22. Format binaire

Définir un format de fichier.

Exemple :

```text
MAGIC
VERSION
FLAGS
ENTRY_POINT
CODE_SIZE
DATA_SIZE
...
CODE
DATA
```

Exemple de header C :

```c
typedef struct {
    uint32_t magic;
    uint16_t version;
    uint16_t flags;
    uint32_t entry_point;
    uint32_t code_size;
    uint32_t data_size;
} BinaryHeader;
```

Le format devra être documenté et versionné.

---

# 23. Loader

Le loader doit :

1. ouvrir le fichier ;
2. vérifier le magic number ;
3. vérifier la version ;
4. vérifier les tailles ;
5. allouer ou préparer la mémoire ;
6. charger le code ;
7. charger les données ;
8. initialiser `PC` ;
9. initialiser `SP` ;
10. lancer le CPU.

Il doit refuser les fichiers corrompus.

---

# 24. Gestion des erreurs

Prévoir des erreurs explicites :

```text
INVALID_OPCODE
INVALID_REGISTER
INVALID_ADDRESS
MEMORY_FAULT
STACK_OVERFLOW
STACK_UNDERFLOW
DIVISION_BY_ZERO
INVALID_BINARY
INVALID_MAGIC
INVALID_VERSION
SYSCALL_ERROR
```

Une erreur doit être diagnostiquable.

Exemple :

```text
CPU FAULT

Type: MEMORY_FAULT
PC:   0x001024
Address: 0xFFFFFFF0
Access: READ
Size: 4
```

---

# 25. Tests

Le projet doit être développé avec des tests.

Exemples :

```text
ADD(40, 2) == 42
SUB(50, 8) == 42
AND(0b1100, 0b1010) == 0b1000
```

Tests mémoire :

```text
write32(100, 42)
read32(100) == 42
```

Tests CPU :

```text
MOV R0, 42
HALT

R0 == 42
```

Tests assembleur :

```text
MOV R0, 42
```

doit produire les bytes attendus.

---

# 26. Debugger

Créer un debugger textuel.

Exemple :

```bash
dahord-debug program.bin
```

Commandes :

```text
help
run
step
continue
regs
memory
break
delete
disassemble
stack
quit
```

Exemple :

```text
(dahord) regs

R0  = 0x0000002A
R1  = 0x00000002
R2  = 0x00000000
R3  = 0x00000000

PC  = 0x00000014
SP  = 0x00FFF000

FLAGS = Z=0 N=0 C=0 V=0
```

Commande :

```text
step
```

doit exécuter exactement une instruction.

---

# 27. Breakpoints

Exemple :

```text
break 0x0040
run
```

Le CPU doit s'arrêter avant ou après l'exécution de l'instruction selon la convention choisie.

---

# 28. Périphériques virtuels

Une fois le CPU stable, créer des périphériques.

Exemples :

```text
Keyboard
Display
Timer
Storage
Serial
Network
```

Chaque périphérique doit avoir une interface documentée.

---

# 29. Console virtuelle

Premier périphérique recommandé : sortie texte.

Exemple :

```asm
MOV R0, 'H'
OUT CONSOLE, R0
```

Puis :

```asm
PRINT "Hello"
```

si une pseudo-instruction ou un mécanisme adapté est ajouté.

---

# 30. Clavier virtuel

Créer un buffer d'entrée :

```text
keyboard buffer
       ↓
OS
       ↓
shell
```

Le shell peut alors être interactif.

---

# 31. Timer virtuel

Le timer permettra ensuite de créer un scheduler.

Concept :

```text
tick 1
tick 2
tick 3
...
```

Le système peut recevoir un événement périodique.

---

# 32. Système de fichiers virtuel

Le stockage virtuel peut commencer très simplement.

Format conceptuel :

```text
DISK
├── superblock
├── file table
├── directories
└── data blocks
```

Fonctions :

```text
create
open
read
write
close
delete
mkdir
readdir
```

Commandes shell :

```text
ls
cd
pwd
cat
mkdir
touch
rm
write
```

---

# 33. Shell

Le shell constitue l'interface utilisateur.

Exemple :

```text
DAHORD OS v0.1

dahord> help
dahord> ls
dahord> cat hello.txt
dahord> mkdir projects
dahord> cd projects
dahord> run hello
```

Le shell doit être une application du système, autant que possible, plutôt qu'un bloc monolithique du kernel.

---

# 34. Processus

Ajouter une structure :

```c
typedef struct {
    uint32_t pid;
    CPUContext context;
    ProcessState state;
    uint32_t memory_base;
    uint32_t memory_size;
} Process;
```

États possibles :

```text
READY
RUNNING
BLOCKED
TERMINATED
```

---

# 35. Scheduler

Premier scheduler :

```text
Round Robin
```

Exemple :

```text
Process 1
   ↓
Process 2
   ↓
Process 3
   ↓
Process 1
```

Chaque processus reçoit un quantum.

Exemple :

```text
P1 : 10 ticks
P2 : 10 ticks
P3 : 10 ticks
```

---

# 36. Context switch

Lors d'un changement de processus :

```text
CPU registers
     ↓
save Process A context
     ↓
load Process B context
     ↓
CPU continues
```

Le contexte doit au minimum contenir :

- registres ;
- PC ;
- SP ;
- flags.

---

# 37. Appels système

Définir une API système.

Exemples :

```text
SYS_EXIT
SYS_WRITE
SYS_READ
SYS_OPEN
SYS_CLOSE
SYS_ALLOC
SYS_SLEEP
SYS_GETPID
```

Un programme utilisateur ne doit pas accéder directement à toutes les ressources.

---

# 38. Séparation kernel / user space

Même dans une machine virtuelle simplifiée, créer une séparation logique :

```text
USER SPACE
------------------
Applications
Shell
Programs

KERNEL SPACE
------------------
Scheduler
Filesystem
Memory manager
Drivers
Syscalls

HARDWARE VIRTUEL
------------------
CPU
RAM
Devices
```

Cela permettra de comprendre les principes des systèmes d'exploitation modernes.

---

# 39. Gestion mémoire du système

Ajouter progressivement :

- allocation kernel ;
- allocation utilisateur ;
- régions mémoire ;
- protection logique ;
- heap ;
- stack ;
- éventuellement pagination virtuelle.

Ne pas commencer directement par la mémoire virtuelle avancée.

---

# 40. Permissions

Ajouter :

```text
read
write
execute
```

Puis éventuellement :

```text
owner
group
permissions
```

Exemple :

```text
-rwxr-x---
```

Cela peut être inspiré des systèmes Unix sans chercher à reproduire Unix exactement.

---

# 41. Programmes utilisateur

Créer plusieurs programmes :

```text
hello
calculator
echo
cat
ls
editor
test
```

Ils doivent utiliser les syscalls.

---

# 42. Mini éditeur de texte

Projet secondaire intégré :

```text
edit file.txt
```

Fonctions minimales :

- affichage ;
- déplacement ;
- insertion ;
- suppression ;
- sauvegarde.

Ce n'est pas obligatoire pour une première version.

---

# 43. Réseau virtuel

Phase avancée.

Créer plusieurs machines DAHORD :

```text
┌──────────────┐
│ DAHORD PC A  │
└──────┬───────┘
       │
   virtual LAN
       │
┌──────┴───────┐
│ DAHORD PC B  │
└──────────────┘
```

Définir :

- adresse ;
- paquet ;
- source ;
- destination ;
- payload ;
- checksum éventuel.

Exemple :

```c
typedef struct {
    uint32_t source;
    uint32_t destination;
    uint16_t type;
    uint16_t length;
} PacketHeader;
```

---

# 44. Interface graphique

Une interface graphique n'est pas prioritaire.

Elle peut être ajoutée après la stabilité du cœur.

Objectifs :

```text
CPU
Registers
RAM
Stack
Instructions
Terminal
Processes
Filesystem
```

SDL2 est une option pertinente pour afficher une interface multiplateforme.

Documentation :

https://wiki.libsdl.org/

---

# 45. Architecture logicielle recommandée

Séparer :

```text
Core
 ├── CPU
 ├── Memory
 ├── ISA
 └── VM

Tools
 ├── Assembler
 ├── Disassembler
 └── Debugger

Hardware
 ├── Console
 ├── Keyboard
 ├── Timer
 ├── Storage
 └── Network

OS
 ├── Kernel
 ├── Scheduler
 ├── Memory Manager
 ├── Filesystem
 ├── Syscalls
 └── Shell

Programs
 ├── Shell utilities
 ├── Editor
 └── Applications
```

Une dépendance importante doit toujours aller vers une couche inférieure, jamais dans l'autre sens.

---

# 46. Compilation

Première commande possible :

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -g \
    src/*.c -Iinclude \
    -o dahord
```

Options importantes :

```text
-std=c17       version du langage
-Wall          warnings principaux
-Wextra        warnings supplémentaires
-Wpedantic     respect plus strict du standard
-g             informations de debug
```

Selon le compilateur et l'environnement, d'autres options peuvent être ajoutées.

---

# 47. Makefile

Créer un Makefile :

```make
CC = gcc
CFLAGS = -std=c17 -Wall -Wextra -Wpedantic -g

SRC = $(wildcard src/*.c)
OBJ = $(SRC:.c=.o)

dahord: $(OBJ)
	$(CC) $(OBJ) -o $@

clean:
	rm -f $(OBJ) dahord
```

Puis :

```bash
make
make clean
```

Le Makefile doit progressivement gérer :

- build debug ;
- build release ;
- tests ;
- nettoyage ;
- outils ;
- documentation éventuellement.

---

# 48. Débogage avec GDB

GDB sera très utile pour comprendre :

- segmentation faults ;
- pointeurs invalides ;
- corruption mémoire ;
- boucles ;
- structures ;
- stack ;
- appels de fonctions.

Commandes utiles :

```text
break main
run
next
step
continue
print variable
display variable
backtrace
info locals
watch variable
```

Documentation officielle :

https://sourceware.org/gdb/documentation/

---

# 49. Détection des erreurs mémoire

Utiliser des outils comme AddressSanitizer lorsque disponibles.

Exemple :

```bash
gcc -fsanitize=address,undefined \
    -g \
    src/*.c \
    -o dahord
```

Objectif :

détecter notamment :

- use-after-free ;
- buffer overflow ;
- accès invalides ;
- double free ;
- certaines formes d'undefined behavior.

---

# 50. Git

Le dépôt doit être versionné dès le début.

Structure de commits recommandée :

```text
feat(cpu): add register file
feat(memory): implement read/write
feat(isa): add arithmetic instructions
feat(assembler): parse registers
fix(memory): reject out-of-bounds access
test(cpu): add ADD instruction tests
docs(isa): document instruction encoding
```

Éviter les commits vagues :

```text
update
stuff
test
final
final2
```

---

# 51. Documentation interne

Chaque composant important doit avoir sa documentation.

Exemple :

```text
docs/
├── architecture.md
├── isa.md
├── memory.md
├── binary-format.md
├── assembler.md
├── cpu.md
├── devices.md
├── filesystem.md
├── processes.md
├── syscalls.md
└── debugger.md
```

La documentation doit répondre à :

- Que fait le composant ?
- Quelle est son interface ?
- Quelles données utilise-t-il ?
- Quelles erreurs peut-il produire ?
- Quelles invariantes doivent rester vraies ?
- Comment le tester ?

---

# 52. Roadmap globale

## Phase 0 — C

- pointeurs ;
- structures ;
- allocation ;
- fichiers ;
- bitwise ;
- tableaux dynamiques ;
- listes ;
- hash tables.

## Phase 1 — CPU minimal

- registres ;
- PC ;
- SP ;
- flags ;
- fetch ;
- decode ;
- execute ;
- HALT.

## Phase 2 — ISA

- MOV ;
- LOAD ;
- STORE ;
- ADD ;
- SUB ;
- JMP ;
- CMP ;
- conditions ;
- CALL ;
- RET.

## Phase 3 — Mémoire

- RAM ;
- lecture ;
- écriture ;
- validation ;
- stack.

## Phase 4 — Assembleur

- lexer ;
- parser ;
- registres ;
- immédiats ;
- labels ;
- deux passes ;
- génération binaire.

## Phase 5 — Exécutables

- format binaire ;
- header ;
- loader ;
- entry point ;
- sections.

## Phase 6 — Outils

- désassembleur ;
- debugger ;
- hexdump ;
- traces CPU.

## Phase 7 — Périphériques

- console ;
- clavier ;
- timer ;
- stockage.

## Phase 8 — OS

- kernel ;
- shell ;
- syscalls ;
- filesystem ;
- processus.

## Phase 9 — Scheduler

- processus ;
- contextes ;
- Round Robin ;
- sleep ;
- états.

## Phase 10 — Mémoire avancée

- heap ;
- isolation ;
- mémoire virtuelle éventuelle.

## Phase 11 — Réseau

- interfaces ;
- paquets ;
- adressage ;
- communication entre machines virtuelles.

## Phase 12 — Interface

- debugger graphique ;
- visualisation CPU ;
- visualisation RAM ;
- terminal graphique.

---

# 53. Progression adaptée aux pauses du midi

Une fonctionnalité doit être découpée en petites tâches.

Exemple :

## Fonctionnalité : ADD

### Session 1

Définir `OP_ADD`.

### Session 2

Définir le format de l'instruction.

### Session 3

Ajouter le décodage.

### Session 4

Implémenter `ADD`.

### Session 5

Mettre à jour les flags.

### Session 6

Créer les tests.

### Session 7

Ajouter la documentation.

### Session 8

Tester les cas limites.

Ainsi, une pause de 30 à 60 minutes peut produire une avancée concrète.

---

# 54. Définition d'une « done definition »

Une fonctionnalité n'est considérée terminée que si :

- elle fonctionne ;
- elle possède au moins un test ;
- les erreurs principales sont gérées ;
- elle est documentée ;
- elle ne crée pas de fuite mémoire connue ;
- elle est intégrée au build ;
- elle ne casse pas les fonctionnalités existantes.

---

# 55. Première milestone

La première vraie version doit être extrêmement petite.

Objectif :

```asm
MOV R0, 40
MOV R1, 2
ADD R0, R1
HALT
```

Commande :

```bash
dahord-as test.asm -o test.bin
dahord test.bin
```

Résultat :

```text
HALT

R0 = 42
```

À ce stade :

```text
[ ] CPU
[ ] RAM
[ ] ISA minimale
[ ] assembleur minimal
[ ] binaire
[ ] loader
[ ] tests
```

Aucun OS.

---

# 56. Deuxième milestone

Ajouter :

```asm
MOV R0, 0
loop:
    ADD R0, 1
    CMP R0, 10
    JNZ loop
    HALT
```

Résultat :

```text
R0 = 10
```

Cela prouve que :

- les labels fonctionnent ;
- les comparaisons fonctionnent ;
- les jumps fonctionnent ;
- la boucle d'exécution fonctionne.

---

# 57. Troisième milestone

Appels de fonctions :

```asm
CALL add
HALT

add:
    ADD R0, R1
    RET
```

Cela impose :

- stack ;
- CALL ;
- RET ;
- sauvegarde d'adresse ;
- convention d'appel.

---

# 58. Quatrième milestone

Premier programme interactif :

```text
DAHORD>
```

Puis :

```text
help
```

Résultat :

```text
Available commands:

help
clear
echo
memory
registers
run
exit
```

---

# 59. Cinquième milestone

Premier filesystem :

```text
DAHORD> mkdir test
DAHORD> cd test
DAHORD> echo Hello > hello.txt
DAHORD> cat hello.txt

Hello
```

À partir de cette étape, le projet commence réellement à ressembler à un petit ordinateur utilisable.

---

# 60. Sixième milestone

Processus :

```text
DAHORD> ps

PID   STATE
1     RUNNING
2     READY
3     SLEEPING
```

Puis :

```text
DAHORD> run program
Process started: PID 4
```

---

# 61. Septième milestone

Scheduler :

```text
tick 001: PID 1
tick 002: PID 2
tick 003: PID 3
tick 004: PID 1
```

---

# 62. Huitième milestone

Plusieurs programmes peuvent fonctionner.

Exemple :

```text
PID 1 shell
PID 2 counter
PID 3 logger
```

Le système doit alterner entre eux.

---

# 63. Neuvième milestone

Créer une machine virtuelle complète lançable par une commande :

```bash
dahord machine run disk.img
```

Puis :

```text
DAHORD COMPUTER
Booting...

Memory: 16 MiB
CPU: DAHORD32
Storage: 8 MiB

Loading DAHORD OS...

DAHORD OS v0.1

dahord>
```

---

# 64. Dixième milestone

Créer plusieurs machines :

```bash
dahord machine create pc1
dahord machine create pc2
```

Puis éventuellement :

```bash
dahord network connect pc1 pc2
```

Cette phase ouvre la possibilité d'un réseau virtuel.

---

# 65. Principes techniques à respecter

## Ne pas optimiser trop tôt

Une implémentation claire vaut mieux qu'une implémentation extrêmement rapide au début.

## Séparer les responsabilités

Éviter une fonction de 500 lignes qui fait :

```text
parse + load + execute + print + error handling
```

Préférer plusieurs fonctions spécialisées.

## Vérifier les entrées

Tout fichier binaire peut être corrompu.

Tout opcode peut être invalide.

Toute adresse peut être hors limites.

## Tester les cas limites

Exemples :

```text
0
UINT32_MAX
adresse 0
dernière adresse mémoire
division par zéro
stack vide
stack pleine
opcode inconnu
fichier vide
fichier tronqué
```

---

# 66. Notions C à apprendre au fur et à mesure

## Niveau 1

- variables ;
- fonctions ;
- conditions ;
- boucles ;
- tableaux ;
- structures.

## Niveau 2

- pointeurs ;
- pointeurs de pointeurs ;
- chaînes ;
- `const` ;
- `static`.

## Niveau 3

- malloc ;
- calloc ;
- realloc ;
- free ;
- ownership ;
- durée de vie ;
- alignement.

## Niveau 4

- bitwise ;
- unions ;
- enums ;
- sérialisation ;
- fichiers binaires.

## Niveau 5

- architecture modulaire ;
- compilation séparée ;
- headers ;
- linkage ;
- build systems.

## Niveau 6

- debugging ;
- sanitizers ;
- profiling ;
- tests ;
- CI.

---

# 67. Références C

## cppreference

Référence générale :

https://en.cppreference.com/w/c

Référence du langage :

https://en.cppreference.com/w/c/language

Gestion mémoire :

https://en.cppreference.com/w/c/memory

`malloc` :

https://en.cppreference.com/w/c/memory/malloc

Types entiers :

https://en.cppreference.com/w/c/types/integer

Entrées/sorties :

https://en.cppreference.com/w/c/io

Bibliothèque standard :

https://en.cppreference.com/w/c/header

---

# 68. GCC

Documentation GCC :

https://gcc.gnu.org/onlinedocs/

GCC sera le compilateur de référence possible pour le développement initial.

---

# 69. GDB

Documentation officielle :

https://sourceware.org/gdb/documentation/

GDB doit être utilisé régulièrement plutôt que seulement lorsqu'un bug devient incompréhensible.

---

# 70. GNU Make

Documentation :

https://www.gnu.org/software/make/manual/

---

# 71. CMake

Optionnel.

Documentation :

https://cmake.org/documentation/

CMake peut remplacer ou compléter Make lorsque le projet devient suffisamment important.

---

# 72. POSIX

Documentation :

https://pubs.opengroup.org/onlinepubs/9699919799/

POSIX est surtout utile pour comprendre les interfaces Unix utilisées par les programmes C sur le système hôte.

Le projet DAHORD OS n'a pas pour objectif d'être POSIX-compatible.

---

# 73. OSDev Wiki

Ressource particulièrement utile pour comprendre les concepts de systèmes d'exploitation :

https://wiki.osdev.org/

Pages particulièrement pertinentes :

- Required Knowledge
- Memory Management
- Processes
- Scheduling
- File Systems
- Interrupts
- IPC
- Kernel Debugging
- Testing
- Instruction Set Architecture

OSDev fournit également des informations sur l'architecture des systèmes, les émulateurs et les méthodes de test.

---

# 74. QEMU

QEMU est un émulateur de machines existantes et constitue une excellente référence pour comprendre ce qu'implique l'émulation d'un ordinateur réel.

Documentation :

https://www.qemu.org/docs/master/

Il n'est pas nécessaire pour le cœur de DAHORD Computer.

Il peut servir de référence conceptuelle et éventuellement d'outil pour une extension ultérieure.

---

# 75. SDL

Pour une interface graphique éventuelle :

https://wiki.libsdl.org/

SDL peut permettre d'afficher :

- registres ;
- RAM ;
- terminal ;
- processus ;
- instructions ;
- périphériques.

---

# 76. Architecture informatique

Concepts à étudier :

```text
CPU
ALU
Register
Instruction Pointer
Stack Pointer
Flags
Memory
Bus
Instruction
Opcode
Operand
Addressing Mode
Interrupt
System Call
Process
Scheduler
Filesystem
Device
```

La lecture doit accompagner l'implémentation.

---

# 77. Instruction Set Architecture

L'ISA doit être traitée comme une véritable spécification.

Document `docs/isa.md` :

```text
Instruction
Opcode
Encoding
Operands
Effect
Flags affected
Exceptions
```

Exemple :

```text
ADD

Opcode:
0x10

Format:
ADD rd, rs

Effect:
rd ← rd + rs

Flags:
Z, N, C, V

Exceptions:
none
```

---

# 78. Versionnement de l'ISA

Chaque version doit être identifiée.

Exemple :

```text
ISA v0.1
ISA v0.2
ISA v1.0
```

Un programme compilé pour une ISA incompatible doit être refusé proprement.

---

# 79. Compatibilité

Un fichier binaire doit contenir :

```text
ISA version
Binary version
Architecture
Entry point
```

Cela permettra plus tard de faire évoluer la machine sans casser tous les programmes.

---

# 80. Disassembler et debugger : priorité élevée

Ces outils ne doivent pas être considérés comme accessoires.

Plus le projet grandit, plus il devient difficile de comprendre ce qui se passe sans outils.

Objectif final :

```text
dahord debug program.bin
```

puis :

```text
(dahord)
regs
disassemble
memory 0x1000 64
step
step
break 0x2040
continue
stack
processes
```

---

# 81. Observabilité

Prévoir des logs :

```text
[CPU] PC=0x0040 OPCODE=ADD
[CPU] R0=40 R1=2
[CPU] R0=42

[MEM] WRITE addr=0x1000 value=42

[OS] syscall=WRITE pid=2
```

Ajouter des niveaux :

```text
ERROR
WARN
INFO
DEBUG
TRACE
```

---

# 82. Performance

Seulement après une version fonctionnelle.

Mesurer :

```text
instructions / seconde
memory accesses / seconde
assembler throughput
filesystem throughput
```

Créer éventuellement :

```bash
dahord benchmark
```

---

# 83. Profiling

Identifier les véritables points coûteux avant toute optimisation.

Une optimisation n'est considérée pertinente que si elle est mesurée.

---

# 84. Sécurité et robustesse

Même si la machine est virtuelle, les entrées externes doivent être considérées comme non fiables.

Particulièrement :

- fichiers binaires ;
- programmes assembleur ;
- tailles déclarées ;
- offsets ;
- longueurs ;
- adresses ;
- labels ;
- arguments ;
- chemins du filesystem.

Ne jamais supposer qu'un fichier est valide.

---

# 85. Documentation utilisateur finale

Le dépôt final doit permettre à quelqu'un d'autre de faire :

```bash
git clone ...
make
./dahord
```

Puis de suivre :

```text
Quick Start
Architecture
ISA
Assembler
Programs
OS
Filesystem
Processes
Networking
Debugger
Development
```

---

# 86. Définition d'une version 1.0

DAHORD Computer 1.0 pourrait être considéré comme terminé lorsque :

- CPU stable ;
- ISA documentée ;
- assembleur fonctionnel ;
- désassembleur fonctionnel ;
- format binaire documenté ;
- loader fonctionnel ;
- debugger fonctionnel ;
- RAM ;
- stack ;
- console ;
- clavier ;
- filesystem ;
- shell ;
- processus ;
- scheduler ;
- syscalls ;
- plusieurs programmes utilisateurs ;
- tests automatisés ;
- documentation complète.

Le réseau et l'interface graphique peuvent rester des extensions si le calendrier devient trop contraignant.

---

# 87. Extensions possibles

Après la version 1.0 :

## Compiler un langage de haut niveau

Créer éventuellement un langage très simple qui compile vers DAHORD ASM.

```text
code source
    ↓
compiler
    ↓
DAHORD ASM
    ↓
DAHORD assembler
    ↓
DAHORD binary
    ↓
DAHORD CPU
```

Cette extension permettrait d'étudier les compilateurs sans que le compilateur soit le projet principal.

## JIT

Ajouter éventuellement une compilation dynamique.

## SMP

Simuler plusieurs CPU.

```text
CPU 0
CPU 1
CPU 2
CPU 3
```

## Virtual memory

Ajouter :

- pages ;
- page tables ;
- translation ;
- protection.

## GPU virtuel

Créer un périphérique graphique virtuel.

## Réseau complet

Créer plusieurs machines et un routeur virtuel.

---

# 88. Ce qu'il ne faut pas faire

Ne pas commencer par :

- réseau ;
- GUI ;
- mémoire virtuelle ;
- multitâche ;
- filesystem complexe ;
- compiler un langage ;
- optimisation ;
- JIT.

La priorité est :

```text
CPU
↓
RAM
↓
ISA
↓
Assembler
↓
Binary
↓
Loader
↓
Debugger
↓
Devices
↓
OS
```

---

# 89. Première session de développement

La toute première séance peut être limitée à :

```text
Créer le dépôt
Créer README.md
Créer src/
Créer include/
Créer tests/
Créer Makefile
Créer main.c
Compiler
Lancer
```

Puis :

```c
#include <stdio.h>

int main(void)
{
    puts("DAHORD Computer");
    return 0;
}
```

Objectif :

```text
DAHORD Computer
```

Le projet commence volontairement très petit.

---

# 90. Première vraie structure CPU

Exemple initial :

```c
typedef struct {
    uint32_t registers[8];

    uint32_t pc;
    uint32_t sp;
    uint32_t flags;

    bool halted;
} CPU;
```

Puis :

```c
void cpu_reset(CPU *cpu);
void cpu_step(CPU *cpu);
void cpu_run(CPU *cpu);
```

Cette API peut évoluer.

---

# 91. Première structure mémoire

```c
typedef struct {
    uint8_t *data;
    size_t size;
} Memory;
```

API :

```c
Memory *memory_create(size_t size);
void memory_destroy(Memory *memory);

bool memory_read8(
    const Memory *memory,
    uint32_t address,
    uint8_t *value
);

bool memory_write8(
    Memory *memory,
    uint32_t address,
    uint8_t value
);
```

Le retour `bool` permet d'indiquer un échec sans forcément provoquer immédiatement un crash.

---

# 92. Première machine

```c
typedef struct {
    CPU cpu;
    Memory memory;
} Machine;
```

Puis :

```c
void machine_reset(Machine *machine);
void machine_run(Machine *machine);
```

Plus tard :

```c
typedef struct {
    CPU cpu;
    Memory memory;

    Storage storage;
    Console console;
    Keyboard keyboard;
    Timer timer;

    OS os;
} Machine;
```

---

# 93. Invariants importantes

Certaines règles doivent toujours être vraies.

Exemples :

```text
PC doit être une adresse valide lorsqu'une instruction est fetchée.

SP doit rester dans la région de pile.

Aucune lecture mémoire ne doit dépasser la RAM.

Aucune écriture mémoire ne doit dépasser la RAM.

Un opcode inconnu doit provoquer une erreur contrôlée.

Un programme terminé ne doit plus être exécuté.

Un processus TERMINATED ne doit pas être sélectionné par le scheduler.
```

Ces invariants doivent être documentés et testés.

---

# 94. Philosophie du projet

Le projet doit privilégier :

```text
Comprendre > copier
Mesurer > supposer
Tester > espérer
Documenter > mémoriser
Découper > complexifier
```

Chaque grande fonctionnalité doit répondre à trois questions :

1. Pourquoi existe-t-elle ?
2. Comment fonctionne-t-elle ?
3. Comment prouver qu'elle fonctionne ?

---

# 95. Résultat final visé

À terme, une démonstration complète pourrait ressembler à :

```text
$ dahord machine run dahord.img

DAHORD COMPUTER
----------------------------

CPU: DAHORD32
RAM: 64 MiB
ISA: v1.0

Booting...

DAHORD OS v1.0

Welcome.

dahord> ls

bin/
etc/
home/
tmp/

dahord> ps

PID   STATE
1     RUNNING
2     READY
3     SLEEPING

dahord> run calculator

Process started: 4

dahord> calculator

DAHORD Calculator
> 40 + 2

42
```

Puis une autre fenêtre :

```text
DAHORD DEBUGGER

CPU
R0  = 42
R1  = 2
PC  = 0x1042
SP  = 0xFF00

Current instruction:
ADD R0, R1
```

Cette démonstration constitue l'objectif conceptuel du projet.

---

# 96. Ressources principales

## C

- cppreference C : https://en.cppreference.com/w/c
- C language reference : https://en.cppreference.com/w/c/language
- C memory management : https://en.cppreference.com/w/c/memory
- C integer types : https://en.cppreference.com/w/c/types/integer

## Compilateur

- GCC : https://gcc.gnu.org/onlinedocs/

## Débogage

- GDB : https://sourceware.org/gdb/documentation/

## Build

- GNU Make : https://www.gnu.org/software/make/manual/
- CMake : https://cmake.org/documentation/

## Systèmes d'exploitation

- OSDev Wiki : https://wiki.osdev.org/
- OSDev Main Page : https://wiki.osdev.org/Main_Page
- OSDev Emulators : https://wiki.osdev.org/Emulators
- OSDev Testing : https://wiki.osdev.org/Testing

## Émulation

- QEMU : https://www.qemu.org/docs/master/

## Graphique

- SDL Wiki : https://wiki.libsdl.org/

---

# 97. Ordre d'apprentissage recommandé

```text
C
│
├── pointeurs
├── structs
├── malloc/free
├── bitwise
├── fichiers binaires
│
▼
Structures de données
│
├── dynamic array
├── linked list
├── hash table
│
▼
Architecture CPU
│
├── registers
├── ALU
├── PC
├── SP
├── flags
│
▼
Machine virtuelle
│
├── memory
├── fetch
├── decode
├── execute
│
▼
ISA
│
▼
Assembler
│
▼
Binary format
│
▼
Loader
│
▼
Debugger
│
▼
Devices
│
▼
Kernel
│
▼
Filesystem
│
▼
Processes
│
▼
Scheduler
│
▼
Syscalls
│
▼
Networking / GUI / advanced memory
```

---

# 98. Règle finale

Le projet n'est pas une course pour obtenir rapidement un « OS ».

La progression recherchée est :

```text
Je ne comprends pas
        ↓
Je lis
        ↓
Je teste un petit concept
        ↓
Je l'implémente
        ↓
Je casse le programme
        ↓
Je comprends pourquoi
        ↓
Je corrige
        ↓
Je documente
        ↓
Je passe à la couche suivante
```

Le système final n'est que la conséquence de cette progression.

---

# 99. Objectif de long terme

Le projet doit permettre de pouvoir expliquer, code à l'appui :

> « Voilà comment une instruction part d'un fichier source assembleur, devient une représentation binaire, est chargée en mémoire, récupérée par le CPU, décodée, exécutée, puis interagit avec la mémoire, les périphériques, le système d'exploitation et les programmes utilisateurs. »

C'est le fil conducteur de l'ensemble du projet.

