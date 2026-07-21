# Excepciones en JIT/AOT: in-JIT catch (Opcion B) — diseno y estado

> Estado: IMPLEMENTADO y validado (2026-06-25).  cross-function throw
> (fd7457f) + same-function in-JIT catch (este sprint).  El catch de
> excepciones de TIPO-USUARIO corre EN JIT; los catch que pueden capturar un
> AV de OS (catch-all o FatalError) bailan a interp (ver "Limitacion AV").
> Foundation: MBlock.extra_succs + force_spill + LEA_LABEL + vrt_resume_jit.

## Resumen de lo implementado

1. Foundation (commits separados): `MBlock.extra_succs` (sucesores abnormales)
   + union en la liveness; `IntervalResult.force_spill` (vregs live-in a un
   sucesor abnormal -> SPILL forzado, memory-resident).
2. `MOp::LEA_LABEL` (lea reg,[rip+disp32] a label local via MFixup) -> el
   TRYENTER carga la direccion nativa del bloque catch.
3. `ProcessVM::ExceptionFrame` + `native_catch_addr/native_rsp/native_rbp`;
   `ProcessVM::jit_exc_rsp/rbp` (handoff transitorio para no usar 5o arg).
4. `vrt_tryenter_jit(proc,type,catch_addr)` + `vrt_resume_jit(catch,rsp,rbp,
   proc)` (stub asm: restaura RBX=proc + RBP + RSP, jmp catch).  RBX critico:
   el path del throw (funciones C) no restaura el RBX del JIT (no retornan).
5. vreg: `LANDINGPAD` (lee proc->registers[0]), `TRYENTER` (handoff rsp/rbp +
   LEA_LABEL catch + CALL tryenter_jit + extra_succs), `TRYLEAVE`, `THROW`.
6. `do_throw`: si el frame matcheado es JIT (native_catch_addr!=0) ->
   vrt_resume_jit; si no, ruta interp clasica.

## Limitacion AV (access violation / fault de OS)

El in-JIT catch BAILA (corre en interp) cuando el catch puede capturar un AV
de OS: catch-all `catch()` o `catch(FatalError)` (marcado por el frontend con
`TRYENTER.imm=1`).  Motivo: un AV dispara el path `av_recovery` que hace
`longjmp` al scheduler; despues `throw_fatal`/`do_throw` corren sobre la MISMA
region de stack del frame JIT (ya muerto), clobbeando sus slots ANTES de que
vrt_resume_jit reentre -> el catch leeria basura.  El throw NORMAL
(vrt_throw_user/vrt_unwrap_throw) NO tiene este problema (stack contiguo, los
frames C estan por debajo del native_rsp).  Por eso:
- `catch(UserType)`: in-JIT (no puede capturar un AV; solo throws de usuario).
- `catch()` / `catch(FatalError)` / synchronized: bail -> interp (correcto).

Cerrar esto (in-JIT catch tambien para AV) requeriria que el av_recovery NO
clobbee los slots del frame JIT (p.ej. resume directo desde el VEH sin pasar
por el scheduler, o un stack dedicado para el recovery).  Trabajo futuro.

## (Historico) diseno original

## Problema

Una funcion JIT-compilada (vreg) con su PROPIO `try/catch`: cuando un `throw`
desenrolla al catch, `do_throw` pone `proc->rip = handler_pc` (bytecode) y el
interp resume el catch.  Pero los locals de la funcion viven en los
slots/vregs del frame JIT (host stack / registros host), NO en
`proc->registers` que el interp lee -> el catch ve basura.

Causa de fondo: el JIT compila desde el IR (SSA -> vregs) y el interp desde el
bytecode (R0-R15).  Son DOS backends independientes del mismo IR con
asignaciones de registro distintas.  Resumir el catch en interp requiere
traducir el estado JIT al estado interp (deopt cross-backend).

Por que "funciona" hoy en slots: slots NO eager-compila `main` (bail en
`__module_init`/`defclass` que slots no soporta) -> `main` (donde vive el
try/catch tipico) corre en interp -> correcto.  vreg SI soporta `defclass` ->
eager-compila `main` -> expone el bug.  No es que slots maneje deopt.

