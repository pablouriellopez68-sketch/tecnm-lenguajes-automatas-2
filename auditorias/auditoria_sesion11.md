# Bitácora de Requerimientos y Auditoría IA — Sesión 11
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arquitectura de la Tabla de Símbolos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla Hash con encadenamiento  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion11_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Función de dispersión y colisiones O(1) |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[CONTEXTO]
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Cualquier advertencia cuenta como error. Solo biblioteca estándar.
El archivo se compila junto con un harness que ya trae main(), así que
no debe incluir main().

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente estas declaraciones de api_sesion11.h, sin alterarlas:

typedef struct Simbolo {
    char nombre[16];
    int tipo;
    int offset;
    struct Simbolo *sig;
} Simbolo;

unsigned int hash_djb2(const char *clave);
int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre, int tipo, int offset);

Reglas del contrato:
- Ningún nombre se duplica: si ya existe en cualquier cubeta, insertar devuelve 0 y no crea nodo.
- Una colisión se encadena; jamás se sobrescribe una entrada existente.
- Con tabla NULL, nombre NULL o cero cubetas, insertar devuelve -1.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion11.h"
#include <stdlib.h>
#include <string.h>

unsigned int hash_djb2(const char *clave) {
    unsigned int h = 5381;
    int c;
    while ((c = (unsigned char)*clave++) != 0)
        h = h * 33 + (unsigned int)c;
    return h;
}

int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre, int tipo, int offset) {
    if (tabla == NULL || nombre == NULL || cubetas == 0) return -1;

    for (unsigned int i = 0; i < cubetas; i++)
        for (Simbolo *s = tabla[i]; s != NULL; s = s->sig)
            if (strcmp(s->nombre, nombre) == 0) return 0;

    Simbolo *nuevo = malloc(sizeof *nuevo);
    if (nuevo == NULL) return -1;
    strncpy(nuevo->nombre, nombre, sizeof nuevo->nombre - 1);
    nuevo->nombre[sizeof nuevo->nombre - 1] = '\0';
    nuevo->tipo = tipo;
    nuevo->offset = offset;

    unsigned int idx = hash_djb2(nombre) % cubetas;
    nuevo->sig = tabla[idx];
    tabla[idx] = nuevo;
    return 1;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** `hash_djb2` es O(k) con k la longitud de la clave. `insertar_simbolo` es O(N + k), con N el total de símbolos, porque el contrato pide revisar duplicados en *todas* las cubetas; si solo revisara la cubeta del hash sería O(1) promedio. Memoria: un nodo por símbolo.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Manejo de claves duplicadas y colisiones

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion11_c.c artesano_sesion11_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 11 - Tabla hash con encadenamiento ===
Caso                               esperado:              obtenido:              resultado
[ 1] hash_djb2("")                      esperado: 5381         obtenido: 5381         OK
[ 2] hash_djb2("abc")                   esperado: 193485963    obtenido: 193485963    OK
[ 3] insertar x dos veces               esperado: 0            obtenido: 0            OK
[ 4] dos claves, una cubeta             esperado: 2            obtenido: 2            OK
[ 5] offset de beta en cadena           esperado: 4            obtenido: 4            OK
[ 6] insertar en tabla NULL             esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion11.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

El hash djb2 está bien (`h * 33 + c` empezando en 5381) y las colisiones se encadenan sin sobrescribir. Los seis casos pasan. El harness no libera los nodos, así que con AddressSanitizer aparecen fugas de 160 bytes, pero esas son del harness porque el contrato no da función para liberar. Lo que sí es de la IA:

### Hallazgo 1: Nombres largos se truncan y dejan de ser únicos

El campo `nombre` es de 16 bytes y la IA usa `strncpy` para cortar. El problema es que después compara el nombre *completo* contra el *cortado*, así que nunca coinciden: dos variables distintas quedan guardadas con el mismo nombre, y la misma variable se puede insertar dos veces.

```text
insertar contador_principal_1 -> 1
insertar contador_principal_2 -> 1
  cubeta 0: "contador_princi" offset 0
  cubeta 1: "contador_princi" offset 4
reinsertar contador_principal_1 -> 1 (deberia ser 0)
```

### Hallazgo 2: `hash_djb2(NULL)` lee la dirección cero

La función pública no revisa su argumento:

```text
ia_sesion11_c.c:13:32: runtime error: load of null pointer of type 'const char'
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Nombres largos se truncan y dejan de ser únicos** — si el nombre no cabe en 15 caracteres, `insertar_simbolo` lo rechaza con -1 en vez de truncarlo.

```text
insertar contador_principal_1 -> -1
insertar contador_principal_2 -> -1
```

**`hash_djb2(NULL)` lee la dirección cero** — con NULL devuelve el valor inicial 5381, igual que la cadena vacía.

```text
hash_djb2(NULL)...
5381
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion11.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Nombres largos se truncan y dejan de ser únicos** | **FALLA** | Pasa | No |
| **`hash_djb2(NULL)` lee la dirección cero** | **CRASH** | Pasa | No |

**Complejidad verificada:** `hash_djb2` es O(k) con k la longitud de la clave. `insertar_simbolo` es O(N + k), con N el total de símbolos, porque el contrato pide revisar duplicados en *todas* las cubetas; si solo revisara la cubeta del hash sería O(1) promedio. Memoria: un nodo por símbolo. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Nombres largos se truncan y dejan de ser únicos.** *Lo que hice:* si el nombre no cabe en 15 caracteres, `insertar_simbolo` lo rechaza con -1 en vez de truncarlo.
>
> **2. `hash_djb2(NULL)` lee la dirección cero.** *Lo que hice:* con NULL devuelve el valor inicial 5381, igual que la cadena vacía.
>
> **Lo que me llevo.** El truncado fue el que más me sorprendió porque `strncpy` se ve como la opción "segura". Sí evita escribir fuera del arreglo, pero cambia el dato, y una tabla de símbolos cuyo trabajo es justamente que no haya dos nombres iguales termina teniendo dos iguales. Aprendí que cortar un dato para que quepa no es manejar el error, es esconderlo.

---
**Firma digital:** pablouriellopez68-sketch