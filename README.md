# Red de Funciones IA — Mapa editable compartido

Mapa interactivo de la taxonomía funcional de la IA (categorías → capacidades → funciones específicas), editable por cualquier persona con el enlace.

**Mapa en vivo:** https://edreirbs.github.io/mapa-red-funciones-ia/

## Cómo funciona

- **Ver:** abre el enlace. Siempre carga la última versión guardada.
- **Editar:** clic en cualquier texto para renombrarlo; arrastra los nodos para moverlos; selecciona un nodo y usa «+» para agregar categorías/capacidades/funciones o «eliminar» para quitarlas.
- **Guardado automático:** cada cambio se guarda solo (≈1 segundo después de editar) en la base de datos compartida. No hay botón de guardar.
- **Sincronización:** si alguien más edita mientras tienes el mapa abierto, verás sus cambios en unos 12 segundos.
- **⤓ imagen PNG:** descarga la versión actual del mapa como imagen en cualquier momento.
- **⤓ respaldo HTML:** descarga una copia autónoma del mapa con el estado actual incrustado (útil como respaldo offline).
- **↺ restablecer:** ⚠️ regresa el mapa a su versión original **para todos** (pide confirmación).

## Arquitectura

| Pieza | Servicio |
|---|---|
| Página (este repo, `index.html`) | GitHub Pages |
| Estado compartido + historial de versiones | Supabase (proyecto `mapa-red-funciones-ia`, ref `lapdnydlalrbpdaphfpz`) |

El estado vive en la tabla `map_state` (una fila) y cada guardado registra una copia en `map_versions` (se conservan las últimas 50). La escritura pasa por la función RPC `save_map`; las tablas solo permiten lectura anónima.

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

- El plan gratuito de Supabase pausa proyectos tras ~1 semana sin actividad. El workflow [keepalive.yml](.github/workflows/keepalive.yml) hace un ping dos veces por semana para evitarlo. GitHub desactiva los crons si el repo no tiene actividad por 60 días — si llega un correo de "scheduled workflow disabled", reactívalo desde la pestaña Actions.
- Si el mapa aparece vacío o sin poder guardar, revisa que el proyecto de Supabase esté activo en el dashboard.

## Notas técnicas

`index.html` es el export autocontenido del editor original con tres parches aplicados:

1. `EVENT_MAP` del runtime: se agregaron los eventos `pointer*` (el export original los perdía al serializar los atributos en minúsculas, lo que dejaba muerta la edición por clic y arrastre).
2. Persistencia: `localStorage` sigue como caché local, pero la fuente de verdad es Supabase (`cloudLoad` al abrir + sondeo cada 12 s, `cloudPush` con debounce de 1.2 s tras cada cambio).
3. «restablecer» pide confirmación porque ahora afecta a todos los usuarios.
