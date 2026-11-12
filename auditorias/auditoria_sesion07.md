# Bitácora de Requerimientos y Auditoría IA — Sesión 07
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Evaluación de Expresiones Aritméticas
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Pablo Uriel Desiderio López
**Matrícula:** 24531372
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Evaluador recursivo de operaciones
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> [AQUÍ ESCRIBES TÚ: qué te costó al programarlo a mano. Lo que sí puedes
> contar es el parámetro de salida `int *error`, que es lo nuevo de esta
> sesión: para escribir el código de error hay que poner `*error = 1`, con
> asterisco, porque `error` es la dirección y `*error` es el casillero;
> en cambio al llamar recursivamente se pasa `error` tal cual, sin
> asterisco, porque ya es un puntero. También dónde pusiste el chequeo de
> NULL y por qué tiene que ir antes de leer `n->tipo`.]

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Manejo estricto de división entre cero |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[CONTEXTO]
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Cualquier advertencia cuenta como error. Solo biblioteca estándar.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente esta firma, sin alterarla:

    typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;

    typedef struct ASTNode {
        TipoNodo tipo;
        int valor;
        char op;
        struct ASTNode *izq;
        struct ASTNode *der;
    } ASTNode;

    int evaluar(ASTNode *n, int *error);

- Evalúa recursivamente el árbol de expresión y devuelve el resultado.
- Un nodo NODO_NUM es una hoja: su valor está en el campo `valor`.
- Un nodo NODO_BINOP aplica el operador de `op` a sus dos hijos.
- Operadores soportados: + - * / %
- Si se intenta dividir o sacar módulo entre cero, escribe 1 en `*error`.
- Si el árbol (o un hijo esperado) es NULL, escribe 2 en `*error`.
- Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion07.h"
#include <stdio.h>
#include <string.h>

int evaluar(ASTNode *n, int *error) {
    if (n == NULL) {
        *error = 2;
        return 0;
    }
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }
    int a = evaluar(n->izq, error);
    int b = evaluar(n->der, error);
    switch (n->op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
        case '/':
            if (b == 0) { *error = 1; return 0; }
            return a / b;
        case '%':
            if (b == 0) { *error = 1; return 0; }
            return a % b;
    }
    return 0;
}
```

**Complejidad declarada por el modelo:** O(n) en tiempo, donde n es el número de nodos, porque visita cada uno exactamente una vez. O(h) en memoria por la pila de recursión, donde h es la altura del árbol; en el peor caso (un árbol degenerado en lista) h = n.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Expresiones complejas y división por 0

### Salida del harness
```text
=== AUDITORIA SESION 07 - Evaluador recursivo con deteccion de errores ===
Caso                               esperado:              obtenido:              resultado
[ 1] (2+3)*4                            esperado: 20           obtenido: 20           OK
[ 2] 7/0 -> error 1                     esperado: 1            obtenido: 1            OK
[ 3] 7%0 -> error 1                     esperado: 1            obtenido: 1            OK
[ 4] arbol NULL -> error 2              esperado: 2            obtenido: 2            OK
[ 5] 10-2-3                             esperado: 5            obtenido: 5            OK
[ 6] (8/2)+1                            esperado: 5            obtenido: 5            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion07.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 en Google Colab, compilado con `-std=c11 -Wall -Wextra -Werror`.

La versión de la IA compila sin advertencias y pasa los seis casos. La recursión está bien planteada y el chequeo de división entre cero está donde debe. Pero las seis pruebas usan árboles pequeños y números chicos, así que me puse a probar con lo que el harness no toca.

### Hallazgo 1: `INT_MIN / -1` mata el proceso aunque el divisor no sea cero

Este es el peor. El código revisa `if (b == 0)` y da por hecho que con eso la división ya es segura. No lo es: en complemento a dos el entero más negativo no tiene positivo, así que `-2147483648 / -1` no cabe en un `int`. El divisor es `-1`, pasa el `if` sin problema, y el procesador lanza una excepción de hardware:

