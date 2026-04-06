Un **map file** es un archivo de texto que describe:
- símbolos globales
- direcciones finales
- secciones
- espacios de direcciones
- relocaciones aplicadas
- módulos incluidos
- tamaño final del ejecutable

```c
VELB MAP FILE
Executable: program.velb
Generated: 2026-03-29 04:28
Linker Version: 1.0

=== MODULES INCLUDED ===
[0] main.velo
[1] math.vela:module_add
[2] math.vela:module_mul

=== GLOBAL SYMBOLS ===
0x100000000  main
0x100000020  add
0x100000040  mul
0x100000080  msg

=== SECTIONS ===
[.code]
  Range: 0x100000000 - 0x100001000
  Size: 4096 bytes

[.data]
  Range: 0x100001000 - 0x100002000
  Size: 4096 bytes

=== ADDRESS SPACES ===
[0] anonymous: 0x00000000 - 0xFFFFFFFFFFFFFFFF
[1] stack:     0x100000000 - 0x1FFFFFFFFFF
...

=== RELOCATIONS APPLIED ===
Offset 0x22: REL32 -> symbol 'add'
Offset 0x44: ABS64 -> symbol 'msg'

=== OPTIMIZATIONS ===
- Eliminated 12 NOPs
- Folded 3 ADD+MOV sequences
- Removed 2 dead blocks

=== FINAL SIZE ===
Executable size: 12,288 bytes
```