## Solucion: in-JIT catch (Opcion B)

En vez de resumir el catch en interp, **resumir el catch EN JIT** (el catch ya
es un bloque IR que vreg compila).  Asi no hay traduccion cross-backend.  Es
ademas el modelo de excepciones de AOT (que no tiene interp al que volver),
y es portable (la unica pieza arch-especifica es un stub de ~3 instrucciones).

### Mecanismo

1. **`tryenter`** (codegen vreg/AOT, arch-neutral): registra un
   `ExceptionFrame` JIT-aware con `{native_catch_addr, native_rsp, native_rbp,
   type}`.  `native_rsp/rbp` = RSP/RBP host en el punto del tryenter (el frame
   estable de la funcion: `rsp = rbp - frame_size`, sin args salientes
   activos).  `native_catch_addr` = direccion nativa del bloque catch.

2. **`throw`** (en cualquier sitio, incl. calls anidados): `do_throw` recorre
   `exc_frame_stack`.  Si el frame que matchea es JIT (`native_catch_addr !=
   0`) -> llama al primitivo de unwind; si es interp -> ruta actual
   (`rip=handler_pc` + return + `did_jump`).

3. **`vrt_resume_jit(catch_addr, native_rsp, native_rbp)`** (UNICA pieza
   arch-especifica): stub asm que hace `mov rsp,native_rsp; mov rbp,native_rbp;
   jmp catch_addr`.  Abandona los frames nativos intermedios (calls anidados +
   los frames C de `vrt_throw_user`/`do_throw`) reseteando RSP.  `do_throw` ya
   hace la limpieza VM (host_allocas, frames OOP) antes del salto.  La
   excepcion NO se pasa en reg: `do_throw` ya dejo `proc->registers[0] =
   exception_ptr`; el catch la lee via `LANDINGPAD`.

4. **`LANDINGPAD`** (vreg): primera op del catch.  Baja a `MOV dst,
   [rbx + REGISTERS_OFFSET + 0]` (lee `proc->registers[0]`, la excepcion).
   HOY vreg BAILA en LANDINGPAD -> hay que implementarlo.

### Por que la excepcion y los datos del catch son memory-resident

El frontend (`lower_try`) YA:
- modela los handlers como sucesores CFG via "edges fantasma"
  (`current_block -> handler_bb`, alcanzables via excepcion).
- spillea TODAS las vars vivas del try (incluido `this` y params) a slots del
  VM stack (`try_spill_slots_`, ALLOCA_VM + STORE).  El catch las recarga via
  `LOAD` desde el slot, DESPUES del `LANDINGPAD`.

Asi los VALORES de las vars del catch ya viven en VM memory (sobreviven al
throw porque `do_throw` restaura el VM-RSP al punto del tryenter, dejando los
slots -- alocados antes del try -- por encima del VM-RSP restaurado).

## EL CRUX PENDIENTE: force-spill de los vregs live-in al handler

Lo que NO esta resuelto: las DIRECCIONES de slot (resultado de `ALLOCA_VM`,
los `v_slot`) son valores SSA que el catch usa en sus `LOAD`.  `ALLOCA_VM`
hace un bump del VM-RSP UNA vez y guarda la direccion en un vreg; NO se
rematerializa (la direccion = valor del VM-RSP al alocar, no recomputable
desde RBP).  Si la regalloc deja `v_slot` en un registro host a traves del
try, el throw lo clobberea -> el `LOAD` del catch usa una direccion basura.

Lo mismo aplica a CUALQUIER valor SSA live-in al handler que no sea
memory-resident (p.ej. temporales definidos antes del try y usados en el
catch).

