# Convenciones de Encoding de Instrucciones Extendidas

Este documento explica como se codifican los bytes de las instrucciones extendidas
(prefijo 0x00) de VestaVM y por que existen dos convenciones distintas.

Es esencial leerlo antes de implementar una nueva instruccion o al depurar
problemas de "registro incorrecto" en instrucciones existentes.

---

## Por que hay dos convenciones

VestaVM fue disenada con un conjunto de instrucciones ALU inicial.  Cuando se
anadieron instrucciones nuevas (strings, closures, monitores...) se usaron
convenciones de encoding distintas porque tienen necesidades diferentes.

Las dos convenciones son:

| Convention | Instrucciones           | Decode function           |
| :--------: | :---------------------- | :------------------------ |
| A (ALU)    | 0x05-0x40 (aritmetica)  | `decode_instr_two_op_reg` |
| B (nueva)  | 0x43-0x54 (string, etc) | `decode_instr_raw_bytes`  |

---

## Convention A: instrucciones ALU (byte3 contiene los registros)

Las instrucciones aritmeticas y logicas (ADD, SUB, MUL, DIV, AND, OR, MOV...)
usan el siguiente layout de 4 bytes:

```
byte0 = 0x00   (prefijo de instruccion extendida)
byte1 = opcode  (p.ej. 0x05 para ADD)
byte2 = ctrl    bits 7-6: modo (00=8bit 01=16bit 10=32bit 11=64bit)
                bit  5:   _signed_instruct (1=con signo)
                bit  4:   direction
byte3 = regs    nibble alto (bits 7-4): r_src (registro fuente)
                nibble bajo (bits 3-0): r_dst (registro destino)
```

La funcion de decode `decode_instr_two_op_reg` lee un uint16 desde RIP+2
y extrae:

```cpp
uint8_t n1 = data & 0xFF;      // byte2: modo y flags
uint8_t n2 = (data >> 8) & 0xFF; // byte3: registros
instr.flags_info.mode = (n1 >> 6) & 0x03;
reg1 = n2 & 0x0F;              // nibble bajo de byte3 -> r_dst
reg2 = n2 >> 4;                // nibble alto de byte3 -> r_src
```

La funcion emit correspondiente (`emit_instr_reg`) hace lo inverso:

```cpp
b2 = ctrl_byte;                // modo y flags en byte2
b3 = (r_src << 4) | r_dst;    // registros en byte3
```

---

## Convention B: instrucciones nuevas (byte2 contiene los registros)

Las instrucciones de strings, excepciones y setcc usan el layout opuesto:
los registros van en byte2 y los datos adicionales van en byte3.

```
byte0 = 0x00   (prefijo)
byte1 = opcode  (p.ej. 0x46 para STRMAKE, 0x43 para SETCC)
byte2 = ctrl    nibble alto (bits 7-4): r_dst
                nibble bajo (bits 3-0): r_src
byte3 = extra   nibble alto (bits 7-4): r3 (tercer operando, si existe)
                nibble bajo (bits 3-0): datos adicionales (encoding, etc.)
```

La funcion de decode `decode_instr_raw_bytes` almacena los bytes en bruto:

```cpp
uint8_t n1 = data & 0xFF;        // byte2 crudo -> reg1
uint8_t n2 = (data >> 8) & 0xFF; // byte3 crudo -> reg2
instr.data_instruction.reg_data.reg1 = n1;
instr.data_instruction.reg_data.reg2 = n2;
```

Las funciones exec extraen los registros directamente desde reg1/reg2:

```cpp
r_dst = (reg1 >> 4) & 0xF;  // nibble alto de byte2
r_src =  reg1       & 0xF;  // nibble bajo de byte2
r3    = (reg2 >> 4) & 0xF;  // nibble alto de byte3
extra =  reg2       & 0xF;  // nibble bajo de byte3
```

Las funciones emit correspondientes son:
- `emit_str_two_reg`: para instrucciones de dos registros (TWO arity)
- `emit_instr_three_reg`: para instrucciones de tres registros (THREE arity)
- `emit_strconv`: para strconv (dos registros + literal de encoding)
- `emit_setcc`: para setcc (un registro + literal de condicion)

---

## Tabla de funciones de decode por instruccion

