Permite obtener información general del manager de instancias de VM.

|    instrucción    | opcode1 | opcode2 | extensión de instrucción | extensión de instrucción | total bytes |
| :---------------: | :-----: | :-----: | :----------------------: | :----------------------: | :---------: |
| ``VmInfoManager`` |   0x0   |   0x8   |        0b00000000        |           reg            |      4      |

```c
typedef struct VmInfoManagerData {
	uint8_t version[2]; // version.subversion
	uint32_t active_instances; // cantidad de instancia de la manager de instancias
	uint64_t total_memory_used;  // cantidad de bytes usadas por todas las instancias
	
	uint64_t timestamp_ms;      // fecha de creacion del manager
	
	union {
		struct {
			uint8_t ipv4_or_6;
		} 
		uint16_t raw;
	} flags1;
	

	
} VmInfoManagerData;
```