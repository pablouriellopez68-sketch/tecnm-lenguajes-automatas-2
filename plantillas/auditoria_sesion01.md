# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** [Nombre Completo]  
**Matrícula:** [No. de Control]  
**Track:** [ C | Rust ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Inversión de buffer in-place (char*)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> /* reverse_buffer.c — Lenguajes y Autómatas II (SCD-1016), Sesión 01
 * Inversión de buffer in-place sobre char*.
 * Restricciones: C99, sin bibliotecas no estándar, O(n) tiempo, O(1) memoria.
 *
 * Compilar:
 *   gcc -std=c99 -Wall -Wextra -pedantic -fsanitize=address,undefined -g \
 *       -o reverse_test reverse_buffer.c
 */

#include <stdio.h>
#include <string.h>
#include <assert.h>

/* --------------------------------------------------------------
 * Intercambio de un rango cerrado [a, b]. Núcleo compartido.
 * -------------------------------------------------------------- */
static void reverse_range(char *a, char *b)
{
    while (a < b) {
        char tmp = *a;
        *a++ = *b;
        *b-- = tmp;
    }
}

/* --------------------------------------------------------------
 * V1 — Inversión byte a byte (la pedida por la especificación).
 * Invierte in-place la cadena terminada en '\0' apuntada por buf.
 * Devuelve el mismo puntero (o NULL si la entrada es NULL).
 * PRECONDICIÓN: buf debe apuntar a memoria escribible, NUNCA a un
 * literal de cadena (char *s = "x" vive en .rodata -> SIGSEGV).
 * Complejidad: O(n) tiempo (una pasada para medir + n/2 swaps), O(1) memoria.
 * -------------------------------------------------------------- */
char *reverse_buffer(char *buf)
{
    if (buf == NULL) {
        return NULL;                 /* contrato defensivo: NULL in, NULL out */
    }

    char *end = buf;
    while (*end != '\0') {           /* equivalente a strlen, sin size_t */
        end++;
    }
    if (end == buf) {
        return buf;                  /* cadena vacía: nada que invertir */
    }

    reverse_range(buf, end - 1);     /* end-1 = último carácter real */
    return buf;
}

/* --------------------------------------------------------------
 * V2 — Corrección del caso borde detectado en la auditoría:
 * la V1 parte las secuencias multibyte UTF-8 y produce mojibake.
 * Estrategia: invertir todos los bytes y después re-invertir, en
 * sitio, cada secuencia multibyte (sus bytes de continuación
 * 10xxxxxx quedan ANTES de su byte líder tras la inversión global).
 * Sigue siendo O(n) tiempo / O(1) memoria: cada byte se toca 2 veces.
 * -------------------------------------------------------------- */
char *reverse_utf8(char *buf)
{
    if (buf == NULL) {
        return NULL;
    }
    reverse_buffer(buf);             /* paso 1: inversión global de bytes */

    char *p = buf;                   /* paso 2: reparar cada secuencia */
    while (*p != '\0') {
        char *q = p;
        while (((unsigned char)*q & 0xC0) == 0x80) {
            q++;                     /* avanza sobre bytes de continuación */
        }
        if (q > p) {
            reverse_range(p, q);     /* restaura el orden líder + continuación */
        }
        p = q + 1;
    }
    return buf;
}

/* ---------------------- Test Harness ---------------------- */

static int fallos = 0;

static void check(const char *caso, const char *obtenido, const char *esperado)
{
    int ok = (obtenido == NULL && esperado == NULL) ||
             (obtenido != NULL && esperado != NULL && strcmp(obtenido, esperado) == 0);
    if (!ok) fallos++;
    printf("[%s] %-22s esperado=\"%s\"  obtenido=\"%s\"\n",
           ok ? "OK  " : "FALLA", caso,
           esperado ? esperado : "(null)",
           obtenido ? obtenido : "(null)");
}

int main(void)
{
    puts("== V1: reverse_buffer (byte a byte) ==");

    /* 1. Puntero NULL */
    check("NULL", reverse_buffer(NULL), NULL);

    /* 2. Cadena vacía */
    char vacia[] = "";
    check("cadena vacia", reverse_buffer(vacia), "");

    /* 3. Un solo carácter */
    char uno[] = "a";
    check("un caracter", reverse_buffer(uno), "a");

    /* 4. Longitud par */
    char par[] = "abcd";
    check("longitud par", reverse_buffer(par), "dcba");

    /* 5. Longitud impar (el central no se mueve) */
    char impar[] = "abcde";
    check("longitud impar", reverse_buffer(impar), "edcba");

    /* 6. Caso realista */
    char frase[] = "Lenguajes";
    check("palabra", reverse_buffer(frase), "sejaugneL");

    /* 7. Doble inversión = identidad */
    char ida[] = "automatas";
    reverse_buffer(ida);
    check("doble inversion", reverse_buffer(ida), "automatas");

    /* 8. El buffer puede ser mayor que la cadena: no se toca la cola */
    char holgado[16] = "hola";
    reverse_buffer(holgado);
    check("buffer holgado", holgado, "aloh");
    assert(holgado[5] == '\0');  /* el relleno posterior sigue intacto */

    /* 9. Bytes de control embebidos antes del terminador */
    char bin[] = {'x', '\t', '9', '\0'};
    check("bytes de control", reverse_buffer(bin), "9\tx");

    /* 10. FALLO CONOCIDO documentado: UTF-8 roto por la V1 */
    char utf1[] = "Autómatas";
    reverse_buffer(utf1);
    printf("[NOTA] V1 sobre UTF-8     \"Autómatas\" -> \"%s\" (mojibake esperado)\n", utf1);

    puts("\n== V2: reverse_utf8 (consciente de multibyte) ==");

    char utf2[] = "Autómatas";
    check("UTF-8 acentuado", reverse_utf8(utf2), "satamótuA");

    char utf3[] = "ñandú";
    check("UTF-8 extremos", reverse_utf8(utf3), "údnañ");

    char utf4[] = "año 2026";
    check("UTF-8 mixto", reverse_utf8(utf4), "6202 oña");

    char ascii[] = "abcde";
    check("ASCII via V2", reverse_utf8(ascii), "edcba");

    char v2vacia[] = "";
    check("vacia via V2", reverse_utf8(v2vacia), "");

    check("NULL via V2", reverse_utf8(NULL), NULL);

    printf("\n%s (%d fallos)\n", fallos == 0 ? "TODAS LAS PRUEBAS PASARON" : "HAY FALLOS", fallos);
    return fallos == 0 ? 0 : 1;
}

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Punteros, O(n) tiempo, O(1) memoria |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[Pega aquí el prompt formal que redactaste para la IA]
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Valores NULL, buffers vacios y longitudes impares

* [ ] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [ ] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> 

---
**Firma digital:** [Usuario de GitHub]
