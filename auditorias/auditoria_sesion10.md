# Bitácora de Requerimientos y Auditoría IA — Sesión 10
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 1: Mini-Evaluador de Expresiones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Integración AST y Tipado Estático  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo primero fue entender por qué son tres funciones: `expr` maneja `+` y `-`, `termino` maneja `*` y `/`, y `primaria` lee un número o un paréntesis. Como `expr` le pide a `termino` cada lado, la multiplicación se resuelve antes que la suma, y así `2+3*4` da 14 y no 20.
>
> En `primaria` aprendí que el carácter `'7'` no es el número 7: hay que restarle `'0'`, y el número se arma dígito por dígito con `v * 10 + dígito`. También que dentro de un paréntesis hay que volver a llamar a `expr`, porque ahí puede venir una expresión completa.
>
> En `expr` me equivoqué al escribir la suma y la resta: mezclé un `if` con el operador `? :` y quedó `v = ? v + d : v - d`, que no es válido. Lo cambié por un `if` / `else` normal.
>


---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Contrato formal de Frontend Semántico |
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
Implementa exactamente estas declaraciones de api_sesion10.h, sin alterarlas:

long evaluar_expresion(const char *fuente, int *error);

Reglas del contrato:
- Sin variables ni menos unario: solo enteros y los operadores + - * /.
- Respeta precedencia y paréntesis.
- Completa las tres funciones marcadas; no cambies la firma de evaluar_expresion.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion10.h"
#include <stdio.h>
#include <string.h>

#include <ctype.h>

/* El analizador ya viene armado: completa las tres funciones marcadas. */
typedef struct { const char *s; int error; } Parser;

static void saltar(Parser *p) {
    while (*p->s == ' ') p->s++;
}

long expr(Parser *p);

long primaria(Parser *p) {
    saltar(p);
    if (*p->s == '(') {
        p->s++;
        long v = expr(p);
        saltar(p);
        if (*p->s != ')') { p->error = 2; return 0; }
        p->s++;
        return v;
    }
    if (isdigit((unsigned char)*p->s)) {
        long v = 0;
        while (isdigit((unsigned char)*p->s)) {
            v = v * 10 + (*p->s - '0');
            p->s++;
        }
        return v;
    }
    p->error = 2;
    return 0;
}

long termino(Parser *p) {
    long v = primaria(p);
    for (;;) {
        saltar(p);
        char op = *p->s;
        if (op != '*' && op != '/') break;
        p->s++;
        long d = primaria(p);
        if (op == '*') v *= d;
        else {
            if (d == 0) { if (p->error == 0) p->error = 1; return 0; }
            v /= d;
        }
    }
    return v;
}

long expr(Parser *p) {
    long v = termino(p);
    for (;;) {
        saltar(p);
        char op = *p->s;
        if (op != '+' && op != '-') break;
        p->s++;
        long d = termino(p);
        v = (op == '+') ? v + d : v - d;
    }
    return v;
}

