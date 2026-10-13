# App del Diario de Sueños

Aplicación personal (un solo usuario) que reemplaza al sistema de registro de
sueños que hoy funciona como proyecto de Claude. Traslada a software lo que es
mecánico —IDs, cómputo, recurrencias, tasas, exportación, PDF— y deja al modelo
de lenguaje sólo lo que requiere lenguaje.

Español (Chile) en toda la interfaz y todo el contenido.

## Estado

Etapa 0, sin empezar. El plan de implementación está propuesto y espera
aprobación.

- [`docs/plan-implementacion.md`](docs/plan-implementacion.md) — stack,
  estructura del repositorio, esquema de base de datos, pruebas de invariante,
  etapas y decisiones pendientes.
- `docs/especificacion.md` — pendiente de agregar: es el documento que gobierna
  este repositorio.

## Los invariantes

No se negocian, y en esta app son arquitectura, no buena conducta:

1. El crudo del sueño es inmutable y nunca se edita.
2. El crudo no pasa por el modelo en el camino de escritura.
3. Las correcciones de transcripción son notas aparte, confirmadas por Roberto.
4. Los IDs son permanentes, correlativos y los asigna sólo el servidor.
5. Cómputo e interpretación están separados: el modelo nunca calcula un hecho
   del corpus.
6. El veredicto es sólo de Roberto.
7. La captura es silenciosa.
8. El sistema degrada sin castigo: ni rachas, ni recordatorios, ni culpa.
9. Facetas, no categorías.
10. Sin diccionarios de símbolos.
11. Privacidad: nada expuesto a internet público, nada de corpus en este
    repositorio.

## Privacidad

El corpus (bitácoras, entidades, taller, base de datos, respaldos) **no se
versiona aquí**. Los datos viven en el VPS, accesible sólo por Tailscale. Las
pruebas usan sueños sintéticos.
