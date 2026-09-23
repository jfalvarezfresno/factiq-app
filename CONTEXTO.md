# FactIQ · Automatización de outsourcing

Contexto completo del proyecto. Si retomás el trabajo sin el historial del chat, leé esto primero.

---

## 1. Qué resuelve

FactIQ presenta perfiles técnicos a **4Latam**, su partner, que le acerca licitaciones de outsourcing.
El circuito manual tenía 12 pasos por búsqueda. Esta app automatiza el recorrido completo:

JD de 4Latam → análisis → candidatos de la red propia → propuesta al candidato →
CV anonimizado → postulación a 4Latam → resultado.

Histórico: ~45 licitaciones presentadas, 2 perfiles colocados (Pablo y Edgar).
El activo real que dejó eso son los **CVs y contactos** acumulados.

---

## 2. Dónde vive cada cosa

| Pieza | Dónde |
|---|---|
| Base de datos | Supabase, proyecto `adxuqnccpjmsbkjaxrbb` (Postgres 17, us-west-2) |
| Código de la app | Tabla `app_files` de Supabase → se publica a GitHub `jfalvarezfresno/factiq-app` |
| Hosting | **Cloudflare Pages** → https://factiq-app.pages.dev, con dominio propio outsourcingapp.fact-iq.com (CNAME desde cPanel). Netlify quedó atrás: agotó su plan gratuito |
| Archivos de origen | OneDrive de `info@fact-iq.com`, carpeta "FactIQ - Consultoría en Data" |
| CVs de postulantes | Supabase Storage, bucket privado `cvs` |
| Plantilla del CV anonimizado | Tabla `plantillas`, fila `cv_anonimo` (docx en base64) |

### Cómo se publica un cambio

1. Editar `app_files` (con `app_patch(path, viejo, nuevo)` para cambios puntuales).
2. Validar: función `validar-app` compila el JS sin ejecutarlo.
3. `select publicar('mensaje')` → hace commit en GitHub → el hosting despliega solo.

`app_patch` falla si el texto viejo no aparece exactamente una vez: es a propósito.

---

## 3. Los datos

Ingesta hecha desde OneDrive en septiembre de 2026, con un pipeline en Google Colab
(el usuario solo tiene la PC del trabajo, que bloquea instalaciones).

- 1.587 archivos inventariados, 451 analistas únicos tras deduplicar.
- Fuentes: 610 CVs, 84 pricings, 196 matrices, y el `Analysts.xlsx` con 175 perfiles.
- 2.041 requisitos de 36 licitaciones históricas en `bid_requirements`.

**Cobertura:** 397 identificados con nombre y apellido · 241 con contacto ·
362 con stack · 276 con precio · 168 completos.

### Cómo se unificaron las identidades

El problema: la misma persona aparecía como mail en un CV, nombre completo en un pricing,
y solo nombre de pila en los CVs anonimizados.

1. Agrupar por mail (único y confiable).
2. Los CVs anonimizados se cruzaron comparando **el texto del CV** (los anonimizados son
   copias del original sin contacto): similitud ≥ 0,80 = misma persona.
3. Nombres completos idénticos = misma persona.
4. Nombres de pila repetidos con CV distinto = personas distintas, se dejan separadas.

### Precedencia entre fuentes

| Dato | Gana | Después |
|---|---|---|
| Rate / costo | Pricing del proyecto (el más reciente) | Rate declarado en entrevista |
| Años por tecnología | Matriz | CV |
| Experiencia y stack | CV | Analysts.xlsx |
| Contacto y notas | Analysts.xlsx | CV |

La tabla `analysts` es una **proyección** calculada desde `extracted_facts`, que guarda cada
dato con su documento de origen. Cambiar una regla de precedencia = reproyectar, no reprocesar.

---

## 4. Reglas de negocio

- **Mes = 160 horas.** Todo rate mensual es `hora × 160`.
- **Al candidato** se le ofrece entre 200 y 500 USD/mes menos de lo que pide (regla del
  instructivo de recruiting). La app sugiere 300 menos, editable.
- **A 4Latam** se cotiza con dos opciones: margen fijo de 1.000 USD/mes, o el margen
  histórico de FactIQ, 45,5%. El promedio real de 216 cotizaciones fue 1.099 USD/mes.
- **Anonimización:** nombre de pila + inicial del apellido, sin mail, teléfono, LinkedIn ni
  dirección. Las empresas donde trabajó **sí** se mantienen.
- El costo para calcular margen es **lo que el candidato aceptó**, no lo que pidió.

---

## 5. Tablas principales

