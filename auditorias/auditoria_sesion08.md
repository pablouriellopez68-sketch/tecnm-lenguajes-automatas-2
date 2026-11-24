# Bitácora de Requerimientos y Auditoría IA — Sesión 08
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Inferencia y Promoción de Tipos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Reglas de casteo implícito  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion08_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Promoción entera y ensanchamiento |
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
Implementa exactamente estas declaraciones de api_sesion08.h, sin alterarlas:

typedef enum { TIPO_ERROR = 0, TIPO_CHAR = 1, TIPO_INT = 2, TIPO_FLOAT = 3 } Tipo;

Tipo promover(Tipo a, Tipo b);
int hay_perdida(Tipo destino, Tipo origen);
const char *nombre_tipo(Tipo t);

Reglas del contrato:
- Jerarquía: CHAR < INT < FLOAT.
- promover devuelve el tipo más ancho; si uno es ERROR, devuelve ERROR.
- hay_perdida es 1 cuando el destino es más angosto que el origen.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion08.h"

static int rango(Tipo t) {
    switch (t) {
        case TIPO_CHAR:  return 1;
        case TIPO_INT:   return 2;
        case TIPO_FLOAT: return 3;
        default:         return 0;
    }
}

Tipo promover(Tipo a, Tipo b) {
    if (a == TIPO_ERROR || b == TIPO_ERROR) return TIPO_ERROR;
    return rango(a) >= rango(b) ? a : b;
}

int hay_perdida(Tipo destino, Tipo origen) {
    return rango(destino) < rango(origen);
}

const char *nombre_tipo(Tipo t) {
    switch (t) {
        case TIPO_CHAR:  return "char";
        case TIPO_INT:   return "int";
        case TIPO_FLOAT: return "float";
        case TIPO_ERROR: return "error";
    }
    return "desconocido";
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(1) en tiempo y en memoria: `rango()` es un `switch` y las tres funciones hacen un número fijo de comparaciones.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Mezcla de tipos enteros y flotantes

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion08_c.c artesano_sesion08_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 08 - Reglas de promocion y perdida ===
Caso                               esperado:              obtenido:              resultado
[ 1] promover(CHAR,INT)                 esperado: int          obtenido: int          OK
[ 2] promover(INT,FLOAT)                esperado: float        obtenido: float        OK
[ 3] promover(FLOAT,FLOAT)              esperado: float        obtenido: float        OK
[ 4] promover(ERROR,INT)                esperado: error        obtenido: error        OK
[ 5] hay_perdida(CHAR,INT)              esperado: 1            obtenido: 1            OK
[ 6] hay_perdida(INT,CHAR)              esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion08.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA resolvió el contrato con una función auxiliar `rango()` que le da un número a cada tipo y compara esos números. Pasa los seis casos. El problema es todo lo que pasa con valores que no son exactamente CHAR, INT o FLOAT, y con una pérdida que la jerarquía no ve.

### Hallazgo 1: Un tipo inválido se cuela como si fuera válido

`rango()` devuelve 0 para cualquier valor fuera del enum, y `promover` solo revisa `TIPO_ERROR` de forma explícita. Entonces un `Tipo` corrupto (por ejemplo `(Tipo)9`) pierde la comparación y la función devuelve el otro tipo, como si todo estuviera bien.

```text
promover((Tipo)9, CHAR)        -> char
promover(CHAR, (Tipo)-1)       -> char
```

### Hallazgo 2: `hay_perdida` trata a ERROR como el tipo más angosto

Como ERROR vale 0 en el rango, `hay_perdida(INT, ERROR)` dice que no hay pérdida, o sea que asignar algo con tipo de error a un `int` es seguro. No lo es: si el origen ya es un error, no hay nada que garantizar.

```text
hay_perdida(ERROR, INT)        -> 1
hay_perdida(INT, ERROR)        -> 0
```

### Hallazgo 3: INT → FLOAT sí pierde, aunque FLOAT esté más arriba

La jerarquía CHAR < INT < FLOAT dice que pasar un `int` a `float` es ensanchar. Pero un `float` de 32 bits solo guarda 24 bits de mantisa, así que enteros grandes se redondean. Lo comprobé con 16777217:

```text
hay_perdida(FLOAT, INT)        -> 0
int 16777217 -> float -> int  = 16777216
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Un tipo inválido se cuela como si fuera válido** — agregué `valido()` y `promover` devuelve `TIPO_ERROR` si cualquiera de los dos no es CHAR, INT o FLOAT.

```text
promover((Tipo)9, CHAR)        -> error
promover(CHAR, (Tipo)-1)       -> error
```

**`hay_perdida` trata a ERROR como el tipo más angosto** — si destino u origen no son válidos, `hay_perdida` devuelve 1.

```text
hay_perdida(ERROR, INT)        -> 1
hay_perdida(INT, ERROR)        -> 1
```

**INT → FLOAT sí pierde, aunque FLOAT esté más arriba** — marqué INT → FLOAT como conversión con posible pérdida. Ojo: el harness no prueba este caso, así que no choca con los seis casos, pero sí es una decisión de diseño que vale la pena platicar con el profe.

```text
hay_perdida(FLOAT, INT)        -> 1
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion08.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Un tipo inválido se cuela como si fuera válido** | **FALLA** | Pasa | No |
| **`hay_perdida` trata a ERROR como el tipo más angosto** | **FALLA** | Pasa | No |
| **INT → FLOAT sí pierde, aunque FLOAT esté más arriba** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(1) en tiempo y en memoria: `rango()` es un `switch` y las tres funciones hacen un número fijo de comparaciones. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Un tipo inválido se cuela como si fuera válido.** *Lo que hice:* agregué `valido()` y `promover` devuelve `TIPO_ERROR` si cualquiera de los dos no es CHAR, INT o FLOAT.
>
> **2. `hay_perdida` trata a ERROR como el tipo más angosto.** *Lo que hice:* si destino u origen no son válidos, `hay_perdida` devuelve 1.
>
> **3. INT → FLOAT sí pierde, aunque FLOAT esté más arriba.** *Lo que hice:* marqué INT → FLOAT como conversión con posible pérdida. Ojo: el harness no prueba este caso, así que no choca con los seis casos, pero sí es una decisión de diseño que vale la pena platicar con el profe.
>
> **Lo que me llevo.** La IA siguió al pie de la letra la jerarquía del enunciado, pero la jerarquía describe *anchura*, no *exactitud*. Ensanchar no siempre es seguro (INT → FLOAT) y un valor fuera del enum no debe tratarse como el más chico. Lo que aprendí es que una tabla de rangos necesita también un caso "no sé qué es esto", y que ese caso debe ser un error y no un cero que se pierde en la comparación.

---
**Firma digital:** pablouriellopez68-sketch