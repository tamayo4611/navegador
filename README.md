# Explorador web para Azure Blob Storage

Reemplaza el uso manual de Azure Storage Explorer por una página web propia:
navegar carpetas, arrastrar una carpeta para subirla (conservando la
estructura, para que tu Function de blob trigger se siga disparando igual),
descargar y eliminar.

## Arquitectura y por qué

| Pieza | Qué hace | Por qué así |
|---|---|---|
| **Function App existente** (Python) | Se le agrega UN endpoint nuevo `GET /api/sas` | Reutiliza lo que ya tienes; no crea infraestructura nueva |
| **Managed Identity** en esa Function App | Autoriza al endpoint a generar tokens sin guardar ninguna clave | La account key nunca se guarda en ningún lado ni viaja al navegador |
| **SAS de delegación de usuario** | Token temporal (1 hora) con permisos read/write/delete/list, generado bajo demanda | Si se filtra, expira solo; nunca es la clave maestra de la cuenta |
| **Azure Static Web Apps** (plan gratis) | Aloja `explorador.html` y exige inicio de sesión (Entra ID) antes de mostrar la página | Nadie llega ni a ver el botón de "subir" sin loguearse con una cuenta de tu organización |
| **explorador.html** | Toda la lógica de listar/subir/descargar/eliminar, hablando directo con Blob Storage vía REST + el SAS token | Sin frameworks ni build step: un solo archivo que puedes editar con cualquier editor |

Se investigaron dos alternativas antes de esta: (a) usar la API "Managed
Functions" integrada de Static Web Apps — descartada porque **no soporta
Managed Identity para acceder a Storage**, solo para Key Vault; y (b) usar el
SDK oficial `@azure/storage-blob` en el navegador — descartada porque
Microsoft recomienda usarlo solo con un bundler (Vite/webpack), lo que
agrega un paso de build innecesario para este caso. Por eso el HTML llama
directamente a la API REST de Blob Storage con `fetch`/`XMLHttpRequest`.

## Archivos de este proyecto

- `agregar_a_function_app.py` — código para pegar en tu Function App existente
- `explorador.html` — la interfaz completa (edítalo y súbelo tal cual)
- `staticwebapp.config.json` — exige login antes de servir cualquier página
- Este README

## Pasos de despliegue

### 1. Habilitar Managed Identity en tu Function App

Portal → tu Function App → **Identity** → pestaña **System assigned** → **On** → Guardar.

### 2. Darle permiso sobre el Storage Account

Portal → tu Storage Account → **Access Control (IAM)** → **Add role
assignment** → rol **Storage Blob Data Contributor** → asignar a tu Function
App (búscala por nombre en "Managed identity").

### 3. Agregar el endpoint `/api/sas`

Copia el contenido de `agregar_a_function_app.py` dentro de tu
`function_app.py` existente (no borres tu función de blob trigger, solo
añade imports que falten y la función `get_sas` al final). Si usas el
modelo v1 (carpetas con `function.json`), la nota al final de ese archivo
explica el equivalente.

Agrega a `requirements.txt` de tu Function App, si no están:
```
azure-identity
azure-storage-blob
```

Despliega este código de la misma forma en que ya despliegas tu Function
App hoy (ya que está en producción, seguramente ya tienes un método:
VS Code, un pipeline, etc.). Si no tienes ninguno y quieres algo 100%
manual sin CLI: Portal → tu Function App → **Deployment Center** → conecta
el mismo repositorio de GitHub del paso 7 (puede vivir en una carpeta
`/api` dentro de ese repo, o en uno aparte) → Azure genera otro GitHub
Action que despliega el código cada vez que subas cambios por la web.

### 4. Configurar Application Settings

Portal → tu Function App → **Configuration** → **Application settings** → agregar:
- `STORAGE_ACCOUNT_NAME` = nombre de tu cuenta (sin `.blob.core.windows.net`)
- `CONTAINER_NAME` = el contenedor donde subes archivos hoy

Despliega la Function App con el nuevo código.

### 5. CORS en la Function App

Portal → tu Function App → **CORS** → agrega el dominio que te dará Static
Web Apps en el paso 7 (ej. `https://tu-app.azurestaticapps.net`). Puedes
usar `*` mientras pruebas, pero restringe al dominio real antes de dejarlo
en producción.

### 6. CORS en el Storage Account (paso que se olvida seguido)

Portal → tu Storage Account → **Resource sharing (CORS)** → pestaña
**Blob service** → agrega una regla:
- Allowed origins: `https://tu-app.azurestaticapps.net`
- Allowed methods: `GET, PUT, DELETE, HEAD, OPTIONS`
- Allowed headers: `*`
- Exposed headers: `*`
- Max age: `3600`

