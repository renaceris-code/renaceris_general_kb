# PostHog: operación de soporte y observabilidad

La telemetría complementa, pero no reemplaza, `audit_logs`. PostHog no debe recibir datos de pacientes, contenido clínico, cuerpos HTTP, tokens ni archivos.

## Despliegue

### Vercel (frontend)

Configurar variables solo para Production:

- `VITE_POSTHOG_PROJECT_TOKEN` y `VITE_POSTHOG_HOST=https://us.i.posthog.com`.
- `VITE_APP_VERSION` es un override opcional; en Vercel se deriva automáticamente de `VERCEL_GIT_COMMIT_SHA`.
- `POSTHOG_PERSONAL_API_KEY`, `POSTHOG_PROJECT_ID` y `POSTHOG_CLI_HOST=https://us.posthog.com` solo para el build.
- Usar `npm run build:posthog` como build command. El comando genera mapas ocultos, inyecta el `chunkId`, los carga como release y elimina los `.map` después de una carga exitosa.

La clave personal requiere `error tracking write` y `organization read`. Nunca debe usar el prefijo `VITE_`.

### Railway (backend)

Configurar en el dashboard de Railway:

- `NODE_ENV=production`
- `POSTHOG_ENABLED=true`
- `POSTHOG_PROJECT_TOKEN`
- `POSTHOG_HOST=https://us.i.posthog.com`

`APP_VERSION` es un override opcional; en Railway se deriva automáticamente de `RAILWAY_GIT_COMMIT_SHA`.
El archivo `railway.json` configura el arranque y el healthcheck de despliegue sobre `/health`. Railway valida ese endpoint durante el despliegue, no como monitor continuo después de activarlo.

Railway debe complementarse con un monitor externo para alertas de disponibilidad. PostHog no puede detectar por sí solo una caída total del proceso.

## Dashboards y etiquetas

Crear los siguientes dashboards y aplicar las etiquetas indicadas:

| Dashboard | Etiquetas | Fuente principal |
| --- | --- | --- |
| `prod · salud-operativa` | `prod`, `operaciones`, `disponibilidad` | `backend_request_completed`, `http_request_failed` |
| `prod · soporte-medicos` | `prod`, `soporte`, `medicos` | eventos filtrados por roles `medico` y `especialista_estetico` |
| `prod · errores` | `prod`, `errores` | `$exception` por release, rol y área |
| `prod · rendimiento` | `prod`, `latencia` | p50/p95/p99 de `duration_ms` por `route_pattern` |
| `prod · flujos-criticos` | `prod`, `clinico` | `critical_flow_completed` por `operation` y `outcome` |

Todos los insights deben filtrar `environment=production`. Nunca crear filtros que dependan de nombres, correos o identificadores clínicos.

## Alertas a `#renaceris-alertas`

Conectar Slack desde PostHog y configurar:

1. Tasa de `backend_request_completed` con `http_status >= 500`: warning ≥2%/5 min con ≥20 solicitudes; critical ≥5%/5 min.
2. P95 de `duration_ms` del backend: warning ≥1500 ms/10 min; high ≥3000 ms/5 min.
3. Nueva `$exception`, o una excepción que afecte ≥5 usuarios en 10 min.
4. `critical_flow_completed` con `outcome=failure`: ≥3 para la misma `operation` en 10 min.
5. `http_request_failed`: ≥3 usuarios distintos en 5 min.

Cada alerta debe enlazar al insight, mostrar `app_version`, `role`, `feature_area`, `route_pattern` y `request_id`, y omitir propiedades de persona.

## Acceso y triage

- Ingeniería: administración, releases, error tracking y configuración.
- Soporte: dashboards y replays seguros; sin administración de proyecto.
- Admin/supervisor clínico: solo dashboard agregado de salud si el plan contratado permite permisos granulares.
- Resto de roles: sin acceso a PostHog.

Triage: localizar el evento por hora/rol/área, copiar `request_id`, revisar el error correlacionado y abrir replay únicamente si existe. Los replays se detienen fuera de login, dashboard y panel médico; aun allí todo input y texto se enmascara salvo etiquetas estáticas marcadas explícitamente con `data-record="true"`.

Antes de cada release, validar una sesión con datos ficticios y comprobar que no sean visibles nombre, DNI, email, teléfono, notas, recetas, pagos, adjuntos ni parámetros de URL.
