# Bitácora de Requerimientos y Auditoría IA — Sesión 13
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Manejo de Ámbitos (Scope Global/Local)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Pila de tablas de símbolos (Stack Scopes)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion13_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Ocultamiento de variables (Shadowing) |
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
Implementa exactamente estas declaraciones de api_sesion13.h, sin alterarlas:

#define AMBITO_MAX 8
typedef struct Ambito {
    const char *nombre;
    const char *nombres[AMBITO_MAX];
    int total;
    struct Ambito *padre;
} Ambito;

int declarar(Ambito *ambito, const char *nombre);
const char *buscar(const Ambito *ambito, const char *nombre);

Reglas del contrato:
- declarar() no consulta los ámbitos padre: el mismo nombre puede existir una vez en cada nivel.
- declarar() devuelve 1 si insertó, 0 si el nombre ya estaba en ese ámbito y -1 si el ámbito está lleno o recibe NULL o un nombre vacío.
- buscar() devuelve el nombre del ámbito más cercano, no el símbolo encontrado.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion13.h"
#include <string.h>

int declarar(Ambito *ambito, const char *nombre) {
    if (ambito == NULL || nombre == NULL || nombre[0] == '\0') return -1;
    for (int i = 0; i < ambito->total; i++)
        if (strcmp(ambito->nombres[i], nombre) == 0) return 0;
    if (ambito->total >= AMBITO_MAX) return -1;
    ambito->nombres[ambito->total++] = nombre;
    return 1;
}

const char *buscar(const Ambito *ambito, const char *nombre) {
    if (nombre == NULL) return NULL;
    for (const Ambito *a = ambito; a != NULL; a = a->padre)
        for (int i = 0; i < a->total; i++)
            if (strcmp(a->nombres[i], nombre) == 0) return a->nombre;
    return NULL;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** `declarar` es O(m) con m los nombres del ámbito (máximo 8). `buscar` es O(d·m), con d la profundidad de la cadena de ámbitos. Memoria O(1) en la versión de la IA.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Búsqueda jerárquica de identificadores

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion13_c.c artesano_sesion13_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 13 - Pila de ambitos y busqueda jerarquica ===
Caso                               esperado:              obtenido:              resultado
[ 1] declarar(&g, "x")                  esperado: 1            obtenido: 1            OK
[ 2] buscar en el mismo ambito          esperado: global       obtenido: global       OK
[ 3] buscar subiendo al padre           esperado: global       obtenido: global       OK
[ 4] shadowing en el hijo               esperado: bloque       obtenido: bloque       OK
[ 5] tres niveles hasta la raiz         esperado: raiz         obtenido: raiz         OK
[ 6] nombre no declarado                esperado: (nulo)       obtenido: (nulo)       OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion13.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA respetó que `declarar` no mira a los padres y que `buscar` devuelve el nombre del ámbito, no del símbolo. Los seis casos pasan, incluido el shadowing. Lo que encontré tiene que ver con de quién es la memoria del nombre.

### Hallazgo 1: Guarda el puntero, no una copia del nombre

`declarar` hace `nombres[total++] = nombre`. Si quien llama reutiliza su buffer (que es justo lo que hace un analizador léxico que lee token por token), el nombre guardado cambia por debajo. Declaré `x` y luego `y` usando el mismo buffer:

```text
declarar con el mismo buffer: "x" y luego "y" -> segundo devuelve 0
buscar x -> (nulo)
contenido: total=1 nombres[0]="y"
```

### Hallazgo 2: El límite de 8 sí se respeta

Lo probé porque es lo primero que suele fallar: la novena declaración devuelve -1 y no escribe fuera del arreglo. Aquí la IA lo hizo bien.

```text
1 1 1 1 1 1 1 1 -1  <- 9 declaraciones con AMBITO_MAX=8
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Guarda el puntero, no una copia del nombre** — `declarar` hace su propia copia con `malloc`. El costo es que el contrato no trae función para liberar el ámbito, así que esa memoria queda viva hasta que termina el programa; en el compilador del equipo habría que agregar un `liberar_ambito`.

```text
declarar con el mismo buffer: "x" y luego "y" -> segundo devuelve 1
buscar x -> global
contenido: total=2 nombres[0]="x"
```

**El límite de 8 sí se respeta** — nada que corregir; además agregué un tope de pasos en `buscar` por si la cadena de padres tuviera un ciclo.

```text
1 1 1 1 1 1 1 1 -1  <- 9 declaraciones con AMBITO_MAX=8
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion13.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Guarda el puntero, no una copia del nombre** | **FALLA** | Pasa | No |
| El límite de 8 sí se respeta | Pasa | Pasa | No |

**Complejidad verificada:** `declarar` es O(m) con m los nombres del ámbito (máximo 8). `buscar` es O(d·m), con d la profundidad de la cadena de ámbitos. Memoria O(1) en la versión de la IA. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Guarda el puntero, no una copia del nombre.** *Lo que hice:* `declarar` hace su propia copia con `malloc`. El costo es que el contrato no trae función para liberar el ámbito, así que esa memoria queda viva hasta que termina el programa; en el compilador del equipo habría que agregar un `liberar_ambito`.
>
> **2. El límite de 8 sí se respeta.** *Lo que hice:* nada que corregir; además agregué un tope de pasos en `buscar` por si la cadena de padres tuviera un ciclo.
>
> **Lo que me llevo.** El error no se ve con las pruebas porque el harness siempre pasa cadenas literales, que nunca cambian. En un compilador real los nombres vienen de un buffer que se reusa, y ahí el ámbito "olvida" sus variables sin que nadie lo note. Me quedó claro que en C cada vez que guardas un `const char *` tienes que preguntarte quién es dueño de esa memoria y cuánto tiempo va a vivir.

---
**Firma digital:** pablouriellopez68-sketch