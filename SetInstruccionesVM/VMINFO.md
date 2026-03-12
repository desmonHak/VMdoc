Permite obtener información relevante de la instancia virtual actual

| instrucción | opcode1 | extensión de instrucción | total bytes |
| :---------: | :-----: | :----------------------: | :---------: |
| ``VmInfo``  |   0x1   |           reg            |      2      |
```c
typedef struct VmInstanceInfoData {
	uint8_t  version[2];        // version.subversion
	uint64_t total_memory;      // cantidad de bytes reservados para la instancia
	uint32_t id;                // id de esta instancia.
	
	uint64_t timestamp_ms;      // fecha de creacion de la instancia
	
	uint64_t code_address; // direccion base del codigo
	uint64_t code_limit;  // direccion limite del codigo
	
	uint64_t data_address; // direccion base de los datos
	uint64_t data_limit;  // direccion limite del codigo
	
	// permite saber los permisos para ejecutar ciertas instrucciones
	union {
		struct {
			uint8_t HLT: 1; // permiso para ejecutar la instruccion HLT
			uint8_t VmInfoManager: 1; // permiso para obtener informacion
			uint8_t VmInfo: 1; // instuccion para ver informacion de la VM
			uint8_t EDM_EDMW: 1; // permite modo distribuido
			uint8_t ENC: 1; // Permiso para Exec Native Code
		} flags;
		uint64_t raw;
	} permissions;
	
	uint16_t port;
	union {
		struct {  
		    uint32_t ipv4; // htonl(192168110)  
		} HostIpv4;  
		struct {  
		    uint8_t  bytes[16];  // IPv6 raw
		} HostIpv6;
	} client_ip;
	
	
	uint32_t entry_point;        // Primera instrucción PC
    uint32_t priority;           // 0=normal, 255=critica?
    uint64_t pid;                // PID sistema operativo
} VmInstanceInfoData;
```