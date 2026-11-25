# Bitácora de Requerimientos y Auditoría IA — Sesión 18
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arreglos y Verificación de Límites  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Cálculo de tamaño de arreglo en memoria  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión me salió sin errores. Reutilicé la tabla de anchos de la sesión 12 en una función auxiliar `ancho`, y con eso `bytes_arreglo` es solo multiplicar: `int[10]` son 4 × 10 = 40 bytes.
>
> Lo que tuve que pensar fue la diferencia entre negativo y cero: un arreglo con elementos negativos no tiene sentido y devuelve -1, pero uno de cero elementos sí es válido y pesa 0 bytes. Por eso la condición es `elementos < 0` y no `<= 0`. También entendí que el `(long)` antes de multiplicar sirve para que un arreglo muy grande no se desborde.
>
> En `indice_en_rango` lo clave fue que el índice más alto es `elementos - 1`: en un arreglo de 5 los índices van del 0 al 4, y el 5 ya está fuera. Por eso uso `indice < elementos` y no `<=`. Además aprendí que en C una comparación ya devuelve 1 o 0, así que no necesito un `if`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Indexación entera no negativa |
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
Implementa exactamente estas declaraciones de api_sesion18.h, sin alterarlas:

long bytes_arreglo(const char *tipo, int elementos);
int indice_en_rango(int indice, int elementos);

Reglas del contrato:
- Tipos válidos: char, short, int, float, double y puntero; otro nombre devuelve -1.
- Un número negativo de elementos es inválido y devuelve -1; cero elementos pesa cero bytes.
- El índice válido más alto es elementos - 1, nunca elementos.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion18.h"
#include <string.h>

static int ancho(const char *tipo) {
    if (tipo == NULL) return -1;
    if (strcmp(tipo, "char") == 0)    return 1;
    if (strcmp(tipo, "short") == 0)   return 2;
    if (strcmp(tipo, "int") == 0)     return 4;
    if (strcmp(tipo, "float") == 0)   return 4;
    if (strcmp(tipo, "double") == 0)  return 8;
    if (strcmp(tipo, "puntero") == 0) return 8;
    return -1;
}

long bytes_arreglo(const char *tipo, int elementos) {
    int w = ancho(tipo);
    if (w < 0 || elementos < 0) return -1;
    return (long)w * elementos;
}

int indice_en_rango(int indice, int elementos) {
    return indice >= 0 && indice < elementos;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo por los `strcmp` del tipo y O(1) en memoria. `indice_en_rango` es O(1).

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Límites estáticos y dimensiones inválidas

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 18 - Tamano de arreglo y limites ===
Caso                               esperado:              obtenido:              resultado
[ 1] int[10]                            esperado: 40           obtenido: 40           OK
[ 2] double[3]                          esperado: 24           obtenido: 24           OK
[ 3] int[-4]                            esperado: -1           obtenido: -1           OK
[ 4] indice 0 de 5                      esperado: 1            obtenido: 1            OK
[ 5] indice 5 de 5                      esperado: 0            obtenido: 0            OK
[ 6] indice negativo                    esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion18.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 18 - Tamano de arreglo y limites ===
Caso                               esperado:              obtenido:              resultado
[ 1] int[10]                            esperado: 40           obtenido: 40           OK
[ 2] double[3]                          esperado: 24           obtenido: 24           OK
[ 3] int[-4]                            esperado: -1           obtenido: -1           OK
[ 4] indice 0 de 5                      esperado: 1            obtenido: 1            OK
[ 5] indice 5 de 5                      esperado: 0            obtenido: 0            OK
[ 6] indice negativo                    esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion18.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

Esta vez la IA salió bien. Hace la multiplicación en `long` para que `int[grande]` no desborde, rechaza dimensiones negativas y la comparación `indice >= 0 && indice < elementos` no tiene sumas que puedan desbordar. Los seis casos pasan y mis pruebas extra también.

### Hallazgo 1: Lo que probé y salió bien

Arreglo gigante, índice en el límite y dimensión negativa en `indice_en_rango`:

```text
double[INT_MAX] -> 17179869176 bytes
indice INT_MAX de INT_MAX -> 0
indice 0 de -3 -> 0
bytes_arreglo("int*",4) -> -1
```

### Hallazgo 2: Riesgo de portabilidad: `long` de 32 bits

El cálculo depende de que `long` sea de 64 bits, que es cierto en Linux (Colab) pero no en Windows, donde `long` mide 32 bits. Ahí `double[INT_MAX]` daría un número basura. Simulé lo que vería un `long` de 32 bits:

```text
int32 de esos bytes (lo que veria un long de 32 bits) -> -8
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Lo que probé y salió bien** — nada que corregir en Colab.

```text
double[INT_MAX] -> 17179869176 bytes
indice INT_MAX de INT_MAX -> 0
indice 0 de -3 -> 0
```

**Riesgo de portabilidad: `long` de 32 bits** — uso `__builtin_mul_overflow` para que, si el producto no cabe en `long`, devuelva -1 en cualquier plataforma.

```text
(misma salida en Colab; en un long de 32 bits ahora devolveria -1)
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion18.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Lo que probé y salió bien** | **FALLA** | Pasa | No |
| **Riesgo de portabilidad: `long` de 32 bits** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(L) en tiempo por los `strcmp` del tipo y O(1) en memoria. `indice_en_rango` es O(1). Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Lo que probé y salió bien.** *Lo que hice:* nada que corregir en Colab.
>
> **2. Riesgo de portabilidad: `long` de 32 bits.** *Lo que hice:* uso `__builtin_mul_overflow` para que, si el producto no cabe en `long`, devuelva -1 en cualquier plataforma.
>
> **Lo que me llevo.** Aquí no encontré fallas en Colab, y eso también es un resultado: lo anoto en lugar de inventar un problema. Lo único que corregí es una suposición sobre el tamaño de `long` que no se cumple en Windows. Para un compilador que debería poder compilarse en cualquier lado, vale la pena no depender de eso.

---
**Firma digital:** pablouriellopez68-sketch