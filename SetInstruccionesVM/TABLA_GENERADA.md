# Tabla completa de instrucciones (GENERADA)

> No se edita a mano.  Se regenera con:
>
> ```
> test_efectos_opcodes --markdown > doc/VMdoc/SetInstruccionesVM/TABLA_GENERADA.md
> ```

Sale del codigo: las tablas de `decode_table.cpp` para el opcode,
el modo y el tamano, y el desensamblador para los operandos --que
es quien sabe de que campo sale cada registro--.  Por eso no se
puede quedar vieja, que es lo que le paso al resto de paginas: 108
instrucciones de la tabla no aparecian en ninguna.

Las paginas escritas a mano siguen siendo las que explican QUE hace
cada instruccion; esta solo garantiza que ninguna falte.

En la columna de operandos, `=` marca la posicion de destino.
`salta` marca las que transfieren control, que no se reordenan
nunca.  `sin impl.` son ranuras con nombre reservado pero sin
`exec`/`decode`: la VM las rechaza.

La columna de efectos lista lo que la instruccion toca SIN
nombrarlo en ningun operando: banderas, pila, marco y contador de
programa.  `=` delante quiere decir que lo escribe.  Sale de
analizar el codigo maquina del manejador, y un `?` final avisa de
que queda una llamada indirecta sin seguir: lo listado es cierto,
pero puede haber mas.

