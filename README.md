# Rutina · PWA de registro de entrenamiento

App web standalone para registrar rutinas de gimnasio. Se instala como PWA
(Android/iOS/desktop), funciona offline, y guarda en LocalStorage con
sincronización opcional a Google Sheets.

**Live:** https://juank66-cell.github.io/Rutina-Gim/

## Features

- **Multi-rutina**: creá, editá, duplicá, importá y exportá rutinas. Vienen
  pre-cargadas 3 plantillas (Mesociclo Hipertrofia 5sem, PPL 6d, Full Body 3d).
- **Editor completo**: meta, semanas (RIR objetivo, carga, deload), días, ejercicios.
- **Librería de ejercicios**: 60+ ejercicios pre-cargados, reutilizables entre rutinas.
- **Registro por serie**: peso, reps, RIR, autocompletado de la serie anterior.
- **Timer de descanso flotante**: con beep, vibración, +15/-15s, pausa.
- **Volumen semanal**: panel con series completadas vs. objetivo MEV-MAV por grupo.
- **Wellness pre-entreno**: sueño, energía, apetito, dolor de hombro, peso.
- **Variantes en semanas pares**: rotación automática de ejercicios.
- **Indicadores de hombro**: alertas y flag visual para ejercicios con cuidado especial.
- **Export/Import JSON**: completo o por rutina individual.
- **Sync opcional a Google Sheets**: vía Apps Script, multi-rutina con `routine_id` y
  `routine_name` para análisis pivotado.
- **PWA installable**: con Service Worker, manifest, íconos.

## Stack

HTML + CSS + JS vanilla en un único `index.html` (~5300 líneas). Sin frameworks,
sin bundler, sin Node. Diseñado para deploy directo en GitHub Pages.

Archivos:
- `index.html` — la app completa
- `sw.js` — Service Worker para offline
- `manifest.json` — manifest PWA
- `icon-192.png` / `icon-512.png` — íconos

## Storage

LocalStorage:
- `rutina_routines_v2` — definiciones de todas las rutinas + `activeRoutineId`
- `rutina_logs_v2` — sesiones registradas por rutina
- `rutina_exercise_library_v1` — librería global de ejercicios
- `rutina_config_v1` — config de Sheets (URL, token, scope)
- `rutina_juank_v1` — backup legacy v1 (intacto, no se modifica)

Migración automática v1 → v2 al primer boot.

## Sync con Google Sheets (opcional)

1. Crear Sheet nuevo → Extensiones → Apps Script.
2. Botón "Sheets" en la app → expandir "Ver script a copiar" → "Copiar script".
3. Pegar en el editor de Apps Script (sobrescribiendo el código por defecto).
4. Cambiar `cambia-este-token` por un token propio.
5. Guardar → Desplegar → Nueva implementación → Aplicación web.
   - Ejecutar como: Yo
   - Quién tiene acceso: **Cualquier usuario** (crítico, sino falla con CORS)
6. Copiar la URL `/exec` y pegarla en la app + token. Probar conexión.

Estrategia: cada sync envía todos los logs del atleta (todas las rutinas) y el
script borra+reescribe las filas de ese atleta para mantener consistencia con
el LocalStorage. Las filas de OTROS atletas se preservan. Toggle "solo la rutina
activa" preserva las demás rutinas del mismo atleta.

Si modificás el script: **Administrar implementaciones → lápiz → Nueva versión**
(no "Nueva implementación", eso cambia la URL).

## Deploy en GitHub Pages

```bash
# En la raíz del repo:
git add index.html sw.js manifest.json icon-192.png icon-512.png
git commit -m "Update PWA"
git push
# GitHub Pages detecta el cambio y redeploya en ~1 min.
```

Cada cambio en `index.html` o `sw.js` requiere subir `CACHE_VERSION` en `sw.js`
(ej: `rutina-v10` → `rutina-v11`) para que el Service Worker invalide la caché
y los usuarios vean los cambios al recargar.

## Tests

Los tests viven en `/home/claude/work/test_*.js` (no se incluyen en el deploy).
Son scripts de Node.js que ejecutan el JS de `index.html` en un VM context
con DOM mockeado. Cubren ~112 asserts en 9 archivos.

Para correrlos:
```bash
# Extraer JS del HTML primero
python3 -c "import re; print(max(re.findall(r'<script>(.*?)</script>', open('index.html').read(), re.DOTALL), key=len))" > extracted.js
# Correr todos
for t in test_*.js; do node "$t"; done
```

## Estructura de datos

### Rutina (`routinesStore.routines[id]`)
```js
{
  meta: { nombre, atleta, objetivo, split, duracion_semanas, lesion_hombro, notas },
  semanas: [{ n, fase, rir, carga, series_extra, deload }],
  dias: [{
    id, nombre, foco,
    ejercicios: [{
      id, codigo,
      // Inline (legacy):
      nombre, grupos, hombro_flag, nota, variante_semana_par,
      // O por referencia a librería (preferido):
      library_id, variante_semana_par_library_id,
      // Overrides opcionales:
      nombre_override, grupos_override, hombro_flag_override, nota_override,
      // Parámetros de rutina:
      series_base, reps, tempo, descanso
    }]
  }],
  objetivos_volumen: { "Pecho": { mev: 10, mav: 16 }, ... }
}
```

### Log de sesión (`logsStore[routineId].weeks["W{n}_{dayId}"]`)
```js
{
  week, dayId,
  started_at, finished_at,
  wellness: { sueno, energia, apetito, dolor_hombro, peso },
  exercises: { [exerciseId]: {
    sets: [{ peso, reps, rir, done, timestamp }],
    notes
  }}
}
```

## Licencia

Uso personal, sin licencia formal. Adaptable libremente.