long evaluar_expresion(const char *fuente, int *error) {
    static int sin_error;
    if (error == NULL) error = &sin_error;
    *error = 0;
    if (fuente == NULL) { *error = 2; return 0; }
    Parser p = { fuente, 0 };
    long v = expr(&p);
    saltar(&p);
    if (p.error == 0 && *p.s != '\0') p.error = 2;
    *error = p.error;
    return p.error != 0 ? 0 : v;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(n) en tiempo, donde n es la longitud de la cadena, porque cada carácter se lee una vez. En memoria es O(p), donde p es la profundidad máxima de paréntesis, por la pila de la recursión `expr → termino → primaria → expr`.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Precedencia, paréntesis, división entre cero y sintaxis inválida

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 10 - Mini-evaluador de expresiones (Hito 1) ===
Caso                               esperado:              obtenido:              resultado
[ 1] "2+3*4"                            esperado: 14           obtenido: 14           OK
[ 2] "(2+3)*4"                          esperado: 20           obtenido: 20           OK
[ 3] "10/0" -> error 1                  esperado: 1            obtenido: 1            OK
[ 4] "2+" -> error 2                    esperado: 2            obtenido: 2            OK
[ 5] "  7 "                             esperado: 7            obtenido: 7            OK
[ 6] "5-10"                             esperado: -5           obtenido: -5           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion10.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 10 - Mini-evaluador de expresiones (Hito 1) ===
Caso                               esperado:              obtenido:              resultado
[ 1] "2+3*4"                            esperado: 14           obtenido: 14           OK
[ 2] "(2+3)*4"                          esperado: 20           obtenido: 20           OK
[ 3] "10/0" -> error 1                  esperado: 1            obtenido: 1            OK
[ 4] "2+" -> error 2                    esperado: 2            obtenido: 2            OK
[ 5] "  7 "                             esperado: 7            obtenido: 7            OK
[ 6] "5-10"                             esperado: -5           obtenido: -5           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion10.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA completó las tres funciones del descenso recursivo respetando la precedencia (`expr` llama a `termino`, que llama a `primaria`) y pasa los seis casos. Probé lo que el harness no toca: números grandes, espacios distintos y anidamiento profundo.

### Hallazgo 1: Desbordamiento al leer números y al operar

La lectura hace `v = v * 10 + dígito` sin revisar el límite de `long`, y la suma y la multiplicación tampoco. UBSan lo marca y el resultado sale como si fuera válido, con `error 0`:

```text
ia_sesion10_c.c:34:19: runtime error: signed integer overflow: 999999999999999999 * 10 cannot be represented in type 'long int'
99999999999999999999         valor 7766279631452241919    error 0
9223372036854775807+1        valor -9223372036854775808   error 0
3037000500*3037000500        valor -9223372036709301616   error 0
```

### Hallazgo 2: Mil paréntesis está bien, dos millones tumban el proceso

Cada paréntesis son tres llamadas recursivas más. Con una entrada suficientemente anidada se acaba la pila y el programa muere sin reportar nada:

```text
1000 parentesis anidados -> valor 1 error 0
Segmentation fault (core dumped)   <- con 2000000 parentesis, codigo de salida 139
```

### Hallazgo 3: Un tabulador cuenta como error de sintaxis

`saltar()` solo brinca el carácter espacio. `"2\t+3"` es una expresión válida que cualquier usuario escribiría, y la rechaza:

```text
2	+3                        valor 0                      error 2
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Desbordamiento al leer números y al operar** — reviso el límite antes de cada dígito y uso `__builtin_*_overflow` en `+`, `-` y `*`; el desbordamiento ahora es el error 3.

```text
99999999999999999999         valor 0                      error 3
9223372036854775807+1        valor 0                      error 3
3037000500*3037000500        valor 0                      error 3
```

**Mil paréntesis está bien, dos millones tumban el proceso** — puse un tope de 256 niveles de paréntesis; pasarlo es error de sintaxis (2).

```text
2000000 parentesis anidados -> valor 0 error 2
```

**Un tabulador cuenta como error de sintaxis** — `saltar()` ahora también acepta `\t`, `\n` y `\r`.

```text
2	+3                        valor 5                      error 0
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion10.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Desbordamiento al leer números y al operar** | **CRASH** | Pasa | No |
| **Mil paréntesis está bien, dos millones tumban el proceso** | **CRASH** | Pasa | No |
| **Un tabulador cuenta como error de sintaxis** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(n) en tiempo, donde n es la longitud de la cadena, porque cada carácter se lee una vez. En memoria es O(p), donde p es la profundidad máxima de paréntesis, por la pila de la recursión `expr → termino → primaria → expr`. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Desbordamiento al leer números y al operar.** *Lo que hice:* reviso el límite antes de cada dígito y uso `__builtin_*_overflow` en `+`, `-` y `*`; el desbordamiento ahora es el error 3.
>
> **2. Mil paréntesis está bien, dos millones tumban el proceso.** *Lo que hice:* puse un tope de 256 niveles de paréntesis; pasarlo es error de sintaxis (2).
>
> **3. Un tabulador cuenta como error de sintaxis.** *Lo que hice:* `saltar()` ahora también acepta `\t`, `\n` y `\r`.
>
> **Lo que me llevo.** El descenso recursivo de la IA es correcto en precedencia y paréntesis, que es lo que el Hito 1 pide. Lo que no se pidió y no hizo fue poner límites: límite al tamaño de los números, a los resultados y a la profundidad. En el compilador del equipo esto importa porque el evaluador corre sobre lo que escriba el usuario, y un programa malicioso o simplemente raro no debería poder tumbar el compilador.

---
**Firma digital:** pablouriellopez68-sketch