| opcode2 | Instruccion       | Decode function             | Emit function            |
| :-----: | :---------------- | :-------------------------- | :----------------------- |
| 0x05    | add (reg,reg)     | decode_instr_two_op_reg     | emit_instr_reg           |
| 0x14    | mov (reg,reg)     | decode_instr_simple_mov     | emit_instr_reg           |
| 0x40    | mods/modu         | decode_instr_two_op_reg     | emit_instr_reg           |
| 0x43    | setcc             | decode_instr_raw_bytes      | emit_setcc               |
| 0x44    | tryenter          | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x45    | tryleave          | nullptr (FIXED_2, sin ops)  | emit_instr_zero          |
| 0x46    | strmake           | decode_instr_raw_bytes      | emit_instr_three_reg     |
| 0x47    | strlen            | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x48    | strcat            | decode_instr_raw_bytes      | emit_instr_three_reg     |
| 0x49    | strcmp            | decode_instr_raw_bytes      | emit_instr_three_reg     |
| 0x4A    | strconv           | decode_instr_raw_bytes      | emit_strconv             |
| 0x4B    | strraw            | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x4C    | strslice          | decode_instr_raw_bytes      | emit_instr_three_reg     |
| 0x4D    | strflat           | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x4E    | strhash           | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x4F    | strintern         | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x50    | strgetenc         | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x51    | strgetbytes       | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x52    | strgetkind        | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x53    | strreserve        | decode_instr_raw_bytes      | emit_str_two_reg         |
| 0x54    | strfinalize       | decode_instr_raw_bytes      | emit_str_two_reg         |

---

## Error tipico: mezclar convenciones

El error mas comun al anadir una instruccion nueva es usar `decode_instr_two_op_reg`
cuando la instruccion deberia usar `decode_instr_raw_bytes` (o viceversa).

**Sintoma:** los registros extraidos en la funcion exec siempre son 0 o valores
inesperados aunque el bytecode compilado parezca correcto.

**Causa:** `decode_instr_two_op_reg` extrae registros de byte3 (nibble alto/bajo
de n2).  Si el emitter puso los registros en byte2 (Convention B), byte3 es 0x00
y los registros extraidos seran 0, 0 siempre.

**Como detectarlo:**
1. Compilar un .vel con la instruccion sospechosa
2. Ejecutar con algun flag de debug o imprimir el valor del registro destino
3. Si r_dst siempre vale 0 independientemente del operando, es un problema de convention

**Como corregirlo:**
- Verificar que la entrada en `decode_table.cpp` usa `decode_instr_raw_bytes` si
  la instruccion usa byte2 para registros
- Verificar que la funcion emit pone los registros en el byte correcto

---

## Guia para anadir una nueva instruccion (Convention B)

Si la nueva instruccion tiene la forma `op r_dst, r_src` (TWO arity):

1. **parser.cpp** (`InstrSet`): anadir entrada `{"mi_op", {"mi_op", OpArity::TWO}}`
2. **parser_to_bytecode.h** (`InstrTable`): anadir
   ```cpp
   {"mi_op", {{0x00, 0xXX, FIXED_4, REG, emit_str_two_reg}}}
   ```
3. **emmit_decl.h**: si se necesita una nueva funcion emit, declararla
4. **emmit_decl.cpp**: si se necesita una nueva funcion emit, implementarla
5. **decode_table.cpp**: anadir entrada en la tabla extendida con `decode_instr_raw_bytes`
6. **exec_instruction_xxx.cpp**: implementar la funcion exec extrayendo:
   ```cpp
   r_dst = (instr.data_instruction.reg_data.reg1 >> 4) & 0xF;
   r_src =  instr.data_instruction.reg_data.reg1        & 0xF;
   ```
7. **exec_instruction.h**: declarar la funcion exec

Si la nueva instruccion tiene tres registros (THREE arity), usar `emit_instr_three_reg`
y extraer el tercero con `r3 = (instr.data_instruction.reg_data.reg2 >> 4) & 0xF`.

---

## Archivos clave

| Archivo                                    | Que contiene                              |
| :----------------------------------------- | :---------------------------------------- |
| `src/runtime/decode_instruction.cpp`       | decode_instr_raw_bytes, decode_instr_two_op_reg |
| `include/runtime/decode_instruction.h`     | Declaraciones de todas las funciones decode |
| `src/runtime/decode_table.cpp`             | Tabla extendida: [opcode2] -> metadata    |
| `src/emmit/emmit_decl.cpp`                 | emit_str_two_reg, emit_strconv, emit_setcc |
| `include/emmit/parser_to_bytecode.h`       | InstrTable: mapeo mnemonic -> emit        |
| `src/parser/parser.cpp`                    | InstrSet: registro de mnemonicos y arity  |