- `analysts` · la red. Proyectada, no se edita a mano.
- `analyst_technologies` / `analyst_roles` / `analyst_languages` · relaciones.
- `analyst_rate_history` · precios por proyecto, con fecha.
- `bids` · las búsquedas (X47, X48...). `bid_requirements` · sus skills.
- `postulaciones` · quién se postuló por el formulario público.
- `bid_submissions` · qué perfil se envió a 4Latam, con costo, precio, margen y resultado.
- `extracted_facts` + `source_documents` · trazabilidad de la ingesta.
- `app_files` · el código de la app. `plantillas` · el docx modelo.
- `app_users` · roles ADMIN y RECRUITER.

**Vistas:** `v_analistas`, `v_analistas_recruiter` (sin precio de venta), `v_proyectos`,
`v_postulaciones`, `v_envios`, `v_licitaciones`.

---

## 6. Roles y seguridad

- **ADMIN** (Juan y su socio): ve todo, incluidos precios de venta y márgenes.
- **RECRUITER** (analista de HR): ve candidatos y su costo, **no** el precio de venta ni el margen.

Las restricciones están en la base con RLS, no solo en la pantalla. Un usuario nuevo se
da de alta solo como RECRUITER; promoverlo a ADMIN es un cambio manual, a propósito.

Funciones protegidas (solo ADMIN): `eliminar_proyecto`, `eliminar_envio`.
El postulante nunca tiene permiso sobre el bucket de CVs: todo pasa por la función `subir-cv`.

---

## 7. Funciones de servidor (Supabase Edge Functions)

| Función | Qué hace |
|---|---|
| `ia` | Analiza el JD, redacta mensajes, consultas a 4Latam y posts de LinkedIn |
| `subir-cv` | Recibe el CV del postulante, lo guarda y lo lee para autocompletar el formulario |
| `procesar-cv` | Al entrar una postulación con CV, enriquece el perfil del analista |
| `cv-anonimo` | Reformatea un CV al formato estándar de FactIQ (devuelve JSON) |
| `cv-word` | Arma el .docx anonimizado usando la plantilla real como molde |
| `publicar` | Hace commit de `app_files` en GitHub |
| `validar-app` | Compila el JS de la app sin ejecutarlo, para no publicar algo roto |

Todas usan el modelo `claude-haiku-4-5-20251001`. Costo aproximado: 1 centavo por CV leído,
2 por CV anonimizado generado.

---

## 8. Configuración externa

- **Entra ID**, app "FactIQ Ingesta CVs": client `fc3730c2-621e-4301-81bb-2854954a1116`,
  tenant `f5a0c69f-356d-4eef-beb4-f3a38045e02c`. Permisos delegados: `Files.Read.All`,
  `User.Read`, `Mail.ReadWrite`. Requiere plataforma SPA con la dirección del sitio.
- **Secretos** en Supabase Edge Functions: `ANTHROPIC_API_KEY`, `RESEND_API_KEY`.
- **Vault** de Supabase: `github_token` (permiso de escritura solo sobre `factiq-app`).
- **Contactos de 4Latam:** marceloeduardo.arrabal@4latam.com y rocio.romero@partner.4latam.com.
  Copia fija en todo mail: info@fact-iq.com y a.stiletano@fact-iq.com.

---

## 9. Decisiones tomadas y por qué

- **Supabase no puede servir HTML.** Fuerza `text/plain` y una CSP restrictiva, tanto en
  funciones como en Storage. Por eso el hosting es externo.
- **El código vive en la base, no en el repo.** Permite cambios quirúrgicos con `app_patch`
  sin reenviar el archivo entero. El repo es el destino de publicación.
- **El CV anonimizado parte del CV original**, no de los datos estructurados: la experiencia
  laboral detallada (empresas, fechas, logros) no está en la base.
- **Se usa la plantilla real como molde**, no una imitación: conserva logo, estilos y numeraciones.
- **Al borrar un proyecto, los analistas quedan.** Los proyectos van y vienen; la red es el activo.
- **LinkedIn:** solo se genera el texto del post para copiar y pegar. Extraer perfiles
  automáticamente viola sus términos y arriesga el bloqueo de la cuenta.
- **El mail nunca se envía solo.** Se crea un borrador o se abre el redactor; siempre revisa una persona.
- **La ventana de login de Microsoft debe abrirse pegada al clic**, antes de cualquier espera,
  o el navegador la bloquea.

---

## 10. Pendiente

- Dominio propio (`app.fact-iq.com`) para que los links no dependan del proveedor.
- Traer las licitaciones abiertas desde el portal de 4Latam (el dueño ya dio el visto bueno;
  conviene pedirle un export o API en vez de scrapear).
- 4 documentos del drive que no se pudieron leer, de 1.587.
- El resultado histórico (`won`) quedó anulado: el extractor marcó 108 ganadores cuando
  fueron 2. El dato real está en el Tracker de licitaciones, sin procesar.
- WhatsApp para contactar candidatos: requiere alta en Meta y plantillas aprobadas.