```text
ia_sesion07.c:21:22: runtime error: division of -2147483648 by -1 cannot be represented in type 'int'
AddressSanitizer:DEADLYSIGNAL
==267==ERROR: AddressSanitizer: FPE on unknown address 0x55fc4320a8e7
    #0 0x55fc4320a8e7 in evaluar ia_sesion07.c:21
```

Sin sanitizers el programa simplemente muere con `Floating point exception` y código de salida 136. No hay valor de retorno, no hay código de error, no hay nada: el compilador deja de existir a media evaluación.

### Hallazgo 2: `evaluar(árbol, NULL)` escribe en la dirección cero

La función escribe en `*error` sin preguntarse nunca si `error` es un puntero válido. Si alguien llama sin querer el código de error, la primera escritura revienta:

```text
ia_sesion07.c:7:16: runtime error: store to null pointer of type 'int'
==269==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000
==269==The signal is caused by a WRITE memory access.
    #0 0x5570319654ca in evaluar ia_sesion07.c:7
```

### Hallazgo 3: un operador que no existe devuelve 0 sin avisar

El `switch` sólo cubre `+ - * / %`. Cualquier otro carácter cae al `return 0;` del final, sin tocar `*error`. Entonces una expresión con un operador que el analizador no supo leer devuelve exactamente lo mismo que una expresión legítima que vale cero:

```text
2^3 (operador desconocido)        valor 0   error 0
```

### Hallazgo 4: la evaluación no se detiene cuando ya hubo un error

Esto hace que el valor de retorno sea basura con pinta de resultado:

```text
(7/0)+5                           valor 5   error 1
```

La división entre cero devolvió 0, el `+` siguió adelante y la función entregó un **5**. Si el que llama se fija sólo en el valor de retorno —que es la costumbre— se lleva un número que nunca existió.

### Hallazgo 5: el código de error se pisa, y nunca se limpia

El `*error` se escribe desde cualquier punto de la recursión, así que el último error en escribirse gana. Una división entre cero desaparece si después aparece un nodo ausente:

```text
(7/0)+<nodo ausente>              valor 0   error 2
```

Y al revés: la función nunca pone `*error = 0` al empezar, así que si el que llama no limpió la variable, una expresión perfectamente válida reporta error:

```text
2+3 con error sucio (entra 99)    valor 5   error 99
```

### Hallazgo 6: desbordamiento con signo sin detectar

```text
ia_sesion07.c:16:28: runtime error: signed integer overflow:
2000000000 + 2000000000 cannot be represented in type 'int'
```

Devuelve `-294967296` con `error = 0`, o sea lo presenta como un resultado correcto.

### Resultados tras endurecer la implementación

Reescribí la función partiéndola en dos: una `evaluar` pública que prepara el estado y una `eval_rec` interna que hace la recursión. Así la limpieza del error ocurre una sola vez, al principio, y no en cada llamada. Añadí cuatro códigos (1 división entre cero, 2 nodo ausente, 3 operador desconocido, 4 desbordamiento), un guardia para cuando `error` viene en NULL, corto circuito para dejar de evaluar en cuanto hay un error, y los chequeos de rango antes de cada operación.

Misma tabla, las dos versiones:

| caso | IA (valor / error) | Endurecido (valor / error) |
| :--- | :--- | :--- |
| `2^3` (operador desconocido) | 0 / 0 | 0 / 3 |
| `(7/0)+5` | **5** / 1 | 0 / 1 |
| `(7/0)+<nodo ausente>` | 0 / **2** | 0 / 1 |
| `2+3` con error sucio (entra 99) | 5 / **99** | 5 / 0 |
| `2000000000+2000000000` | **-294967296** / 0 | 0 / 4 |
| `INT_MIN / -1` | **SIGFPE, muere** | 0 / 4 |
| `evaluar(árbol, NULL)` | **SIGSEGV** | sobrevive, devuelve 0 |

