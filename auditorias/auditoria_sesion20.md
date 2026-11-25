# Bitácora de Requerimientos y Auditoría IA — Sesión 20
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 2: Analizador Semántico y Scopes  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Integración de Tabla de Símbolos y AST  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> `profundidad_de` es como `buscar` de la sesión 13, pero en lugar del nombre del ámbito devuelve cuántos niveles subí. Me equivoqué al contar: puse `nivel++` en el `for` y otra vez dentro del ciclo, así que los niveles salían de 2 en 2. Luego, al quitarlo, dejé `nivel+` con un solo `+`, que no compila. Al final dejé un solo `nivel++`, dentro del ciclo.
>
> Lo que más me costó fue `contar_visibles`, porque la misma variable puede estar en dos ámbitos y solo debe contarse una vez. La idea que usé es reutilizar `profundidad_de`: un nombre se ve desde el ámbito actual solo si `profundidad_de` lo encuentra en el mismo nivel donde está. Si una `x` está en el nivel 2 pero `profundidad_de` la encuentra en el 0, es porque otra `x` más adentro la tapa, y no se cuenta.
>
> Primero comparé contra `0` en lugar de `nivel`, y así contaba la `x` dos veces y se olvidaba de las demás. Después cambié por error `cuenta++` por `nivel++`. La línea correcta es `if (profundidad_de(actual, a->simbolos[i]) == nivel) cuenta++;`. También se me olvidó la última llave de la función. Mi versión quedó más corta que la de la IA, que usa tres ciclos anidados para lo mismo.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Contrato de Validación Semántica Integral |
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
Implementa exactamente estas declaraciones de api_sesion20.h, sin alterarlas:

#define AMBITO_MAX 8
typedef struct Ambito {
    const char *nombre;
    const char *simbolos[AMBITO_MAX];
    int total;
    struct Ambito *padre;
} Ambito;

int profundidad_de(const Ambito *actual, const char *nombre);
int contar_visibles(const Ambito *actual);

Reglas del contrato:
- profundidad_de() devuelve -1 si el nombre no existe y -2 si algún argumento es NULL.
- contar_visibles() cuenta cada nombre una sola vez, aunque esté en varios ámbitos.
- El ámbito más interno siempre gana: el shadowing no se promedia ni se suma.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion20.h"
#include <string.h>

int profundidad_de(const Ambito *actual, const char *nombre) {
    if (actual == NULL || nombre == NULL) return -2;
    int nivel = 0;
    for (const Ambito *a = actual; a != NULL; a = a->padre, nivel++)
        for (int i = 0; i < a->total; i++)
            if (strcmp(a->simbolos[i], nombre) == 0) return nivel;
    return -1;
}

int contar_visibles(const Ambito *actual) {
    if (actual == NULL) return -2;
    int cuenta = 0;
    for (const Ambito *a = actual; a != NULL; a = a->padre) {
        for (int i = 0; i < a->total; i++) {
            const char *n = a->simbolos[i];
            int repetido = 0;
            /* ¿ya apareció en un ámbito más interno (o antes en este)? */
            for (const Ambito *b = actual; b != NULL && !repetido; b = b->padre) {
                int lim = (b == a) ? i : b->total;
                for (int j = 0; j < lim; j++)
                    if (strcmp(b->simbolos[j], n) == 0) { repetido = 1; break; }
                if (b == a) break;
            }
            if (!repetido) cuenta++;
        }
    }
    return cuenta;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** `profundidad_de` es O(d·m) con d los ámbitos de la cadena y m los símbolos por ámbito (máximo 8). `contar_visibles` es O((d·m)²) porque cada nombre se compara contra los anteriores para no contarlo dos veces. Memoria O(1).

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Casos complejos de shadowing y ámbito

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 20 - Resolucion de nombres y visibilidad ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre en el ambito actual         esperado: 0            obtenido: 0            OK
[ 2] nombre dos niveles arriba          esperado: 2            obtenido: 2            OK
[ 3] shadowing gana el interno          esperado: 0            obtenido: 0            OK
[ 4] nombre no visible                  esperado: -1           obtenido: -1           OK
[ 5] visibles sin repetir               esperado: 3            obtenido: 3            OK
[ 6] ambito NULL                        esperado: -2           obtenido: -2           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion20.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 20 - Resolucion de nombres y visibilidad ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre en el ambito actual         esperado: 0            obtenido: 0            OK
[ 2] nombre dos niveles arriba          esperado: 2            obtenido: 2            OK
[ 3] shadowing gana el interno          esperado: 0            obtenido: 0            OK
[ 4] nombre no visible                  esperado: -1           obtenido: -1           OK
[ 5] visibles sin repetir               esperado: 3            obtenido: 3            OK
[ 6] ambito NULL                        esperado: -2           obtenido: -2           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion20.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA resolvió bien lo difícil del Hito 2: `profundidad_de` se queda con el ámbito más interno (shadowing) y `contar_visibles` no cuenta dos veces el mismo nombre. Lo verifiqué con tres niveles que repiten `x`. Los seis casos pasan. Lo que no maneja es una lista de símbolos con huecos.

### Hallazgo 1: El conteo con shadowing sí sale bien

Tres ámbitos (`global` con x,y; `funcion` con x,z; `bloque` con x): debe dar 3.

```text
visibles (x,y,z) -> 3
```

### Hallazgo 2: Un símbolo NULL dentro de la lista truena `strcmp`

Si un ámbito tiene `total = 2` pero la segunda posición está en NULL (por ejemplo, un símbolo que se borró), la búsqueda le pasa NULL a `strcmp`:

```text
ia_sesion20_c.c:14:17: runtime error: null pointer passed as argument 1, which is declared to never be null
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**El conteo con shadowing sí sale bien** — nada que corregir.

```text
visibles (x,y,z) -> 3
```

**Un símbolo NULL dentro de la lista truena `strcmp`** — salto posiciones NULL y además recorto `total` a `AMBITO_MAX` por si viniera corrupto, para no leer fuera del arreglo.

```text
simbolo NULL en la lista...
-1
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion20.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| El conteo con shadowing sí sale bien | Pasa | Pasa | No |
| **Un símbolo NULL dentro de la lista truena `strcmp`** | **CRASH** | Pasa | No |

**Complejidad verificada:** `profundidad_de` es O(d·m) con d los ámbitos de la cadena y m los símbolos por ámbito (máximo 8). `contar_visibles` es O((d·m)²) porque cada nombre se compara contra los anteriores para no contarlo dos veces. Memoria O(1). Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. El conteo con shadowing sí sale bien.** *Lo que hice:* nada que corregir.
>
> **2. Un símbolo NULL dentro de la lista truena `strcmp`.** *Lo que hice:* salto posiciones NULL y además recorto `total` a `AMBITO_MAX` por si viniera corrupto, para no leer fuera del arreglo.
>
> **Lo que me llevo.** La lógica de ámbitos de la IA está bien; donde falla es en confiar ciegamente en los datos de la estructura (`total` y cada puntero). En el Hito 2 del equipo los ámbitos se llenan desde el parser, y si algún día se agrega borrar símbolos, una entrada en NULL es muy posible. Me quedo con que una función de consulta debe aguantar datos raros sin tronar, porque se llama muchísimas veces y desde muchos lugares.

---
**Firma digital:** pablouriellopez68-sketch