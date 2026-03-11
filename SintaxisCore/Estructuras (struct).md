## Forma 1
Declaración con ``struct`` explícito (Clásica)
```c
struct Punto {
    uint8_t x, y;
};

struct Punto p1 = {1, 2};  // Siempre struct Punto
```

## Forma 2
``typedef`` tradicional (Más común)
```c
typedef struct {
    uint8_t x, y;
} Punto;

Punto p1 = {1, 2};  // Sin "struct"
```

## Forma 3
``typedef`` con nombre de tag
```c
typedef struct Punto {
    uint8_t x, y;
} Punto;

Punto p1 = {1, 2};
struct Punto p1 = {1, 2};  // Ambas formas
```

## Forma 4
Declaración ``inline`` en variable
```c
typedef struct { uint8_t x, y; } Punto;
```

## Forma 5
``struct`` anidado con ``typedef``
```c
typedef struct {
    uint8_t x, y;
    struct {  // Anidado
        float z;
    } coord3d;
} Punto3D;

Punto3D p = {{1, 2}, {3.14f}};
```

## Forma 6
```c
typedef struct Punto {
    uint8_t x, y;
} Punto;

Punto sumar(Punto a, Punto b) {
    Punto resultado = {a.x + b.x, a.y + b.y};
    return resultado;
}
```

----
También se podrá usar ``struct`` al estilo C++, permitiendo crear clases ligeras sin heredar de ``Object``:
```c++
struct Rectangulo {
    Punto esq_sup, esq_inf;
    
    Rectangulo(Punto sup, Punto inf) 
        : esq_sup(sup), esq_inf(inf) {}
    
    double area() const {
        return (esq_inf.x - esq_sup.x) * 
               (esq_inf.y - esq_sup.y);
    }
    
    bool contiene(const Punto& p) const {
        return p.x >= esq_sup.x && p.x <= esq_inf.x &&
               p.y >= esq_sup.y && p.y <= esq_inf.y;
    }
};
```