**Fix requerido (regalloc):** los vregs live-in a un bloque handler deben ser
**memory-resident (spilled a un slot host)** a traves del edge anormal del
throw.  Con la maquinaria de spill existente (`ra.spilled`/`slot_of`):
- (a) Asegurar que el edge fantasma (tryenter-block -> handler-block) este en
  el CFG de MachineIR que usa la liveness del regalloc.  **CONFIRMADO 2026-06-25
  que HOY NO lo esta**: `build_intervals` (`src/jit/interval.cpp:431-434`) hace
  `live_out[b] = union(live_in[succ_a], live_in[succ_b])` -- el bloque MachineIR
  solo tiene DOS sucesores (`succ_a`, `succ_b`), poblados desde el terminador
  (BR -> succ_a; BR_COND -> succ_a+succ_b).  El edge al handler NO es un branch
  terminador -> no cabe en succ_a/succ_b -> la liveness NO ve el handler como
  sucesor -> los valores live-in al handler se consideran MUERTOS tras su
  ultimo uso normal -> liberados -> el catch lee basura.
  **Implica un cambio estructural**: extender `MBlock` con "sucesores de
  excepcion" (lista, no solo succ_a/b) que la liveness union-e ademas de
  succ_a/succ_b; poblarlos en el vreg lowering (el bloque con TRYENTER tiene
  como exc-succ al/los handler-block(s)); y la liveness debe unir su live_in.
  Esto MANTIENE vivos los valores live-in al handler a traves del try.
- (b) Forzar spill (stack home) de todo vreg live-in a algun handler.  El def
  (p.ej. ALLOCA_VM antes del try) escribe al slot; el throw no toca el slot
  (memoria); el catch recarga del slot.  Es lo que hace LLVM con landingpad +
  valores en stack slots.

Sin (a)+(b) el in-JIT catch da resultados incorrectos (visto: 30_unwrap_null
da garbage en vez de 999).

## Respuestas a portabilidad

- **AOT**: identico.  AOT no tiene interp -> obligatoriamente usa in-JIT
  catch.  Este mecanismo ES el modelo de excepciones de AOT (mas simple que
  `.pdata`/`.xdata` PE o `.eh_frame`/DWARF ELF de la  G).
- **Multi-arch**: todo es MachineIR arch-neutral (MOV/CALL/jmp + loads + el
  force-spill) salvo el stub `vrt_resume_jit` (~3 instr/arch).  ARM64 = escribir
  esas 3 instrucciones.
- **Eficiencia**: cero `setjmp`.  Path normal = el force-spill de los live-in
  al catch (unos stores) + `tryleave` (pop).  Path de throw = frio.  El catch
  corre en JIT nativo.

## Cambios concretos (orden de implementacion)

1. `ProcessVM::ExceptionFrame`: + `native_catch_addr`, `native_rsp`,
   `native_rbp` (0 = frame interp, ruta actual intacta -> backward-compatible).
2. `vrt_resume_jit` (libvesta_rt, stub asm x86-64) + decl + wiring
   RuntimeEntries.  Auditar que `vrt_throw_user`/`do_throw` no tengan locals
   C++ con destructores no-triviales vivos al salto (son funciones C de
   punteros -> seguro; confirmar).
3. vreg `LANDINGPAD`: `MOV dst, [rbx + REGISTERS_OFFSET]`.
4. vreg `TRYENTER`: capturar native rsp/rbp (MOV reg,rsp / MOV reg,rbp) +
   native catch addr (LEA-local-label: nuevo MOp reusando el fixup de jmp/jcc,
   o lookup por handler_pc en un mapa registrado post-encode) ->
   `vrt_tryenter_jit(proc, type, native_catch_addr, native_rsp, native_rbp)`.
5. vreg `TRYLEAVE`: CALL `vrt_tryleave` (pop).
6. **Regalloc**: el crux (a)+(b) de arriba.
7. `do_throw`: rama JIT-frame -> `vrt_resume_jit`.
8. Validacion: 30_unwrap_null=999, suite try/catch, e2e, diff_harness
   vreg==interp.  Probar throw anidado (try{foo();} con foo() que lanza).

## Ya hecho (commit fd7457f)

- `do_throw` guarda `if (vm->decoded_ptr)` (2 sitios): evita null-deref cuando
  el throw corre desde contexto JIT.
- vreg `UNWRAP` null-path emite `make_ret()` tras `vrt_unwrap_throw`: el frame
  nativo retorna limpio a enter_jit en el throw CROSS-FUNCTION (catch en el
  caller interp -> sus locals viven en proc->registers -> correcto).
- vreg BAILA en TRYENTER/TRYLEAVE/THROW (boundary correcto): same-function
  try/catch corre en interp hasta que el crux de regalloc este hecho.
