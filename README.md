# Red de Funciones IA — Mapa editable compartido

Mapa interactivo de la taxonomía funcional de la IA (categorías → capacidades → funciones específicas), editable por cualquier persona con el enlace.

**Mapa en vivo:** https://edreirbs.github.io/mapa-red-funciones-ia/

## Cómo funciona

- **Ver:** abre el enlace. Siempre carga la última versión guardada.
- **Editar:** clic en cualquier texto para renombrarlo; arrastra los nodos para moverlos; selecciona un nodo y usa «+» para agregar categorías/capacidades/funciones o «eliminar» para quitarlas.
- **Guardado automático:** cada cambio se guarda solo (≈1 segundo después de editar) en la base de datos compartida. No hay botón de guardar.
- **Deshacer / rehacer:** `Ctrl+Z` (o `Cmd+Z` en Mac) deshace tu último cambio; `Ctrl+Y` o `Ctrl+Shift+Z` lo rehace. El historial de deshacer es local a tu pestaña (cubre lo que tú hiciste en esta sesión) y lo que deshagas también se guarda para todos.
- **Sincronización:** si alguien más edita mientras tienes el mapa abierto, verás sus cambios en unos 12 segundos.
- **⤓ imagen PNG:** descarga la versión actual del mapa como imagen en cualquier momento.
- **⤓ respaldo HTML:** descarga una copia autónoma del mapa con el estado actual incrustado (útil como respaldo offline).
- **Contador automático:** la cuenta de categorías · capacidades · funciones (esquina inferior derecha) se recalcula sola al agregar o borrar ramas; no es editable.
- **Zoom y paneo:** rueda del mouse (o pinch) hace zoom hacia el cursor; arrastra el fondo para desplazarte; controles − / % / + / «ajustar» en la esquina inferior derecha. El zoom es local a tu pantalla, no se comparte.
- **🔍 Buscar:** la barra de la esquina superior derecha (sustituye a la antigua leyenda de puntos) resalta en neón los nodos que coinciden con lo que escribes — sin distinguir mayúsculas ni acentos, con varias palabras en cualquier orden. La leyenda de puntos sigue apareciendo en el PNG exportado.
- **⊞ Vista tablero:** el botón de la esquina inferior izquierda alterna entre la red y un tablero de tarjetas por categoría, mucho más fácil de leer. En el tablero también funcionan la búsqueda, la selección y edición (clic en cualquier elemento) y Ctrl+Z, y hay botones para crear: «+ función» en cada capacidad, «+ capacidad» en cada tarjeta y «+ nueva categoría» al final del tablero. La vista es local a tu pantalla; el PNG y reacomodar viven en la vista de red.
- **✨ reacomodar:** limpia los desplazamientos manuales de todos los nodos y rebalancea las ramas entre ambos lados para que el mapa quede simétrico y legible (el acomodo base es algorítmico). Afecta a todos; se puede revertir con Ctrl+Z.

## Arquitectura

| Pieza | Servicio |
|---|---|
| Página (este repo, `index.html`) | GitHub Pages |
| Estado compartido + historial de versiones | Supabase (proyecto `mapa-red-funciones-ia`, ref `lapdnydlalrbpdaphfpz`) |

El estado vive en la tabla `map_state` (una fila) y cada guardado registra una copia en `map_versions` (se conservan las últimas 200). La escritura pasa por la función RPC `save_map`; las tablas solo permiten lectura anónima.

## Restaurar una versión anterior

Si alguien borra algo por accidente, en el editor SQL de Supabase:

```sql
-- ver el historial
select id, saved_at from map_versions order by id desc;

-- restaurar una versión (reemplaza 42 por el id deseado)
update map_state
   set state = jsonb_set((select state from map_versions where id = 42),
                         '{savedAt}', to_jsonb((extract(epoch from now())*1000)::bigint))
 where id = 1;
```

## Mantenimiento

- El plan gratuito de Supabase pausa proyectos tras ~1 semana sin actividad. El workflow [keepalive.yml](.github/workflows/keepalive.yml) hace **una escritura diaria** en la tabla `heartbeat` (vía la RPC `keepalive_ping`) — actividad real de base de datos, no solo lecturas — con reintentos. Además, si el repo lleva 25+ días sin commits, el propio workflow hace un commit de latido para que GitHub no desactive el cron (los desactiva tras 60 días de repos inactivos). Si aun así llegara a pausarse, se reactiva en un clic desde el dashboard de Supabase sin pérdida de datos.
- Si el mapa aparece vacío o sin poder guardar, revisa que el proyecto de Supabase esté activo en el dashboard.

## Notas técnicas

`index.html` es el export autocontenido del editor original con tres parches aplicados:

1. `EVENT_MAP` del runtime: se agregaron los eventos `pointer*` (el export original los perdía al serializar los atributos en minúsculas, lo que dejaba muerta la edición por clic y arrastre).
2. Persistencia: `localStorage` sigue como caché local, pero la fuente de verdad es Supabase (`cloudLoad` al abrir + sondeo cada 12 s, `cloudPush` con debounce de 1.2 s tras cada cambio).
3. «restablecer» pide confirmación porque ahora afecta a todos los usuarios.
