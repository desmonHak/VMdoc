**Offset**           = ``0xFFF`` (`12bits`) \[`12bits`] 
Page Table    = ``0xFFF`` (`12bits`) \[`24bits`]
Page Table1  = ``0xFFFF`` (`16bits`) \[`40bits`]
Page Table2  = ``0xFFFFFF`` (`24bits`) \[`64bits`]

```
64 bits: [24][16][12][12] = 64 bits
0xFFFFFF | 0xFFFF | 0xFFF | offset(4KB)
   PT2      PT1      PT      offset
```

| Nivel      | Bits   | Máscara    | Entradas | Tamaño tabla | Shift  |
| ---------- | ------ | ---------- | -------- | ------------ | ------ |
| **PT2**    | **24** | `0xFFFFFF` | **16M**  | **128MB**    | **48** |
| **PT1**    | **16** | `0xFFFF`   | **64K**  | **512KB**    | **32** |
| **PT**     | **12** | `0xFFF`    | **4K**   | **32KB**     | **12** |
| **Offset** | **12** | `0xFFF`    | -        | **4KB**      | **0**  |

|  0xFFFFFF   |   0xFFFF    |   0xFFF    | 0xFFF  |
| :---------: | :---------: | :--------: | :----: |
| Page Table2 | Page Table1 | Page Table | Offset |
