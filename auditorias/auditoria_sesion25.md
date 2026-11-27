# Bitácora de Requerimientos y Auditoría IA — Sesión 25
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Control de Flujo: Bucles while y for  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Estructura de ciclo en instrucciones TAC  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion25_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Garantía de actualización de iterador |
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
Implementa exactamente estas declaraciones de api_sesion25.h, sin alterarlas:

void emitir_while(char lineas[][64], int max, int *n, const char *cond, int *etiquetas);
void emitir_for(char lineas[][64], int max, int *n, const char *inicio, const char *cond, const char *post, int *etiquetas);

Reglas del contrato:
- El while emite cuatro líneas: marca de inicio, salto condicional, GOTO de regreso y marca de fin.
- El for emite init, marca, salto condicional, post, GOTO y marca de fin; si post es NULL, se omite la actualización.
- Cada ciclo consume dos etiquetas y las agrega a *etiquetas; no reutilices nombres.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion25.h"
#include <stdio.h>

static void linea(char lineas[][64], int max, int *n, const char *texto) {
    if (*n < max) {
        snprintf(lineas[*n], 64, "%s", texto);
        (*n)++;
    }
}

void emitir_while(char lineas[][64], int max, int *n, const char *cond, int *etiquetas) {
    if (lineas == NULL || n == NULL || cond == NULL || etiquetas == NULL) return;
    char ini[16], fin[16], buf[64];
    snprintf(ini, sizeof ini, "L%d", (*etiquetas)++);
    snprintf(fin, sizeof fin, "L%d", (*etiquetas)++);
    snprintf(buf, sizeof buf, "%s:", ini);                       linea(lineas, max, n, buf);
    snprintf(buf, sizeof buf, "IFFALSE %s GOTO %s", cond, fin);  linea(lineas, max, n, buf);
    snprintf(buf, sizeof buf, "GOTO %s", ini);                   linea(lineas, max, n, buf);
    snprintf(buf, sizeof buf, "%s:", fin);                       linea(lineas, max, n, buf);
}

void emitir_for(char lineas[][64], int max, int *n, const char *inicio, const char *cond, const char *post, int *etiquetas) {
    if (lineas == NULL || n == NULL || cond == NULL || etiquetas == NULL) return;
    char ini[16], fin[16], buf[64];
    snprintf(ini, sizeof ini, "L%d", (*etiquetas)++);
    snprintf(fin, sizeof fin, "L%d", (*etiquetas)++);
    if (inicio != NULL) linea(lineas, max, n, inicio);
    snprintf(buf, sizeof buf, "%s:", ini);                       linea(lineas, max, n, buf);
    snprintf(buf, sizeof buf, "IFFALSE %s GOTO %s", cond, fin);  linea(lineas, max, n, buf);
    if (post != NULL) linea(lineas, max, n, post);
    snprintf(buf, sizeof buf, "GOTO %s", ini);                   linea(lineas, max, n, buf);
    snprintf(buf, sizeof buf, "%s:", fin);                       linea(lineas, max, n, buf);
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo por los `snprintf`, con L la longitud de las expresiones, y O(1) en memoria. El while emite 4 líneas y el for hasta 6.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Bucles anidados y condiciones de parada

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion25_c.c artesano_sesion25_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 25 - Esqueleto TAC de while y for ===
Caso                               esperado:              obtenido:              resultado
[ 1] while: marca de inicio             esperado: L0:          obtenido: L0:          OK
[ 2] while: condicion
     esperado: IFFALSE a < b GOTO L1
     obtenido: IFFALSE a < b GOTO L1
     -> OK
[ 3] dos ciclos sin colision            esperado: L2:          obtenido: L2:          OK
[ 4] for: inicializacion                esperado: i = 0        obtenido: i = 0        OK
[ 5] for: actualizacion                 esperado: i = i + 1    obtenido: i = i + 1    OK
[ 6] for sin post (NULL)                esperado: 5            obtenido: 5            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion25.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA saca dos etiquetas por ciclo (inicio y fin), emite el while en cuatro líneas y el for desazucarado en seis, y omite `post` cuando es NULL. Los seis casos pasan, incluido el de dos ciclos seguidos sin colisión de etiquetas. Revisé qué pasa cuando no hay espacio o la condición es muy larga.

### Hallazgo 1: Si no caben las cuatro líneas, el ciclo queda sin marca de fin

Con `max = 3`, la IA escribe la marca de inicio, el salto condicional y el `GOTO` de regreso, pero no la marca `L1:`. El salto condicional apunta a una etiqueta que no existe, y además ya gastó las dos etiquetas:

```text
while con max=3 -> n=3, etiquetas usadas=2, ultima linea: "GOTO L0"
```

### Hallazgo 2: Una condición larga corta el destino del salto

Cada línea mide 64 bytes. Con una condición larga, `IFFALSE ... GOTO L1` se corta al final y el salto queda como `GOTO L`:

```text
condicion larga -> "IFFALSE contador_principal < limite_superior_del_arreglo GOTO L"
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Si no caben las cuatro líneas, el ciclo queda sin marca de fin** — calculo antes cuántas líneas necesita el ciclo; si no caben todas, no emito nada ni gasto etiquetas.

```text
while con max=3 -> n=0, etiquetas usadas=0
```

**Una condición larga corta el destino del salto** — antes de emitir reviso que la línea del salto quepa completa en 63 caracteres.

```text
condicion larga -> ""
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion25.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Si no caben las cuatro líneas, el ciclo queda sin marca de fin** | **FALLA** | Pasa | No |
| **Una condición larga corta el destino del salto** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(L) en tiempo por los `snprintf`, con L la longitud de las expresiones, y O(1) en memoria. El while emite 4 líneas y el for hasta 6. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Si no caben las cuatro líneas, el ciclo queda sin marca de fin.** *Lo que hice:* calculo antes cuántas líneas necesita el ciclo; si no caben todas, no emito nada ni gasto etiquetas.
>
> **2. Una condición larga corta el destino del salto.** *Lo que hice:* antes de emitir reviso que la línea del salto quepa completa en 63 caracteres.
>
> **Lo que me llevo.** La estructura de los ciclos de la IA es la correcta. El problema es que emite línea por línea sin saber si va a terminar, y un ciclo a medias es un salto a ninguna parte. Aprendí a revisar todo lo que necesito antes de escribir la primera línea, para que el resultado sea completo o no exista.

---
**Firma digital:** pablouriellopez68-sketch