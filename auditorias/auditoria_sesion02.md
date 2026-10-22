# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Gestión en GitHub Web y Nodos Dinámicos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López
**Matrícula:** 24531272
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lista enlazada de tokens en Heap  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> 

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Gestión de memoria dinámica C99/Rust |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[CONTEXTO]
Estándar C99 compilado con GCC usando -std=c99 -Wall -Wextra -pedantic.
Únicamente biblioteca estándar; sin dependencias externas. Destino:
front-end de un compilador, tabla de tokens construida en memoria dinámica.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa una lista enlazada simple de tokens alojada en el heap:

    typedef struct Token { char *lexema; struct Token *sig; } Token;
    typedef struct { Token *cabeza; size_t n; } Lista;

    void lista_init(Lista *l);
    int  lista_agregar(Lista *l, const char *lexema);
    void lista_liberar(Lista *l);

- `lista_agregar` inserta un token nuevo con el lexema indicado y
  devuelve 0 en éxito.
- `lista_liberar` libera toda la memoria reservada por la lista.
- Debe manejar entradas nulas sin desreferenciar punteros inválidos.
- Complejidad obligatoria: inserción en O(1), liberación en O(n) con
  O(1) de memoria auxiliar.

[OUTPUT]
Entrega únicamente:
1. El código C modular (cabecera + implementación).
2. Un test harness con main() que ejercite la lista.
3. El análisis de complejidad en tiempo y memoria, justificado.
No incluyas explicación en prosa fuera de los comentarios del código.

```

---

## 3. Código Devuelto por el Modelo
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Token {
    char *lexema;
    struct Token *sig;
} Token;

typedef struct {
    Token *cabeza;
} Lista;

void lista_init(Lista *l)
{
    l->cabeza = NULL;
}

int lista_agregar(Lista *l, const char *lexema)
{
    Token *n = malloc(sizeof(Token));
    n->lexema = strdup(lexema);
    n->sig = l->cabeza;
    l->cabeza = n;
    return 0;
}

void lista_liberar(Lista *l)
{
    Token *p = l->cabeza;
    while (p) {
        Token *sig = p->sig;
        free(p);
        p = sig;
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Prevención de Use-After-Free y cadenas nulas

* [ ] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [ ] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> 

---
**Firma digital:** [Usuario de GitHub]