| instruccion | prefijo | opcode | modo | bytes | operandos | efectos | notas |
| :---------- | :-----: | :----: | :--: | :---: | :-------- | :------ | :---- |
| `EXT_OPCODE` | --- | 0x00 | NONE | 4 | =r5  | --- | sin impl. |
| `vminfo` | --- | 0x01 | NONE | 1 | --- | --- | sin impl. |
| `vminfomanager` | --- | 0x02 | NONE | 1 | --- | --- | sin impl. |
| `inc / dec` | --- | 0x04 | REG | 2 | =r3  | =flags  |  |
| `callvm` | --- | 0x10 | INMED | 10 | --- | flags =pila marco =pc ? | salta |
| `jmp` | --- | 0x11 | INMED | 10 | --- | flags =pc ? | salta |
| `push` | --- | 0x12 | REG | 2 | r3  | flags =pila marco pc  |  |
| `pop` | --- | 0x13 | REG | 2 | =r3  | =flags =pila =marco =pc  |  |
| `xchg` | --- | 0x14 | REG | 4 | =r7  | =flags =pila =marco =pc ? |  |
| `jmpr` | --- | 0x15 | REG | 2 | r3  | flags pila marco =pc  | salta |
| `callvmr` | --- | 0x16 | REG | 2 | r3  | flags =pila marco =pc ? | salta |
| `enter` | --- | 0x28 | INMED | 10 | --- | =pila =marco  |  |
| `leave` | --- | 0x29 | NONE | 1 | --- | =pila =marco  |  |
| `nop1` | --- | 0x33 | NONE | 1 | --- | --- | sin impl. |
| `callnr` | --- | 0x55 | REG | 1 | --- | --- | sin impl. |
| `ret` | --- | 0xC3 | NONE | 1 | --- | =pila =pc  | salta |
| `edmw4` | 0x00 | 0x00 | NONE | 1 | --- | --- | sin impl. |
| `edmw6` | 0x00 | 0x01 | NONE | 1 | --- | --- | sin impl. |
| `edm` | 0x00 | 0x02 | NONE | 1 | --- | --- | sin impl. |
| `hlt` | 0x00 | 0x03 | NONE | 2 | --- | ? | salta |
| `add` | 0x00 | 0x05 | REG | 4 | =r7 r8  | =flags  |  |
| `add` | 0x00 | 0x06 | MEM | 5 | =r5  | =flags ? |  |
| `add` | 0x00 | 0x07 | SIB | 6 | =r8 r7  | =flags  |  |
| `sub` | 0x00 | 0x08 | REG | 4 | =r7 r8  | =flags  |  |
| `sub` | 0x00 | 0x09 | MEM | 5 | =r5  | =flags ? |  |
| `sub` | 0x00 | 0x0A | SIB | 6 | =r8 r7  | =flags  |  |
| `mul` | 0x00 | 0x0B | REG | 4 | =r7 r8  | =flags  |  |
| `mul` | 0x00 | 0x0C | MEM | 5 | =r5  | =flags ? |  |
| `mul` | 0x00 | 0x0D | SIB | 6 | =r8 r7  | =flags  |  |
| `div` | 0x00 | 0x0E | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `div` | 0x00 | 0x0F | MEM | 5 | =r5  | =flags =pila =marco =pc ? |  |
| `div` | 0x00 | 0x10 | SIB | 6 | =r8 r7  | =flags =pila =marco =pc ? |  |
| `cmp` | 0x00 | 0x11 | REG | 4 | =r7 r8  | =flags  |  |
| `cmp` | 0x00 | 0x12 | MEM | 5 | =r5  | =flags ? |  |
| `cmp` | 0x00 | 0x13 | SIB | 6 | =r8 r7  | =flags  |  |
| `mov` | 0x00 | 0x14 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `mov` | 0x00 | 0x15 | MEM | 5 | =r5  | =flags =pila =marco =pc  |  |
| `mov` | 0x00 | 0x16 | SIB | 6 | =r8 r7  | --- |  |
| `and` | 0x00 | 0x17 | REG | 4 | =r7 r8  | =flags  |  |
| `or` | 0x00 | 0x18 | REG | 4 | =r7 r8  | =flags  |  |
| `xor` | 0x00 | 0x19 | REG | 4 | =r7 r8  | =flags  |  |
| `not` | 0x00 | 0x1A | REG | 4 | =r7 r8  | =flags =pila marco =pc  |  |
| `shl` | 0x00 | 0x1B | REG | 4 | =r7 r8  | =flags  |  |
| `shr` | 0x00 | 0x1C | REG | 4 | =r7 r8  | =flags  |  |
| `sar` | 0x00 | 0x1D | REG | 4 | =r7 r8  | =flags  |  |
| `movc` | 0x00 | 0x1E | MEM | 4 | =r5  | flags  |  |
| `movc` | 0x00 | 0x1F | REG | 4 | r5 r7  | flags  |  |
| `mkclosure` | 0x00 | 0x20 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `callclosure` | 0x00 | 0x21 | REG | 4 | =r7 r8  | =flags =pila marco =pc ? | salta |
| `mkrawclosure` | 0x00 | 0x22 | REG | 4 | =r7 r8  | --- |  |
| `callrawclosure` | 0x00 | 0x23 | REG | 4 | =r7 r8  | =pc  | salta |
| `tailcall` | 0x00 | 0x24 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? | salta |
| `isnull` | 0x00 | 0x25 | REG | 4 | =r7 r8  | --- |  |
| `unwrap` | 0x00 | 0x26 | REG | 4 | =r7 r8  | --- |  |
| `jumptable` | 0x00 | 0x27 | REG | 4 | r6 r5  | =pc  | salta |
| `typeswitch` | 0x00 | 0x28 | REG | 4 | r6 r5  | =pc  | salta |
| `future` | 0x00 | 0x29 | NONE | 2 | --- | --- |  |
| `await` | 0x00 | 0x2A | REG | 4 | =r7 r8  | --- |  |
| `fulfill` | 0x00 | 0x2B | REG | 4 | =r7 r8  | --- |  |
| `reject` | 0x00 | 0x2C | REG | 4 | =r7 r8  | --- |  |
| `jrel` | 0x00 | 0x2D | INMED | 8 | --- | flags =pc ? | salta |
| `subsp` | 0x00 | 0x2E | MEM | 5 | =r5  | =pila  |  |
| `addsp` | 0x00 | 0x2F | MEM | 5 | =r5  | =pila  |  |
| `weakref` | 0x00 | 0x30 | REG | 4 | =r7 r8  | --- |  |
| `loop` | 0x00 | 0x31 | INMED | 1 | --- | --- | sin impl. |
| `deref_weak` | 0x00 | 0x32 | REG | 4 | =r7 r8  | --- |  |
| `nop2` | 0x00 | 0x33 | NONE | 1 | --- | --- | sin impl. |
| `free_weak` | 0x00 | 0x34 | REG | 4 | =r7 r8  | --- |  |
| `monenter` | 0x00 | 0x35 | REG | 4 | =r7 r8  | --- |  |
| `monexit` | 0x00 | 0x36 | REG | 4 | =r7 r8  | =pc  |  |
| `monwait` | 0x00 | 0x37 | REG | 4 | =r7 r8  | --- |  |
| `monnoti` | 0x00 | 0x38 | REG | 4 | =r7 r8  | flags =pc ? |  |
| `monnota` | 0x00 | 0x39 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `specialize` | 0x00 | 0x3A | REG | 4 | r6 r5  | --- |  |
| `rspawn` | 0x00 | 0x3B | REG | 4 | =r7 r8  | --- |  |
| `msgsend` | 0x00 | 0x3C | REG | 4 | r6 r5 r8  | --- |  |
| `msgrecv` | 0x00 | 0x3D | REG | 4 | =r7 r8  | --- |  |
| `memsync` | 0x00 | 0x3E | REG | 4 | =r7 r8  | --- |  |
| `mod` | 0x00 | 0x40 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `mod` | 0x00 | 0x41 | MEM | 5 | =r5  | =flags =pila =marco =pc ? |  |
| `mod` | 0x00 | 0x42 | SIB | 6 | =r8 r7  | =flags =pila =marco =pc ? |  |
| `setcc` | 0x00 | 0x43 | REG | 4 | =r5  | flags ? |  |
| `tryenter` | 0x00 | 0x44 | REG | 4 | =r6 r5  | =flags pila =marco  |  |
| `tryleave` | 0x00 | 0x45 | NONE | 2 | --- | --- |  |
| `strmake` | 0x00 | 0x46 | REG | 4 | =r6 r5 r8  | =flags =pila =marco =pc ? |  |
| `strlen` | 0x00 | 0x47 | REG | 4 | =r6 r5  | --- |  |
| `strcat` | 0x00 | 0x48 | REG | 4 | =r6 r5 r8  | =flags =pila =marco =pc ? |  |
| `strcmp` | 0x00 | 0x49 | REG | 4 | =r6 r5 r8  | =flags  |  |
| `strconv` | 0x00 | 0x4A | REG | 4 | =r6 r5 r8  | =flags =pila =marco =pc ? |  |
| `strraw` | 0x00 | 0x4B | REG | 4 | =r6 r5  | --- |  |
| `strslice` | 0x00 | 0x4C | REG | 4 | =r6 r5 r8  | =flags =pila =marco =pc ? |  |
| `strflat` | 0x00 | 0x4D | REG | 4 | =r6 r5  | --- |  |
| `strhash` | 0x00 | 0x4E | REG | 4 | =r6 r5  | --- |  |
| `strintern` | 0x00 | 0x4F | REG | 4 | =r6 r5  | --- |  |
| `strgetenc` | 0x00 | 0x50 | REG | 4 | =r6 r5  | --- |  |
| `strgetbytes` | 0x00 | 0x51 | REG | 4 | =r6 r5  | --- |  |
| `strgetkind` | 0x00 | 0x52 | REG | 4 | =r6 r5  | --- |  |
| `strreserve` | 0x00 | 0x53 | REG | 4 | =r6 r5  | =flags =pila =marco =pc ? |  |
| `strfinalize` | 0x00 | 0x54 | REG | 4 | =r6 r5  | --- |  |
| `calln` | 0x00 | 0x55 | INMED | 10 | --- | =flags =pila marco =pc ? |  |
| `gchandle` | 0x00 | 0x56 | REG | 4 | =r7 r8  | pila marco ? |  |
| `getpid` | 0x00 | 0x57 | REG | 4 | =r7 r8  | --- |  |
| `spawnon` | 0x00 | 0x58 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? | salta |
| `loadmod` | 0x00 | 0x59 | REG | 4 | =r7 r8  | --- | salta |
| `panic` | 0x00 | 0x5A | REG | 4 | =r7 r8  | =flags =pila marco =pc ? |  |
| `setmethdbg` | 0x00 | 0x5B | REG | 4 | =r7 r8  | --- |  |
| `fextend` | 0x00 | 0x5C | REG | 4 | =r7 r8  | --- |  |
| `fnarrow` | 0x00 | 0x5D | REG | 4 | =r7 r8  | --- |  |
| `strmake_h` | 0x00 | 0x5E | REG | 4 | =r6 r5 r8  | =flags =pila =marco =pc ? |  |
| `fmadd` | 0x00 | 0x5F | REG | 4 | =f5 f6 f8  | --- |  |
| `getstatic` | 0x00 | 0x60 | REG | 8 | =r6 r5  | --- |  |
| `setstatic` | 0x00 | 0x61 | REG | 8 | r6 r5  | --- |  |
| `dlopen` | 0x00 | 0x62 | REG | 4 | =r6 r5 r8  | =flags =pila =marco =pc ? |  |
| `dlsym` | 0x00 | 0x63 | REG | 4 | =r6 r5 r8 r7  | =flags =pila =marco =pc ? |  |
| `callni` | 0x00 | 0x64 | REG | 4 | r6  | =flags =pila =marco =pc ? |  |
| `gcallocp` | 0x00 | 0x65 | REG | 4 | =r6 r5  | =flags =pila =marco =pc ? |  |
| `spawnargs` | 0x00 | 0x66 | REG | 4 | r6  | =flags =pila =marco =pc ? | salta |
| `fulfillhlt` | 0x00 | 0x67 | REG | 4 | r6 r5  | =pc ? |  |
| `cmpjmp` | 0x00 | 0x68 | REG | 8 | r6 r5  | =flags =pc  | salta |
| `cmpjmpu` | 0x00 | 0x69 | REG | 8 | r6 r5  | =flags =pc  | salta |
| `decjnz` | 0x00 | 0x6A | REG | 8 | =r6  | =flags pc  | salta |
| `getargc` | 0x00 | 0x6B | REG | 4 | =r7 r8  | --- |  |
| `getarg` | 0x00 | 0x6C | REG | 4 | =r7 r8  | --- |  |
| `unloadmod` | 0x00 | 0x6D | REG | 4 | =r7 r8  | --- |  |
| `getmethat` | 0x00 | 0x6E | REG | 4 | =r7 r8  | --- |  |
| `getfldat` | 0x00 | 0x6F | REG | 4 | =r7 r8  | pila marco  |  |
| `fastpush` | 0x00 | 0x70 | INMED | 4 | r0 r2 r5 r6 r8 r9 r10 r15  | =pila  |  |
| `fastpop` | 0x00 | 0x71 | INMED | 4 | =r0 =r2 =r5 =r6 =r8 =r9 =r10 =r15  | =pila  |  |
| `mvtake` | 0x00 | 0x72 | REG | 4 | =r7 r8  | --- |  |
| `adds3` | 0x00 | 0x73 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `subs3` | 0x00 | 0x74 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `muls3` | 0x00 | 0x75 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `addu3` | 0x00 | 0x76 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `subu3` | 0x00 | 0x77 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `mulu3` | 0x00 | 0x78 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `and3` | 0x00 | 0x79 | REG | 4 | =r5 r6 r8  | =flags  |  |
| `or3` | 0x00 | 0x7A | REG | 4 | =r5 r6 r8  | =flags  |  |
| `xor3` | 0x00 | 0x7B | REG | 4 | =r5 r6 r8  | =flags  |  |
| `loadz` | 0x00 | 0x7C | REG | 4 | =r7 r8  | --- |  |
| `loadzh` | 0x00 | 0x7D | REG | 4 | =r7 r8  | --- |  |
| `htrack` | 0x00 | 0x7E | REG | 4 | =r7 r8  | --- |  |
| `gcfinal` | 0x00 | 0x7F | REG | 4 | =r7 r8  | --- |  |
| `fmin` | 0x00 | 0x80 | REG | 4 | =r7 r8  | --- |  |
| `fmax` | 0x00 | 0x81 | REG | 4 | =r7 r8  | --- |  |
| `ffloor` | 0x00 | 0x82 | REG | 4 | =r7 r8  | --- |  |
| `fceil` | 0x00 | 0x83 | REG | 4 | =r7 r8  | --- |  |
| `fround` | 0x00 | 0x84 | REG | 4 | =r7 r8  | --- |  |
| `ftrunc` | 0x00 | 0x85 | REG | 4 | =r7 r8  | --- |  |
| `bitg2z` | 0x00 | 0x86 | REG | 4 | =r7 r8  | --- |  |
| `bitz2g` | 0x00 | 0x87 | REG | 4 | =r7 r8  | --- |  |
| `gccollect` | 0x00 | 0x8C | NONE | 2 | --- | =flags =pila =marco =pc ? |  |
| `gcfinalc` | 0x00 | 0x8D | REG | 4 | =r7 r8  | --- |  |
| `gcfinall` | 0x00 | 0x8E | NONE | 2 | --- | ? |  |
| `csel` | 0x00 | 0x8F | REG | 4 | =r6 r5 r8 r7  | --- |  |
| `mld` | 0x00 | 0x90 | REG | 8 | =r10 r7  | --- |  |
| `mst` | 0x00 | 0x91 | REG | 8 | r10 r7  | pila  |  |
| `sext` | 0x00 | 0x92 | REG | 4 | =r5  | --- |  |
| `newobj` | 0x00 | 0xA0 | REG | 4 | =r7 r8  | --- |  |
| `gcrun` | 0x00 | 0xA1 | NONE | 2 | --- | =flags =pila =marco =pc ? |  |
| `gcconfig` | 0x00 | 0xA2 | REG | 4 | =r7 r8  | --- |  |
| `drop` | 0x00 | 0xA3 | REG | 4 | =r7 r8  | --- |  |
| `gcwb` | 0x00 | 0xA4 | REG | 4 | =r7 r8  | --- |  |
| `gcalloc` | 0x00 | 0xA5 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `newobjs` | 0x00 | 0xA6 | REG | 4 | =r7 r8  | --- |  |
| `gcpromote` | 0x00 | 0xA7 | REG | 4 | =r7 r8  | =flags =pila marco =pc ? |  |
| `gcdemote` | 0x00 | 0xA8 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `atomicld` | 0x00 | 0xA9 | REG | 4 | =r7 r8  | --- |  |
| `atomicst` | 0x00 | 0xAA | REG | 4 | =r7 r8  | --- |  |
| `atomiccas` | 0x00 | 0xAB | REG | 6 | =r8 r7 r10 r9  | --- |  |
| `atomicadd` | 0x00 | 0xAC | REG | 6 | =r8 r7 r10 r9  | --- |  |
| `sharedstat` | 0x00 | 0xAD | REG | 4 | =r7 r8  | --- |  |
| `callitf` | 0x00 | 0xAE | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? | salta |
| `alloc` | 0x00 | 0xB0 | REG | 4 | =r7 r8  | ? |  |
| `free` | 0x00 | 0xB1 | REG | 4 | =r7 r8  | =marco ? |  |
| `realloc` | 0x00 | 0xB2 | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `memset` | 0x00 | 0xB6 | REG | 4 | =r5 r6 r8  | --- |  |
| `memseth` | 0x00 | 0xB7 | REG | 4 | =r5 r6 r8  | --- |  |
| `memcpy` | 0x00 | 0xB8 | REG | 4 | =r5 r6 r8  | --- |  |
| `memcpyh` | 0x00 | 0xB9 | REG | 4 | =r5 r6 r8  | --- |  |
| `readcur` | 0x00 | 0xC0 | REG | 4 | =r7  | --- |  |
| `writecur` | 0x00 | 0xC1 | REG | 4 | =r7  | --- |  |
| `gcderef` | 0x00 | 0xC2 | REG | 4 | =r7  | --- |  |
| `addcur` | 0x00 | 0xC3 | INMED | 6 | --- | --- |  |
| `vmcopy` | 0x00 | 0xC4 | REG | 4 | r5 r8 r2  | --- |  |
| `vcopyh` | 0x00 | 0xC5 | REG | 4 | r5 r8 r2  | --- |  |
| `getproc` | 0x00 | 0xC6 | REG | 4 | =r7 r8  | --- |  |
| `getvm` | 0x00 | 0xC7 | REG | 4 | =r7 r8  | --- |  |
| `getmgr` | 0x00 | 0xC8 | REG | 4 | =r7 r8  | --- |  |
| `defclass` | 0x00 | 0xC9 | REG | 4 | =r7 r8  | --- |  |
| `deffield` | 0x00 | 0xCA | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `defmethod` | 0x00 | 0xCB | REG | 4 | =r7 r8  | =flags =pila =marco =pc ? |  |
| `findclass` | 0x00 | 0xCC | REG | 4 | =r7 r8  | --- |  |
| `findmethod` | 0x00 | 0xCD | REG | 4 | =r7 r8  | --- |  |
| `addadvice` | 0x00 | 0xCE | REG | 4 | r6 r5  | =flags =pila =marco =pc ? |  |
| `findfield` | 0x00 | 0xCF | REG | 4 | =r7 r8  | --- |  |
| `newobjraw` | 0x00 | 0xD0 | REG | 4 | =r7 r8  | --- |  |
| `callvirt` | 0x00 | 0xD1 | REG | 4 | =r5  | =flags =pila =marco =pc ? | salta |
| `callsuper` | 0x00 | 0xD2 | REG | 4 | =r5  | =flags =pila marco =pc ? | salta |
| `throw` | 0x00 | 0xD3 | REG | 4 | =r7 r8  | --- |  |
| `rethrow` | 0x00 | 0xD4 | NONE | 2 | --- | =flags  |  |
| `getclass` | 0x00 | 0xD5 | REG | 4 | =r7 r8  | --- |  |
| `instanceof` | 0x00 | 0xD6 | REG | 4 | =r7 r8  | --- |  |
| `checkcast` | 0x00 | 0xD7 | REG | 4 | =r7 r8  | --- |  |
| `getfield` | 0x00 | 0xD8 | REG | 4 | =r5  | pila marco  |  |
| `getmethod` | 0x00 | 0xD9 | REG | 4 | =r5  | --- |  |
| `fieldcount` | 0x00 | 0xDA | REG | 4 | =r7 r8  | marco  |  |
| `methodcount` | 0x00 | 0xDB | REG | 4 | =r7 r8  | --- |  |
| `classname` | 0x00 | 0xDC | REG | 4 | =r7 r8  | --- |  |
| `classdoc` | 0x00 | 0xDD | REG | 4 | =r7 r8  | --- |  |
| `classattrcount` | 0x00 | 0xDE | REG | 4 | =r7 r8  | --- |  |
| `classattrkey` | 0x00 | 0xDF | REG | 4 | =r5  | --- |  |
| `classattrval` | 0x00 | 0xE0 | REG | 4 | =r5  | --- |  |
| `methodname` | 0x00 | 0xE1 | REG | 4 | =r7 r8  | --- |  |
| `methoddoc` | 0x00 | 0xE2 | REG | 4 | =r7 r8  | --- |  |
| `methoddesc` | 0x00 | 0xE3 | REG | 4 | =r7 r8  | --- |  |
| `methodattrcount` | 0x00 | 0xE4 | REG | 4 | =r7 r8  | --- |  |
| `methodattrkey` | 0x00 | 0xE5 | REG | 4 | =r5  | --- |  |
| `methodattrval` | 0x00 | 0xE6 | REG | 4 | =r5  | --- |  |
| `fieldname` | 0x00 | 0xE7 | REG | 4 | =r7 r8  | --- |  |
| `fielddoc` | 0x00 | 0xE8 | REG | 4 | =r7 r8  | --- |  |
| `fieldattrcount` | 0x00 | 0xE9 | REG | 4 | =r7 r8  | marco  |  |
| `fieldattrkey` | 0x00 | 0xEA | REG | 4 | =r5  | pila marco  |  |
| `fieldattrval` | 0x00 | 0xEB | REG | 4 | =r5  | pila marco  |  |
| `yield` | 0x00 | 0xEC | NONE | 2 | --- | --- |  |
| `resume` | 0x00 | 0xED | REG | 4 | r7  | =pc  |  |
| `spawn` | 0x00 | 0xEE | REG | 4 | r7  | =flags =pila =marco =pc ? | salta |
| `swapctx` | 0x00 | 0xEF | REG | 4 | =r8 r7  | =flags =pila =marco =pc  | salta |
| `fmov` | 0x00 | 0xF0 | REG | 4 | =f7 f8  | --- |  |
| `fadd` | 0x00 | 0xF1 | REG | 4 | =f7 f8  | --- |  |
| `fsub` | 0x00 | 0xF2 | REG | 4 | =f7 f8  | --- |  |
| `fmul` | 0x00 | 0xF3 | REG | 4 | =f7 f8  | --- |  |
| `fdiv` | 0x00 | 0xF4 | REG | 4 | =f7 f8  | --- |  |
| `fcmp` | 0x00 | 0xF5 | REG | 4 | =f7 f8  | =flags  |  |
| `fsqrt` | 0x00 | 0xF6 | REG | 4 | =f7 f8  | --- |  |
| `fabs` | 0x00 | 0xF7 | REG | 4 | =f7 f8  | --- |  |
| `fneg` | 0x00 | 0xF8 | REG | 4 | =f7 f8  | --- |  |
| `fcvt` | 0x00 | 0xF9 | REG | 4 | =r8 f7  | --- |  |
| `fmowi` | 0x00 | 0xFA | INMED | 11 | =f5  | --- |  |
| `fload` | 0x00 | 0xFB | REG | 4 | =f7 r8  | --- |  |
| `fstore` | 0x00 | 0xFC | REG | 4 | r8 f7  | --- |  |
| `callm` | 0x00 | 0xFD | REG | 4 | =r7 r8  | =flags =pila marco =pc ? | salta |
| `proceed` | 0x00 | 0xFE | NONE | 2 | --- | =pila marco =pc  | salta |
