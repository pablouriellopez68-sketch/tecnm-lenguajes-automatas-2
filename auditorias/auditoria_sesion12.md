# Bitácora de Requerimientos y Auditoría IA — Sesión 12
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Atributos de Símbolos y Memoria  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Estructura de registro de símbolos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> `ancho_de_tipo` fue una tabla con `strcmp`: char = 1, short = 2, int y float = 4, double y puntero = 8, y cualquier otro tipo devuelve -1. Me equivoqué en una cosa simple: se me olvidó el punto y coma después de `return 8` en la línea de `double`. Sin él, el compilador cree que la instrucción sigue en la línea de abajo.
>
> Lo que más me costó entender fue la alineación: cada tipo solo puede empezar en una posición que sea múltiplo de su tamaño. Si declaro `char a` y luego `int b`, el cursor queda en 1, pero el `int` no puede empezar ahí y tiene que brincar hasta 4. Por eso hay bytes de relleno que no se usan.
>
> La fórmula `(cursor + ancho - 1) / ancho * ancho` la entendí haciendo el ejemplo a mano: `(1 + 3) / 4 * 4 = 4`. Funciona porque la división entera de C corta los decimales, así que redondea hacia arriba al siguiente múltiplo.
>
> También me quedó claro qué guardar y qué devolver: en el registro va el offset ya alineado, y la función devuelve dónde empieza el siguiente símbolo, que es el offset más el ancho (`4 + 4 = 8`).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Cálculo de desplazamientos y alineación |
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
Implementa exactamente estas declaraciones de api_sesion12.h, sin alterarlas:

typedef struct Simbolo {
    const char *nombre;
    const char *tipo;
    int offset;
} Simbolo;

int ancho_de_tipo(const char *tipo);
int asignar_offset(Simbolo *simbolo, int cursor);

Reglas del contrato:
- Los tipos válidos son char, short, int, float, double y puntero; cualquier otro devuelve -1.
- El cursor nunca retrocede y el offset guardado siempre es múltiplo del ancho del tipo.
- Un registro NULL, un cursor negativo o un tipo desconocido se rechazan con -1.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion12.h"
#include <string.h>

int ancho_de_tipo(const char *tipo) {
    if (tipo == NULL) return -1;
    if (strcmp(tipo, "char") == 0)    return 1;
    if (strcmp(tipo, "short") == 0)   return 2;
    if (strcmp(tipo, "int") == 0)     return 4;
    if (strcmp(tipo, "float") == 0)   return 4;
    if (strcmp(tipo, "double") == 0)  return 8;
    if (strcmp(tipo, "puntero") == 0) return 8;
    return -1;
}

int asignar_offset(Simbolo *simbolo, int cursor) {
    if (simbolo == NULL || cursor < 0) return -1;
    int ancho = ancho_de_tipo(simbolo->tipo);
    if (ancho < 0) return -1;
    int alineado = (cursor + ancho - 1) / ancho * ancho;
    simbolo->offset = alineado;
    return alineado + ancho;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo por los `strcmp` contra los seis nombres de tipo (L es la longitud del nombre), y O(1) en memoria.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Alineación de tipos y tamaños en bytes

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 12 - Registro de simbolo y offsets alineados ===
Caso                               esperado:              obtenido:              resultado
[ 1] ancho_de_tipo("int")               esperado: 4            obtenido: 4            OK
[ 2] ancho_de_tipo("double")            esperado: 8            obtenido: 8            OK
[ 3] int en cursor 0                    esperado: 4            obtenido: 4            OK
[ 4] char e int: offset de b            esperado: 4            obtenido: 4            OK
[ 5] double desde cursor 4              esperado: 8            obtenido: 8            OK
[ 6] registro NULL                      esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion12.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 12 - Registro de simbolo y offsets alineados ===
Caso                               esperado:              obtenido:              resultado
[ 1] ancho_de_tipo("int")               esperado: 4            obtenido: 4            OK
[ 2] ancho_de_tipo("double")            esperado: 8            obtenido: 8            OK
[ 3] int en cursor 0                    esperado: 4            obtenido: 4            OK
[ 4] char e int: offset de b            esperado: 4            obtenido: 4            OK
[ 5] double desde cursor 4              esperado: 8            obtenido: 8            OK
[ 6] registro NULL                      esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion12.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA alinea con `(cursor + ancho - 1) / ancho * ancho`, que es la fórmula clásica, y los seis casos pasan. El riesgo está en esa misma fórmula cuando el cursor es grande.

### Hallazgo 1: La alineación desborda `int` cerca del límite

Con un cursor cercano a `INT_MAX`, la suma `cursor + ancho - 1` (y luego `alineado + ancho`) se sale del rango de `int`. Es comportamiento indefinido y UBSan lo detecta:

```text
ia_sesion12_c.c:24:28: runtime error: signed integer overflow: 2147483644 + 8 cannot be represented in type 'int'
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**La alineación desborda `int` cerca del límite** — calculo el relleno con `cursor % ancho` y reviso contra `INT_MAX` antes de sumar; si no cabe, devuelvo -1 y no toco el registro (por eso el offset se queda en -77, el valor de prueba).

```text
double en cursor INT_MAX-3 -> devuelve -1, offset -77
char en cursor INT_MAX -> devuelve -1, offset -77
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion12.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **La alineación desborda `int` cerca del límite** | **CRASH** | Pasa | No |

**Complejidad verificada:** O(L) en tiempo por los `strcmp` contra los seis nombres de tipo (L es la longitud del nombre), y O(1) en memoria. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. La alineación desborda `int` cerca del límite.** *Lo que hice:* calculo el relleno con `cursor % ancho` y reviso contra `INT_MAX` antes de sumar; si no cabe, devuelvo -1 y no toco el registro (por eso el offset se queda en -77, el valor de prueba).
>
> **Lo que me llevo.** El código de la IA es correcto para cualquier programa real, porque nadie declara 2 GB de variables locales. Aun así lo corregí porque un cursor negativo o gigante solo llega ahí si algo antes ya falló, y en ese caso prefiero un -1 claro a un offset negativo que luego se use para escribir en memoria. Lo que me llevo es que la fórmula de redondeo que todos usamos tiene una suma escondida que también puede desbordar.

---
**Firma digital:** pablouriellopez68-sketch