# Inventario de identidad y plan de marca blanca

Resultado de una revisión **solo de lectura** de `rankapp` y `programacion` (copias superficiales, sin modificar nada de producción).
Objetivo: vender el producto a varias ligas con una sola base de código.

Etiquetas: **[Seguro]** comprobado en el código · **[Probable]** inferencia fuerte · **[Suposición]** sin verificar.

---

## 1. Hallazgos principales

1. **[Seguro] RankApp es de una sola liga.** Hay 254 archivos de migración y ninguno menciona `organization_id`, `league_id` ni `tenant`. No existe separación de datos por organización.
   Consecuencia: para vender, lo realista es **una instalación por liga** (un proyecto Supabase y un sitio Laravel por cliente), con el mismo código. Multi-tenant real queda para más adelante.
2. **[Seguro] La marca está parcialmente centralizada.**
   - `src/lib/branding.ts` ya concentra el pie de página, la URL de Programación y el formulario de registro.
   - `tailwind.config.js` define toda la paleta verde en un solo token `brand` (50–900). Cambiar el color de una liga es editar ese bloque.
   - `src/components/LeagueLogo.tsx` centraliza el logo, aunque con el archivo y el texto alternativo fijos.
3. **[Seguro] Hay valores de producción como valor por defecto.** Esto es el mayor riesgo al copiar:
   - `supabase/functions/_shared/mailer.ts`: `APP_URL` cae a `https://app.lattenis.com` y `SMTP_FROM_NAME` a "Liga Antioqueña de Tenis" si falta el secreto.
   - `programacion/config/services.php`: `RANKAPP_APP_URL` cae a `https://app.lattenis.com`.
   - `src/lib/branding.ts` y `_shared/whatsapp.ts`: `https://programacion.lattenis.com` escrito a mano.
   Una copia a la que se le olvide un secreto apunta a producción sin avisar.
4. **[Seguro] El pie de página acredita a una liga concreta** ("Desarrollado por Adrian Gutierrez para Liga Antioqueña de Tenis de Campo") en `branding.ts` y en `site-footer.blade.php`.
5. **[Seguro] El código declara licencia MIT** (README de `rankapp`). Para vender hay que cambiarla antes de entregar código a clientes.

## 2. Dónde está la identidad hoy

### RankApp (React + Supabase)

| Capa | Archivos | Qué cambiar |
|---|---|---|
| Marca centralizada | `src/lib/branding.ts`, `tailwind.config.js`, `src/components/LeagueLogo.tsx`, `LeagueHeaderCard.tsx` | Pasar a un único `brand.config` por cliente |
| HTML estático | `index.html` (título, descripción, Open Graph, Twitter, `og:image` con `app.lattenis.com`), `public/logo_lat.webp`, `favicon-64.png`, `apple-touch-icon.png`, `og-image.png`, `public/membrete/` | Generarlo en el build a partir de la configuración |
| Correos | `supabase/functions/_shared/mailer.ts` (nombre, dirección `notificacionestorneos@lattenis.com`, pie, banda con el escudo) | Leer nombre, ciudad y remitente de la configuración |
| Mensajes WhatsApp | `_shared/whatsapp.ts`, `whatsapp-send`, `whatsapp-webhook` | URL de Programación desde configuración |
| Textos legales | `src/pages/PoliticaDevoluciones.tsx` (17 coincidencias), páginas de checkout y convenios | Texto legal propio de cada liga |
| Certificados | `src/lib/certificatePdf.ts`, función `certificates` | Logo, firmas y textos desde configuración |
| Públicas | `PublicHome.tsx`, `PublicNavbar.tsx`, `PublicFooter.tsx`, `LeagueCalendar.tsx` | Leer nombre y logo de la configuración |
| Integraciones | `siigo-*` (facturación Siigo), `whatsapp-*`, `federation-sync`, `programacion-*`, `registration-payments` | Apagables por liga; credenciales por liga |
| Formulario externo | `REGISTRO_FORM_URL` (Google Form de la liga) | Opcional por liga |

Total aproximado: **174 coincidencias en 61 archivos** de código y documentación (excluye migraciones). Parte son pruebas y manuales.

### Programación (Laravel + Filament)

| Capa | Archivos | Qué cambiar |
|---|---|---|
| Vista pública | `resources/css/public.css` (paleta verde), `resources/views/public/index.blade.php`, `public/partials/site-footer.blade.php` | Variables CSS y textos desde `config/brand.php` |
| PDF y exportes | `resources/views/exports/*.blade.php` (grilla, programación, cuadro de honor, resumen), `app/Tennis/PygExport.php`, `GridPdf.php` | Nombre y logo desde configuración |
| Panel | `AdminPanelProvider.php` (`brandName('Torneos de tenis')`) y `JuezPanelProvider.php` | Ya es neutro; conectar a la configuración |
| Federación | `app/Tennis/Federation/FederationFetcher.php` | Revisar si depende de una federación concreta [Suposición] |
| Entorno | `.env.example`, `config/services.php`, `DESPLIEGUE.md` (43 coincidencias), `README.md` | Documentación genérica |
| Datos de demo | `database/seeders/DemoSeeder.php` | Datos ficticios por cliente |

Total aproximado: **82 coincidencias en 18 archivos**, casi la mitad en documentación.

## 3. Esfuerzo estimado

Estimaciones **[Suposición]**: no se ha medido, y dependen de cuánto se pruebe.

| Fase | Trabajo | Esfuerzo |
|---|---|---|
| A | Archivo `brand.config` y sustitución de valores fijos en RankApp (front y funciones) | 3–5 días |
| B | `config/brand.php` y sustitución en Laravel (vista pública, PDF, panel) | 1–2 días |
| C | Eliminar valores por defecto de producción: que una configuración faltante **falle** en lugar de apuntar a producción | 1 día |
| D | Script de aprovisionamiento por liga (proyecto, migraciones, funciones, marca, administrador) | 3–4 días |
| E | Plantilla de textos legales por liga y apagado de módulos por liga | 2–3 días |
| F | CI, entornos `staging` y producción, respaldos y monitoreo | 3–5 días |

## 4. Orden recomendado

1. **C primero.** Es barato y evita el accidente más grave (un entorno que escribe o envía a producción).
2. **A y B** en una rama aparte, sin tocar producción hasta pasar pruebas en `staging`.
3. **D** con el demo de la primera liga como primer cliente del proceso.
4. **E y F** antes de firmar con la segunda liga.

## 5. Reglas para esta etapa

- No se modifica `rankapp` ni `programacion` desde aquí: los cambios se hacen en copias o ramas aparte, y se revisan antes de llegar a producción.
- Ningún secreto, clave ni dato real de jugadores entra en este repositorio.
- Antes de crear proyectos Supabase o Vercel nuevos se confirma el costo.
