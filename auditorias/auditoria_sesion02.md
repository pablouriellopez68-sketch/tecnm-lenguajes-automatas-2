# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Gestión en GitHub Web y Nodos Dinámicos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** [Nombre Completo]  
**Matrícula:** [No. de Control]  
**Track:** [ C | Rust ]  

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
Únicamente la biblioteca estándar de C99: sin POSIX, sin extensiones GNU,
sin dependencias externas. Destino: front-end de un compilador, tabla de
tokens construida en memoria dinámica.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores. Tu criterio
de calidad es la seguridad de memoria y el cumplimiento estricto del
estándar, no la brevedad.

[ESPECIFICACIÓN]
Implementa una lista enlazada simple de tokens alojada en el heap:

    typedef struct Token { char *lexema; struct Token *sig; } Token;
    typedef struct { Token *cabeza; size_t n; } Lista;

    void lista_init(Lista *l);
    int  lista_agregar(Lista *l, const char *lexema);
    void lista_liberar(Lista *l);

Requisitos obligatorios:
- La lista debe ser DUEÑA de su copia del lexema: `lista_agregar` reserva
  memoria propia y copia la cadena, de modo que el llamador pueda
  modificar o destruir su buffer sin afectar a la lista.
- Toda función que reserve memoria debe comprobar el valor devuelto por
  malloc y propagar el fallo sin dejar reservas a medias.
- `lista_liberar` debe liberar el lexema Y el nodo de cada elemento, y
  dejar la lista en un estado reutilizable, de forma que una segunda
  llamada sea inocua (sin doble free ni use-after-free).
- Entradas nulas: puntero a lista NULL y lexema NULL deben rechazarse
  devolviendo -1, sin desreferenciar.
- La cadena vacía "" es un lexema válido, no un error.
- Complejidad obligatoria: `lista_agregar` en O(1); `lista_liberar` en
  O(n) con O(1) de memoria auxiliar (sin recursión).
- No uses strdup: no forma parte de C99.

[OUTPUT]
Entrega únicamente:
1. El código C modular (cabecera + implementación).
2. Un test harness con main() que ejerza: lista vacía, lexema NULL,
   lista NULL, cadena vacía, independencia de la copia, doble liberación
   y liberación de NULL.
3. El análisis de complejidad en tiempo y memoria, justificado.
No incluyas explicación en prosa fuera de los comentarios del código.

```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
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
