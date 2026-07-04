# Super-instrucciones - Fusion de patrones comunes en una sola instr VM

Las **super-instrucciones** son opcodes que combinan secuencias muy frecuentes
del codigo emitido por el IR en una sola instruccion VM. Reducen el numero de
dispatches del scheduler (cada dispatch cuesta ~3 ns) y el tamaño del bytecode.

> **Por que hace falta esto?**
> El intérprete despacha una instruccion VM a la vez via `Scheduler::run_loop`.
> Cada dispatch hace decode + lookup + ejecutar handler + advance PC. Para hot
> loops con patrones repetitivos (ej. `mov rd, rs1; add rd, rs2`), fusionar
> 2-3 instrucciones en 1 reduce el conteo de dispatches proporcionalmente.
> Las 4 familias de super-instrucciones documentadas aquí permiten una
> reducción del conteo de instrucciones VM ejecutadas del orden del 20-30%
> en hot loops típicos.

---

## Indice

1. [`cmpjmp.cc` / `cmpjmpu.cc` (0x68/0x69)](#1-cmpjmp-cmpjmpu) — comparacion + branch
2. [`decjnz` (0x6A)](#2-decjnz) — decremento + branch-if-not-zero
3. [`mvtake` (0x72)](#3-mvtake) — move-and-take para smart pointers
4. [`alu3` (0x73-0x7B)](#4-alu3-aritmetica-3-operandos) — `mov + OP` fusion
5. [`loadz` / `loadzh` (0x7C/0x7D)](#5-loadz-loadzh) — LOAD con zero-extend
6. [Reglas de diseno](#reglas-de-diseno)

---

## 1. cmpjmp / cmpjmpu (0x68 / 0x69)

Cmps + jmp.cc fusionados en una sola instruccion VM. El IR emitter detecta el
patron `CMP_* + BR_COND` y fusiona automaticamente cuando no hay phi copies
entre el cmp y el branch.

**Sintaxis:**

```c
cmpjmp.cc r_a, r_b, label // signed compare + branch
cmpjmpu.cc r_a, r_b, label // unsigned compare + branch
```

**Sufijos `.cc`:** `je/jz/jne/jnz/jcs/jb/jcc/jae/jmi/jpl/jvs/jvc/jhi/jls/jge/jlt/jgt/jle`
(mismo set que `jmp.j*`).

**Codificacion binaria (FIXED_8, 8 bytes):**

```
+--------+--------+-----------------+--------+------------------+
| 0x00 | 0x68 | (r_a<<4) | r_b | cond | target_u32_LE |
+--------+--------+-----------------+--------+------------------+
    byte 0 byte 1 byte 2 byte 3 bytes 4-7
```

- `cmpjmp.cc` usa opcode2=0x68 (signed: afecta OF/SF).
- `cmpjmpu.cc` usa opcode2=0x69 (unsigned: afecta CF).
- `target` es u32 little-endian (cabe en VA <4GB, suficiente para el code section actual).
- Relocation type `RelocTypeVELB::ABSOLUTE32` con truncate + range check en el linker.

**Ejemplo en bench_factorial:**

```asm
# Antes (2 instr VM):
cmps r1, r2
jmp.jge done_label

# Despues (1 instr VM):
cmpjmp.jge r1, r2, done_label
```

**Cuando se aplica:** el emitter IR lo emite cuando NO hay phi copies entre el
CMP y el BR_COND (verificado via `has_phi_copies_to`). Cuando si hay,
fallback al patron tradicional (las phi copies clobreraian regs del CMP).

**Bench observado:** `58_quicksort.vx` 10/11 comparaciones fusionadas (91%);
`01_factorial.vx`, `03_contador.vx`, `17_ecs_basico.vx`: 100% fusion rate.

---

## 2. decjnz (0x6A)

Decremento + branch-if-not-zero. Patron clasico de loops contador
`for (i = N; i > 0; i--)`.

**Sintaxis:**

```c
decjnz r_counter, label // r_counter -= 1; if (r_counter != 0) jmp label
```

**Codificacion (FIXED_8):**

```
+--------+--------+----------------+--------+------------------+
| 0x00 | 0x6A | (r_ctr<<4) | 0 | 0 | target_u32_LE |
+--------+--------+----------------+--------+------------------+
```

Reusa `SubOp` para flags consistentes con `subs r, 1`.

**Estado:** opcode disponible desde , pero el frontend Vesta AUN NO lo
emite automaticamente. La fusion automatica del patron `i--; if(i!=0) goto top`
requeriria reordenar las phi copies del back-edge para que capturen el valor
POST-decjnz, lo que necesita un trampoline block adicional que negaria el
ahorro. Disponible para uso manual desde `.vel` o futuro JIT con stackmaps.

---

## 3. mvtake (0x72)

Move-and-take: copia un qword desde `[r_src_addr]` a `[r_dst_addr]` y zerifica
`[r_src_addr]` en 1 instruccion VM. Primitivo de move-ownership para smart
pointers `unique<T>` / `shared<T>`: tras la operacion, el slot fuente queda
invalidado (0) lo que garantiza que el cleanup automatico en scope exit no
haga double-free.

**Sintaxis:** `mvtake r_dst_addr, r_src_addr`

**Codificacion (FIXED_4):** `[0x00][0x72][(r_src<<4)|r_dst][0x00]`

**No atomic cross-thread** por diseno (move es siempre intra-thread).

Cuando llegue el JIT, baja a 3 host x86-64: `mov rax,[src]; mov [dst],rax;
mov qword [src],0`.

---

## 4. alu3 - Aritmetica 3-operandos (0x73-0x7B)

Combinan el patron `mov rd, rs1; OP rd, rs2` (2-address codegen cuando el
regalloc no pudo coalescer dst con src1) en una sola instruccion VM.

**9 variantes:**

| Mnemonic | opcode2 | Operacion | Signed/Unsigned |
| :------- | :-----: | :-------- | :-------------- |
| `adds3` | 0x73 | `rd = rs1 + rs2` | signed (OF/SF flags) |
| `subs3` | 0x74 | `rd = rs1 - rs2` | signed |
| `muls3` | 0x75 | `rd = rs1 * rs2` | signed |
| `addu3` | 0x76 | `rd = rs1 + rs2` | unsigned (CF flag) |
| `subu3` | 0x77 | `rd = rs1 - rs2` | unsigned |
| `mulu3` | 0x78 | `rd = rs1 * rs2` | unsigned |
| `and3` | 0x79 | `rd = rs1 & rs2` | bitwise (CF=OF=0) |
| `or3` | 0x7A | `rd = rs1 | rs2` | bitwise |
| `xor3` | 0x7B | `rd = rs1 ^ rs2` | bitwise |

**Sintaxis textual:** `adds3 r_dst, r_src1, r_src2` (arity THREE).

**Codificacion (FIXED_4, Convention B / `decode_instr_raw_bytes`):**

```
+--------+----------+--------------------+--------------------+
| 0x00 | opcode2 | (r_src1<<4) | r_dst| (r_src2<<4) | flags|
+--------+----------+--------------------+--------------------+
    byte 0 byte 1 byte 2 byte 3
```

- `byte2`: `(r_src1 << 4) | r_dst` (low nibble = destino, mismo layout que `mvtake`/`addadvice`).
- `byte3`: `(r_src2 << 4) | flags_low` (flags reservados, low nibble = 0).
- La operacion se determina por `opcode2` (no por flags).

**Flags actualizados:**

- **signed** (0x73/0x74/0x75): ZF, SF segun resultado; OF segun overflow signed; CF=0.
- **unsigned** (0x76/0x77/0x78): ZF, SF; CF segun overflow unsigned; OF=0.
- **bitwise** (0x79/0x7A/0x7B): ZF, SF segun resultado; CF=OF=0.

**Ejemplo:**

```asm
# Antes (2 instr VM):
mov r1, r8 ; copy %2 a r1 (dst del muls)
muls r1, r2 ; r1 = r1 * r2

# Despues (1 instr VM):
muls3 r1, r8, r2 ; r1 = r8 * r2
```

**Cuando se aplica:** el IR emitter `emit_binop` (en `src/ir/ir_emitter.cpp`)
detecta `rd != rs1` (caso tipico del 2-address codegen cuando regalloc no
coalesce) y emite la variante 3-op via tabla `alu3_mnemonic_for()`:

```cpp
adds -> adds3, subs -> subs3, muls -> muls3,
addu -> addu3, subu -> subu3, mulu -> mulu3,
and -> and3, or -> or3, xor -> xor3
```

Fallback al patron viejo cuando: (a) no hay variante alu3 (DIV/MOD/SHL/SHR/SAR/CMP),
(b) `rd == rs1` (sin MOV de todos modos).

**Bench observado:** `bench_tight_loop` body 10 instr/iter -> 8 instr/iter
(-20%). `bench_jit_method` -14% wall time.

**Fast path en scheduler:** los 9 opcodes comparten un solo label `L_ALU3` en
`run_loop`. Switch interno por `opcode_index` para elegir la operacion. Evita
inflar la tabla con 9 entries y mantiene buena prediccion BTB.

---

## 5. loadz / loadzh (0x7C / 0x7D)

Super-instruccion LOAD con zero-extend a 64-bit. Fusiona `mov rd, 0; mov
rd_sized, [rs]` en una sola instruccion VM.

**Sintaxis:** `loadz r_dst, r_src` (memoria VM) / `loadzh r_dst, r_src` (memoria HOST).

El size (8/16/32/64-bit) se selecciona por el sufijo de tamaño del registro
destino (igual que `mov`):

- `loadz r11b, r1` -> load 8-bit
- `loadz r11w, r1` -> load 16-bit
- `loadz r11d, r1` -> load 32-bit
- `loadz r11, r1` -> load 64-bit (equivalente a mov pero zero-extend explicito)

**Por que existe:**

La VM **NO** hace zero-extend implicito al escribir bytes parciales en un
registro (a diferencia de x86-64 que zero-extiende en writes a registros de
32-bit). Por eso el IR emitter generaba historicamente:

```asm
mov r11, 0 ; clear upper bits primero
mov r11d, [r1] ; load 32-bit (preserva los upper 32 bits que ya son 0)
```

`loadz` hace ambas cosas en una sola instruccion.

**Codificacion (FIXED_4, `decode_instr_simple_mov`):**

```
+--------+----------+--------------------+--------------------+
| 0x00 | 0x7C | ctrl byte | regs byte |
+--------+----------+--------------------+--------------------+
    byte 0 byte 1 byte 2 byte 3

ctrl bits 7-6 : mode (0=8b, 1=16b, 2=32b, 3=64b)
ctrl bit 5 : is_host (0 = vm_mem, 1 = host_ptr) <-- bit s del simple_mov
regs nibbles : (r_src<<4) | r_dst
```

- `loadz` (0x7C): `is_host=0` (lee de `vm_mem`).
- `loadzh` (0x7D): `is_host=1` (lee del proceso host directamente).

**Diferencia VM vs HOST mem:**

- `loadz` se usa para cargar de la memoria virtual de la VM (resultado de
 `ALLOCA`, datos en static_data del modulo, etc.). Usa `vm_mem.read_uN()`.
- `loadzh` se usa para cargar de memoria del proceso host (resultado de
 `malloc` via `RAW_ALLOC`, payloads de objetos GC con `host_ptr`, fields de
 StringObjects, etc.). Hace deref directo del puntero.

**Ejemplo:**

```asm
# Antes (2 instr VM, 10 bytes):
mov r11, 0
mov r11d, [r1]

# Despues (1 instr VM, 4 bytes):
loadz r11d, r1
```

**Cuando se aplica:** el IR emitter en `IrOp::LOAD` lo emite cuando
`tsz < 8` (carga i8/i16/i32 / u8/u16/u32). Elige `loadzh` cuando
`operands[0].is_host_ptr=true`, `loadz` en otro caso.

**Bench observado:** `bench_struct_field` 2345 ms -> 1994 ms (**-15%**), 90M
instrucciones VM ahorradas en el bench. `bench_array_sum` -9%.

**Sobre el fast path en scheduler:** se intento añadir un handler dedicado
`L_LOADZ` en `run_loop`, pero hubo regresion +5-8% en benches polimorficos
(BTB pressure por añadir mas labels al threaded dispatch). `loadz`/`loadzh`
quedan en el slow path usando `exec_cached(exec_instr_loadz)`; el ahorro
real viene de reducir el conteo de instrucciones VM, no del cycle cost del
handler. Documentado "Leccion observada con fast paths".

---

## Reglas de diseno

1. **Solo super-instrucciones donde el patron es FRECUENTE.** Cada nuevo
 opcode consume un slot en `decode_table_extended[0x00..0xFF]` y añade
 complejidad al codegen + emitter + decoder + tests. La barrera para
 aceptar uno: el patron aparece en >=2 benches y la ganancia es >5% en
 al menos uno.

2. **Mantener la VM stack-machine-like, no microcode.** No añadir
 instrucciones especulativas, ni con multiples efectos colaterales
 ocultos. Cada super-instruccion debe ser CLARAMENTE expresable como
 "secuencia de N instrucciones VM existentes ejecutada atomicamente".

3. **Encoding consistente con familias existentes.** FIXED_4 (4 bytes) para
 variantes simples reg-reg/reg-mem, FIXED_8 (8 bytes) para variantes con
 target u32 (branches). Reusar `decode_instr_raw_bytes` (Convention B) o
 `decode_instr_simple_mov` cuando el formato coincida.

4. **Fast path en scheduler con cuidado.** Mas handlers != mas rapido. El
 numero optimo de fast-path handlers en el threaded dispatch parece ser
 ~10-12 (mas alla, BTB e icache pressure regresionan benches mixtos).
 Cuando un opcode ya no gana del fast path, dejarlo en el slow path
 (la llamada via `exec_cached` es bien predicha por BTB cuando el opcode
 es frecuente).

5. **Documentar el patron IR que se detecta.** El frontend que emite la
 super-instruccion debe explicarse aqui. Sin esto, mantenedores futuros
 no sabran cuando se aplica y podrian quitar el emit "porque no se
 ejecuta nunca" sin entender que estaba ligado a un patron especifico.

---

Ver tambien: [[Aritmetica y logica]] (ops 2-operandos tradicionales),
[[MOV, MOVH, MOVC, MOCH]] (variantes de LOAD/STORE con memoria HOST),
[[JMP]] (saltos condicionales con flags), [[REGISTROS]] (modelo de flags).