Y el harness sigue en verde:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion07.md
```

| Caso borde | IA | Endurecido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| `(2+3)*4`, `10-2-3`, `(8/2)+1` | Pasa | Pasa | Sí |
| `7/0` y `7%0` | Pasa | Pasa | Sí |
| Árbol NULL | Pasa | Pasa | Sí |
| **`INT_MIN / -1`** | **CRASH** | Pasa | No |
| **Puntero `error` en NULL** | **CRASH** | Pasa | No |
| **Operador no reconocido** | **FALLA** | Pasa | No |
| **Seguir evaluando tras un error** | **FALLA** | Pasa | No |
| **Error pisado / nunca limpiado** | **FALLA** | Pasa | No |
| **Desbordamiento con signo** | **FALLA** | Pasa | No |

**Complejidad verificada:** sigue en O(n) de tiempo y O(h) de memoria. Los chequeos que agregué son comparaciones constantes dentro de cada nodo, así que no cambian el orden; el corto circuito sólo puede visitar menos nodos, nunca más. Lo que declaró el modelo era correcto y la corrección no lo cambia.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El modelo entregó una recursión limpia que pasa los seis casos y compila con `-Werror` sin una sola advertencia. El problema es que todo lo que protege está protegido por una sola pregunta —`¿el divisor es cero?`— y resulta que esa pregunta no cubre ni la mitad de lo que puede salir mal.
>
> **1. `INT_MIN / -1` tumba el proceso.** Este fue el hallazgo que no me esperaba. El divisor es `-1`, no es cero, así que pasa el `if` sin resistencia; pero en complemento a dos el entero más negativo no tiene positivo, el cociente no cabe en un `int` y el procesador lanza una excepción. El programa muere con `Floating point exception`, sin valor de retorno y sin código de error. Un compilador que se cae en vez de reportar un error es peor que uno que reporta mal. *Lo arreglé* agregando `if (a == INT_MIN && b == -1)` junto al chequeo de cero, en división y en módulo.
>
> **2. Escribe en `*error` sin comprobar que el puntero exista.** Si alguien llama con `NULL` ahí, la primera instrucción de la función escribe en la dirección cero. *Lo arreglé* con una variable local de descarte: si `error` viene en NULL, apunto a esa y la función sigue funcionando en vez de reventar.
>
> **3. Un operador desconocido devuelve 0 en silencio.** El `switch` no tiene `default`, así que cualquier carácter raro cae al `return 0;` del final sin tocar el error. Eso hace que un fallo del analizador léxico sea indistinguible de una expresión que legítimamente vale cero. *Lo arreglé* con un `default` que pone el código 3.
>
> **4. La evaluación no se detiene tras un error.** `(7/0)+5` devuelve **5**. La división falló, devolvió 0, y la suma siguió como si nada. El error sí queda marcado en la bandera, pero quien revise sólo el valor de retorno se lleva un número inventado. *Lo arreglé* revisando el error después de cada rama y cortando ahí mismo.
>
> **5. El código de error se pisa y nunca se limpia.** Como cualquier punto de la recursión escribe en la misma variable, gana el último: una división entre cero desaparece si luego aparece un nodo ausente. Y como la función nunca pone el error en 0 al empezar, hereda la basura que traiga el que llama. *Lo arreglé* separando la función en dos, una pública que limpia el estado una sola vez y una interna que recursa.
>
> **6. Desbordamiento con signo sin detectar**, que devuelve un negativo presentándolo como resultado válido. *Lo arreglé* con chequeos de rango antes de sumar, restar y multiplicar.
>
> **Lo que me llevo.** Las seis fallas salen del mismo hábito: la IA protegió el caso que el enunciado nombró —división entre cero— y dio por supuesto que ese era el único peligro de la división. Pero el enunciado nombra los casos que ya conocemos, y auditar es justamente buscar los que no están en la lista. También me quedó claro que un parámetro de salida es un contrato de dos partes: no basta con escribir el error cuando pasa algo, hay que decir quién lo limpia, qué pasa si se pisa y si el valor de retorno significa algo cuando la bandera está encendida. Nada de eso estaba dicho, y por eso las tres cosas salieron mal.

---
**Firma digital:** pablouriellopez68-sketch
