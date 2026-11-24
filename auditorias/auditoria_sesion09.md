# Bitácora de Requerimientos y Auditoría IA — Sesión 09
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Reporte Formal de Errores Semánticos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Diagnóstico con línea y columna  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> No conocía `snprintf`. Entendí que funciona como `printf` pero escribe en un buffer y nunca se pasa del tamaño que le das, y que por eso el caso `cap=10` sale cortado en `main.c:14`: solo caben 9 caracteres más el `'\0'`.
>
> Lo que más me costó fue Colab, no el código. Primero el harness daba `0/4` con el texto vacío porque yo no había cambiado el esqueleto, que solo hace `salida[0] = '\0'`. Luego, por error, edité la celda de `api_sesion09.h` en lugar de la del artesano y borré la declaración de `reportar`. Tuve que restaurarla con todos sus parámetros. Aprendí a fijarme en la primera línea de cada celda (`%%writefile api_...` contra `%%writefile artesano_...`) y a volver a correrla después de cambiarla.
>
> Mi versión quedó igual a la de la IA, lo cual tiene sentido porque con `snprintf` el ejercicio casi tiene una sola forma de escribirse.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Estructura estándar de errores GCC |
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
Implementa exactamente estas declaraciones de api_sesion09.h, sin alterarlas:

void reportar(const char *archivo, int linea, int columna, const char *mensaje, char *salida, int cap);

Reglas del contrato:
- El formato es exacto: sin espacios extra ni punto final.
- Nunca escribas más de cap-1 caracteres: el buffer es del llamador.
- El resultado siempre termina en '\0'.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion09.h"
#include <stdio.h>

void reportar(const char *archivo, int linea, int columna, const char *mensaje, char *salida, int cap) {
    if (salida == NULL || cap <= 0) return;
    snprintf(salida, (size_t)cap, "%s:%d:%d: error: %s", archivo, linea, columna, mensaje);
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo, donde L es la longitud del mensaje armado (lo que tarda `snprintf`), y O(1) de memoria extra porque escribe directo en el buffer del llamador.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Múltiples errores sin abortar parser

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 09 - Diagnostico con linea y columna ===
Caso                               esperado:              obtenido:              resultado
[ 1] main.c:14:5
     esperado: main.c:14:5: error: tipo incompatible
     obtenido: main.c:14:5: error: tipo incompatible
     -> OK
[ 2] a.c:1:1
     esperado: a.c:1:1: error: x
     obtenido: a.c:1:1: error: x
     -> OK
[ 3] linea 0
     esperado: x.c:0:0: error: y
     obtenido: x.c:0:0: error: y
     -> OK
[ 4] cap=10 trunca                      esperado: main.c:14    obtenido: main.c:14    OK

RESULTADO: 4/4 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion09.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 09 - Diagnostico con linea y columna ===
Caso                               esperado:              obtenido:              resultado
[ 1] main.c:14:5
     esperado: main.c:14:5: error: tipo incompatible
     obtenido: main.c:14:5: error: tipo incompatible
     -> OK
[ 2] a.c:1:1
     esperado: a.c:1:1: error: x
     obtenido: a.c:1:1: error: x
     -> OK
[ 3] linea 0
     esperado: x.c:0:0: error: y
     obtenido: x.c:0:0: error: y
     -> OK
[ 4] cap=10 trunca                      esperado: main.c:14    obtenido: main.c:14    OK

RESULTADO: 4/4 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion09.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

Aquí la IA usó `snprintf` con el tamaño del buffer, que es justo lo correcto: nunca escribe más de `cap-1` caracteres y siempre pone el `'\0'`. Los cuatro casos pasan. Lo que revisé fueron los argumentos raros.

### Hallazgo 1: `archivo` o `mensaje` en NULL dependen de glibc

Pasarle `NULL` a un `%s` es comportamiento indefinido en C. En Colab funciona porque glibc imprime `(null)`, pero en otra biblioteca estándar puede tronar. El diagnóstico tampoco le dice nada útil al usuario.

```text
archivo NULL   -> "(null):1:1: error: x"
mensaje NULL   -> "a.c:1:1: error: (null)"
```

### Hallazgo 2: Línea o columna negativas se imprimen tal cual

Una línea `-3` no existe en ningún archivo. Si llega así es porque algo antes falló, y el diagnóstico la repite sin chistar.

```text
linea -3       -> "a.c:-3:0: error: x"
```

### Hallazgo 3: El truncado es silencioso

Con `cap=12` el mensaje queda cortado a la mitad y la función no avisa. El contrato pide que la firma sea `void`, así que no lo cambié, pero lo dejo anotado: el que llama no tiene forma de saber que perdió parte del diagnóstico. `snprintf` sí devuelve cuántos caracteres *quería* escribir; con una firma que devolviera `int` se podría detectar.

```text
cap=12         -> "a.c:1:1: er" (sin aviso de truncado)
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**`archivo` o `mensaje` en NULL dependen de glibc** — sustituyo NULL por un texto fijo antes de llamar a `snprintf`.

```text
archivo NULL   -> "<desconocido>:1:1: error: x"
mensaje NULL   -> "a.c:1:1: error: (sin mensaje)"
```

**Línea o columna negativas se imprimen tal cual** — las negativas se normalizan a 0, igual que el caso `linea 0` del harness.

```text
linea -3       -> "a.c:0:0: error: x"
```

**El truncado es silencioso** — no lo cambié porque rompería la firma del contrato; lo documento como limitación.

```text
cap=12         -> "a.c:1:1: er" (sin aviso de truncado)
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 4/4 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion09.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **`archivo` o `mensaje` en NULL dependen de glibc** | **FALLA** | Pasa | No |
| **Línea o columna negativas se imprimen tal cual** | **FALLA** | Pasa | No |
| El truncado es silencioso | Limitación | Sin cambio (firma del contrato) | No |

**Complejidad verificada:** O(L) en tiempo, donde L es la longitud del mensaje armado (lo que tarda `snprintf`), y O(1) de memoria extra porque escribe directo en el buffer del llamador. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. `archivo` o `mensaje` en NULL dependen de glibc.** *Lo que hice:* sustituyo NULL por un texto fijo antes de llamar a `snprintf`.
>
> **2. Línea o columna negativas se imprimen tal cual.** *Lo que hice:* las negativas se normalizan a 0, igual que el caso `linea 0` del harness.
>
> **3. El truncado es silencioso.** *Lo que hice:* no lo cambié porque rompería la firma del contrato; lo documento como limitación.
>
> **Lo que me llevo.** Esta fue la sesión donde la IA salió más sólida: usó `snprintf` y no `sprintf`, que es lo que más importaba. Lo que omitió fue pensar en quién le pasa los datos. Un reporte de errores se llama justo cuando algo ya salió mal, así que es el lugar donde más probable es recibir un NULL o una línea basura, y donde menos queremos que el programa truene.

---
**Firma digital:** pablouriellopez68-sketch