Sin esto, el navegador bloqueará las llamadas directas a Blob Storage
aunque el SAS token sea válido.

### 7. Crear el Static Web App y desplegar (sin CLI)

Static Web Apps **no tiene botón de "subir archivo" en el Portal** — solo
despliega desde un repositorio (GitHub/DevOps) o con su CLI. Elegir
"Other" como fuente al crearlo solo evita que Azure genere el workflow
automático, pero igual necesitas algo que suba el contenido después; no
hay una opción intermedia. Para quedarte 100% manual, sin terminal, usamos
GitHub como intermediario — subir un archivo a GitHub también es
arrastrar y soltar desde el navegador, no requiere instalar nada:

**a. Crear el repositorio**
1. Si no tienes cuenta, crea una gratis en [github.com](https://github.com)
2. **New repository** → nómbralo, ej. `explorador-blob` → puede ser privado → **Create repository**
3. Dentro del repo: **Add file** → **Upload files** → arrastra `explorador.html`
   (súbelo y luego renómbralo a `index.html` con el lápiz de editar, o
   descárgalo con ese nombre antes de subirlo) y `staticwebapp.config.json`
   → **Commit changes**

**b. Crear el Static Web App conectado a ese repo**

Portal → **Create a resource** → **Static Web App**:
- **Resource Group**: el mismo de tu Function App y Storage Account
- **Name**: el que quieras
- **Plan type**: **Free**
- **Source**: **GitHub** → inicia sesión y autoriza a Azure → elige tu organización, el repositorio y la rama (`main`)
- **Build details** → **Build Presets**: **Custom**
  - **App location**: `/`
  - **Api location**: *(déjalo vacío — la API vive en tu Function App aparte)*
  - **Output location**: *(déjalo vacío — no hay paso de build)*
- **Review + create** → **Create**

Azure crea automáticamente un GitHub Action dentro de tu repositorio y lo
ejecuta solo; en 1-2 minutos el sitio queda publicado. De ahí en adelante,
cada vez que subas un archivo nuevo por la web de GitHub, se vuelve a
desplegar automáticamente — sigue sin requerir CLI.

**Alternativa más simple pero menos segura:** si prefieres evitar GitHub
por completo, puedes alojar el HTML con **Static website** en un Storage
Account (Portal → tu cuenta → **Static website** → **Enabled**, y luego
arrastras el archivo directo en el contenedor `$web` desde **Storage
browser**, ambos en el Portal). Es más simple, pero esa página queda
**pública para cualquiera con el link** — no hay login de Entra ID como en
Static Web Apps, así que la function key quedaría expuesta a quien
encuentre la URL. Solo la recomiendo si además restringes el acceso por
red (IP/VPN) en la cuenta de almacenamiento, o si el contenido no incluye
borrado y no te preocupa que sea de solo lectura para terceros.

### 8. Obtener la function key y editar `CONFIG`

Portal → tu Function App → función `get_sas` → **Function Keys** → copia
`default`. Abre `explorador.html` y completa al inicio del `<script>`:

```js
const CONFIG = {
  accountName: 'tu_storage_account',
  containerName: 'tu_contenedor',
  sasEndpoint: 'https://tu-function-app.azurewebsites.net/api/sas',
  sasFunctionKey: 'la_clave_que_copiaste'
};
```

Vuelve a desplegar el HTML con estos valores.

### 9. Probar

Abre la URL de tu Static Web App: debe pedirte iniciar sesión (Entra ID),
y luego mostrar el contenido del contenedor. Arrastra una carpeta de
prueba a la página.

## Nota sobre seguridad de la function key

La clave queda visible en el código fuente del HTML (cualquiera con acceso
a la página puede verla en las herramientas de desarrollador). La
protección real es que **Static Web Apps exige iniciar sesión antes de
servir la página** — solo gente con cuenta en tu organización llega a ver
esa clave. Si más adelante quieres una protección más estricta (por
ejemplo, validar el usuario logueado dentro de la propia función), se
puede agregar leyendo el header `x-ms-client-principal` que Static Web
Apps inyecta automáticamente; con gusto lo agrego si lo necesitas.

## Limitaciones conocidas

- No hay botón de "crear carpeta vacía": en Blob Storage las carpetas no
  existen realmente, aparecen solas cuando subes el primer archivo dentro.
- Los archivos se suben en una sola petición PUT; para archivos
  individuales mayores a ~256 MB conviene subir por bloques (`Put
  Block`/`Put Block List`) — lo puedo agregar si manejas archivos así de
  grandes.
- El SAS generado da permisos sobre **todo el contenedor**. Si quieres
  limitar a una subcarpeta específica (por ejemplo, solo la carpeta que
  dispara tu Function), lo ajustamos en `generate_container_sas` o
  restringiendo qué `prefix` acepta el propio endpoint.
