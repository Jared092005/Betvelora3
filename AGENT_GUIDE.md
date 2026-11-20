# Guía del proyecto para agentes de IA

## 1. Qué es este proyecto

Betvelora es una aplicación web de casino construida con React 19, JavaScript,
Vite 8, React Router 7 y Tailwind CSS 4. La interfaz está en inglés por defecto
y tiene traducciones al español. El catálogo de juegos consume una API externa;
varias pantallas de cuenta y autenticación son actualmente maquetas de interfaz
y no deben tratarse como operaciones conectadas al servidor.

## 2. Ejecutar y validar

Requiere Node.js y npm. Desde la raíz:

```sh
npm install
npm run dev
npm run lint
npm run build
npm run preview
```

Scripts definidos en `package.json`:

- `dev`: servidor de desarrollo de Vite.
- `lint`: ESLint sobre el proyecto.
- `build`: compilación de producción.
- `preview`: servidor local para revisar la compilación.

No hay un script de pruebas automatizadas definido en `package.json`.

## 3. Variables de entorno e integración

Configurar las variables en un `.env` local (no guardar credenciales ni secretos
en Git):

| Variable | Uso |
|---|---|
| `VITE_API_BASE_URL` | URL absoluta base de la API de catálogo; es necesaria para las funciones de `src/api/games.js`. |
| `VITE_API_BRAND` | Cabecera `Brand` enviada por el cliente API. Valor por defecto: `12`. |
| `VITE_GAMES_API_URL` | Endpoint opcional del componente genérico `GameGrid`; si no se define, intenta usar `/games.json`. |

`GameGrid` es una integración genérica separada del cliente de catálogo. El
fallback `/games.json` no aparece entre los archivos actuales de `public/`.
Antes de usar ese componente, configurar un endpoint válido o añadir el fixture
correspondiente.

### Cliente y respuesta API

Usar `apiFetch` desde `src/api/client.js` para las llamadas al backend de
catálogo. Forma general:

```js
apiFetch('/games/category/casino', {
  params: { page: 1, per_page: 40 },
  signal,
})
```

- `apiFetch` acepta `method`, `params`, `body`, `auth` y `signal`.
- Añade `Accept`, `X-Requested-With` y `Brand`. Si se pasa `auth: true`, intenta
  leer el token de `localStorage.auth` y enviar autorización Bearer.
- Los cuerpos JSON se serializan; `FormData` se pasa directamente.
- Comprueba `response.ok` y `json.status === false`; devuelve `json.message`.
- Los errores HTTP/API se lanzan como `ApiError`, con `status` y `payload`
  cuando están disponibles. No convertir errores en respuestas exitosas.
- Las listas esperan que `message.items` sea un array y, opcionalmente,
  `message.pagination` contenga paginación.
- `buildUrl` agrega query params ignorando valores `null`, `undefined` y `''`.

### API pública de juegos (`src/api/index.js`)

Todas estas funciones también se exportan directamente desde
`src/api/games.js` o `src/api/normalize.js`.

| Función | Propósito y resultado |
|---|---|
| `listByCategory(category, options)` | Lista una categoría; acepta `page`, `perPage`, `search`, `signal`; devuelve `{ games, pagination }`. |
| `listByType(type, options)` | Lista por tipo con las mismas opciones y forma de respuesta. |
| `listNew(options)` | Juegos nuevos de `casino`. |
| `listBest(options)` | Juegos populares/mejores de `casino`. |
| `listByProvider(brandId, options)` | Juegos de un proveedor; por defecto `perPage: 5`. |
| `getGame(gameId, options)` | Busca y normaliza un juego por ID; devuelve juego o valor nulo si la respuesta no trae uno. |
| `listBrands(options)` | Lista todos los proveedores. |
| `listCategoryBrands(category, options)` | Lista proveedores asociados a una categoría. |
| `searchGames(query, options)` | Busca en casino, casino live, crash, virtual y poker; deduplica por ID. |
| `normalizeGame(game)` | Convierte el objeto de API al modelo usado por las tarjetas. |
| `normalizeGamePage(message)` | Normaliza `message.items` y conserva `pagination`. |
| `apiFetch(path, options)` | Cliente HTTP común, devuelve el `message` del sobre API. |
| `buildUrl(path, params)` | Construye un `URL` con query params. |

`searchGames` usa `Promise.allSettled`: devuelve resultados de categorías que
respondieron aunque otras fallen. Para flujos donde todas las categorías deben
ser obligatorias, manejar ese comportamiento explícitamente.

### Modelo normalizado de juego

`normalizeGame` produce:

```js
{
  id,          // string
  name,        // texto
  image,       // URL o null
  provider,    // nombre o null
  providerId,  // ID o null
  type,        // tipo en mayúsculas o null
  category,    // tipo en minúsculas o null
  badge,       // 'LIVE' para LIVE, si no null
  raw,         // objeto original de la API
}
```

Preservar este contrato al modificar el catálogo; `GameCard`, los filtros y la
página de juego lo consumen.

### Caché de catálogo

`src/api/cache.js` implementa caché en memoria (no sobrevive a recarga):

- listas de invitados: 15 minutos;
- listas con `localStorage.auth`: 3 minutos;
- proveedores: 45 minutos;
- juego individual: 5 minutos;
- búsquedas: 2 minutos.

Los valores se cachean por clave de consulta. Si se agrega una mutación o una
acción que altera datos del catálogo, revisar si hay que invalidar con
`clearCache(prefix)`.

## 4. Enrutamiento y estructura

La aplicación arranca en `src/main.jsx` con `StrictMode`, `HashRouter`,
`LanguageProvider` y el componente `App`. En producción las rutas se resuelven
con `HashRouter` (URLs del estilo `/#/casino`). Vercel reescribe las solicitudes
a `index.html`.

| Ruta | Pantalla |
|---|---|
| `/` | Inicio con carruseles de juegos y búsqueda. |
| `/home` | Redirección a `/`. |
| `/casino` | Catálogo casino; acepta `?type=`, `?filter=best`, `?filter=new` y `?brand=`. |
| `/crash-games` | Juegos crash. |
| `/casino-live` | Catálogo live. |
| `/poker` | Catálogo de poker. |
| `/virtual` | Catálogo virtual. |
| `/promotions` | Promociones públicas con pestañas locales. |
| `/games/:gameId` | Ficha y juegos del mismo proveedor. |
| `/games` | Redirección a `/casino`. |
| `/auth/login`, `/auth/register`, `/auth/forgot-password` | Pantallas de acceso. |
| `/login`, `/register` | Redirecciones a rutas `/auth/...` antiguas. |
| `/account` | Dashboard de cuenta. |
| `/account/deposit`, `/account/withdrawal`, `/account/transactions` | Secciones de cuenta. |
| `/account/promotions`, `/account/verify-account`, `/account/responsible-gambling` | Secciones de cuenta. |
| `/account/message` | Mensajes. |
| `/account/messages` | Redirección a `/account/message`. |
| `/account/personal-information` | Información personal. |
| Cualquier otra ruta | `NotFound`. |

`Layout` muestra cabecera, pie y navegación inferior para páginas normales. Los
flujos `/games/...` y `/auth/...` ocultan esa navegación. El área `/account`
conserva el layout global.

## 5. Hooks y estado de catálogo

### `useGames(key, load, { limit })`

En `src/hooks/useGames.js`. Carga la página inicial al montar/cambiar `key`,
aborta llamadas previas al desmontar y deduplica juegos por ID. `load` recibe
`{ signal, page }`; puede devolver un array o `{ games, pagination }`.

Devuelve:

```js
{ games, status, error, retry, pagination, hasMore, loadingMore, loadMore }
```

`status` es `loading`, `ready` o `error`. Usar `retry` para volver a consultar.
`loadMore` carga otra página según `pagination`. El `limit` limita la lista
inicial, no es un paginador visual.

### `useGameBrowser(options)`

En `src/hooks/useGameBrowser.js`. Hook de catálogo con filtros por categoría,
tipo, popularidad, novedades y proveedor, y con scroll infinito. Opciones:

- `category` (requerida): categoría a cargar;
- `baseMode` (por defecto `category`);
- `baseValue`;
- `brandMode` (`server` por defecto o `client`);
- `initialVisible` (por defecto 12).

Devuelve `{ games, status, error, retry, hasMore, loadingMore, loadMore,
filter, brand, select, selectBrand }`. Las páginas de catálogo envían estos
datos a `GameFilters`, `GamesGrid` e `InfiniteScrollSentinel`.

La paginación del backend puede repetir elementos: el hook deduplica por ID y
puede saltar páginas que no agregan juegos nuevos. `crash` y `virtual` filtran
proveedores en cliente; el resto usa la consulta por proveedor del servidor.

## 6. Catálogo de funciones por archivo

Este inventario incluye funciones y componentes nombrados, incluidos helpers
privados. Las lambdas anónimas usadas directamente en JSX son detalles internos;
su comportamiento se describe junto al componente que las contiene.

### Aplicación y API

- `src/App.jsx`: `App` define rutas, layouts y redirecciones.
- `src/api/client.js`: `ApiError`, `readToken` (privada),
  `buildUrl(path, params)`, `apiFetch(path, options)`.
- `src/api/cache.js`: `getCached(key)`, `setCached(key, value, ttlMs)`,
  `clearCache(prefix?)`.
- `src/api/games.js`: `sessionTtl` (privada), `cached` (privada),
  `listByCategory`, `listByType`, `listNew`, `listBest`, `listByProvider`,
  `getGame`, `listBrands`, `listCategoryBrands`, `searchGames`.
- `src/api/normalize.js`: `normalizeGame`, `normalizeGamePage`.
- `src/api/index.js`: barrel de las funciones públicas de API.

### Hooks e idioma

- `src/hooks/useGames.js`: `dedupeById` (privada), `useGames`; `run` es la
  rutina interna de carga.
- `src/hooks/useGameBrowser.js`: `dedupeById` (privada), `useGameBrowser`;
  `loadPage` y `run` son rutinas internas de carga.
- `src/i18n/context.js`: `useLanguage`; falla explícitamente si se usa fuera
  de `LanguageProvider`.
- `src/i18n/LanguageProvider.jsx`: `readStoredLanguage` (privada),
  `LanguageProvider`.
- `src/i18n/applyTranslations.js`: `insideNoTranslate` (privada),
  `applyTranslations(language)`.
- `src/i18n/dictionary.js`: exporta el diccionario `translations` y los
  atributos traducibles `TRANSLATABLE_ATTRIBUTES`.
- `src/i18n/index.js`: barrel de `LanguageProvider`, `useLanguage` y
  `applyTranslations`.

### Páginas

- `src/pages/Home.jsx`: `Home`, `tr(es, en)` (helper local); búsqueda con debounce de 300 ms, catálogo
  inicial por secciones y resultados agregados con `searchGames`.
- `src/pages/Casino.jsx`: `Casino`; filtro inicial derivado de query params.
- `src/pages/CrashGames.jsx`: `CrashGames`.
- `src/pages/CasinoLive.jsx`: `CasinoLive`.
- `src/pages/Poker.jsx`: `Poker`.
- `src/pages/Virtual.jsx`: `Virtual`.
- `src/pages/Game.jsx`: `Game` y helper privado `GameScreen`; helpers internos
  `tr`, `load`, `goBack`, `toggleFullscreen`, además de listeners
  `onChange`/`onKeyDown`. Carga el juego y hasta 30 juegos del proveedor;
  admite fullscreen del navegador y fallback expandido.
- `src/pages/Promotions.jsx`: `Promotions`; las pestañas cambian estado local.
- `src/pages/NotFound.jsx`: `NotFound`.
- `src/pages/auth/Login.jsx`: `Login`.
- `src/pages/auth/Register.jsx`: `Register`.
- `src/pages/auth/ForgotPassword.jsx`: `ForgotPassword`, `handleSubmit`.
- `src/pages/account/Dashboard.jsx`: `Dashboard`.
- `src/pages/account/Deposit.jsx`: `Deposit`.
- `src/pages/account/Withdrawal.jsx`: `Withdrawal`.
- `src/pages/account/Transactions.jsx`: `Transactions`.
- `src/pages/account/Promotions.jsx`: `AccountPromotions`.
- `src/pages/account/VerifyAccount.jsx`: `VerifyAccount`.
- `src/pages/account/ResponsibleGambling.jsx`: `ResponsibleGambling`.
- `src/pages/account/Messages.jsx`: `Messages`.
- `src/pages/account/PersonalInformation.jsx`: `PersonalInformation`.
- `src/pages/index.js`, `src/pages/auth/index.js` y
  `src/pages/account/index.js`: exportaciones agrupadas de páginas.

### Componentes de juegos y proveedores

- `src/components/games/GameCard.jsx`: `GameCard({ game, badge, onPlay, to })`;
  activa un favorito local de tarjeta, muestra portada y enlaza a `to` si existe.
- `src/components/games/GameCarousel.jsx`: `GameCarousel({ games, label, onPlay })`,
  `updateEdges` y `scrollByCards(direction)`; carrusel horizontal.
- `src/components/games/GameFilters.jsx`: `GameFilters({ title, items,
  activeKey, onSelect, category, activeBrand, onSelectBrand, initialBrandId })`.
- `src/components/games/GameGrid.jsx`: `normalizePayload` (privada),
  `GameGrid({ endpoint, title, limit, query, onPlay, className })`,
  `retry` y `load` (internas). Componente autónomo que usa `fetch` directo,
  no el cliente API común.
- `src/components/games/GamesGrid.jsx`: `GamesGrid({ games, status, error,
  retry, className, skeletonCount, emptyMessage })`, `tr(es, en)` (helper local);
  presenta carga, error, vacío o tarjetas.
- `src/components/games/GamesCarousel.jsx`: `GamesCarousel({ games, status,
  error, retry, label })`, `tr(es, en)` (helper local); gestiona estados y
  delega en `GameCarousel`.
- `src/components/games/InfiniteScrollSentinel.jsx`:
  `InfiniteScrollSentinel({ hasMore, onLoad, disabled })`; llama `onLoad` al
  entrar en pantalla.
- `src/components/games/TopWinnersOfTheMonth.jsx`:
  `TopWinnersOfTheMonth({ winners, games })`, `stepWinners(direction)`.
- `src/components/provider/ProviderFilter.jsx`:
  `ProviderFilter({ category, activeBrand, onSelect, initialBrandId })` y
  `load` (interna); obtiene proveedores y aplica selección inicial.
- `src/components/provider/ProviderGamesCarousel.jsx`:
  `ProviderGamesCarousel({ games, label })`, `updateEdges`,
  `scrollByCards(direction)`.
- `src/components/games/index.js` y `src/components/provider/index.js`:
  exportaciones agrupadas.

### Inicio, layout y elementos reutilizables

- `src/components/home/HomeAside.jsx`: `HomeAside`; obtiene proveedores y
  filtra localmente el listado por texto.
- `src/components/home/HomePromos.jsx`: `promoSlideStyle` (privada),
  `HomePromos`, `goToPromo`, `handlePromoTouchStart`,
  `handlePromoTouchEnd`; carrusel de promociones con autoplay y swipe.
- `src/components/layout/Layout.jsx`: `Layout`.
- `src/components/layout/Header.jsx`: `Header` y handlers internos
  `onKeyDown`, `onPointerDown`, `onPointerMove`, `stopDrag`,
  `onClickCapture`, `onWheel`, `onDragStart`; menú móvil y navegación
  desplazable por ratón/rueda en pantallas grandes.
- `src/components/layout/MobileNavDrawer.jsx`: `MobileNavDrawer`.
- `src/components/layout/LanguageSwitcher.jsx`: `LanguageSwitcher`,
  `handlePointerDown`, `handleKeyDown`, `select`.
- `src/components/layout/ScrollToTop.jsx`: `ScrollToTop`.
- `src/components/layout/Footer.jsx`: `Footer`.
- `src/components/layout/AppBackground.jsx`: `AppBackground`.
- `src/components/layout/PageShell.jsx`: `PageShell({ title, subtitle, icon,
  children })`.
- `src/components/layout/Flag.jsx`: `Flag({ code, className })`.
- `src/components/account/AccountLayout.jsx`: `AccountLayout`.
- `src/components/account/AccountSection.jsx`:
  `AccountSection({ title, description, icon })`.
- `src/components/account/AccountSidebar.jsx`: `AccountSidebar`.
- `src/components/skeleton/Skeleton.jsx`: `Skeleton({ className })`.
- `src/components/skeleton/CardSkeleton.jsx`: `CardSkeleton({ className })`.
- `src/components/skeleton/CarouselSkeleton.jsx`:
  `CarouselSkeleton({ count })`.
- `src/components/spinners/Spinner.jsx`: `Spinner({ size, tone, className })`.
- `src/components/spinners/AppLoader.jsx`: `AppLoader`, `finish` (interna);
  loader inicial que respeta un tiempo mínimo de visualización.
- `src/components/index.js`, `src/components/layout/index.js`,
  `src/components/games/index.js`, `src/components/provider/index.js`,
  `src/components/home/index.js`, `src/components/skeleton/index.js` y
  `src/components/spinners/index.js`: barrels de componentes. Ver el archivo
  si se necesita la lista exacta de cada grupo.
- `src/main.jsx`: punto de entrada React; no declara funciones auxiliares.

## 7. Componentes reutilizables principales

Al reutilizar componentes, respetar estos contratos:

```jsx
<GameCard game={normalizedGame} to={`/games/${normalizedGame.id}`} />
<GamesGrid games={games} status={status} error={error} retry={retry} />
<InfiniteScrollSentinel hasMore={hasMore} onLoad={loadMore} />
<GameFilters
  title="Casino"
  items={filterItems}
  category="casino"
  activeKey={activeKey}
  onSelect={select}
  activeBrand={brand?.id ?? null}
  onSelectBrand={selectBrand}
/>
```

- `status` de los componentes de catálogo: `loading`, `ready` o `error`.
- `GameCard` solo marca el favorito en estado local de esa instancia. `onPlay`
  es opcional; al proporcionar `to`, la tarjeta navega a la ficha del juego.
- `InfiniteScrollSentinel` requiere `hasMore` y `onLoad`; sus páginas padres
  usan `key={games.length}` para revisar de nuevo el centinela al cambiar el
  catálogo.
- `Spinner` acepta `size`: `xs|sm|md|lg|xl`; `tone`: `gold|red|light`.
- `LanguageSwitcher` acepta `variant="compact"` (por defecto) o `"full"`.
- `GameGrid` acepta endpoint propio, límite, búsqueda y callback opcional;
  no confundirlo con `GamesGrid`, que recibe los juegos ya cargados.

## 8. Idioma y estilos

- Idiomas soportados: `en` y `es`; por defecto `en`. La preferencia se guarda
  con clave `velora:lang` en `localStorage`.
- Consumir el idioma con `useLanguage()` y cambiarlo con `setLanguage(code)`.
- Añadir traducciones al diccionario de `src/i18n/dictionary.js`. La aplicación
  también traduce texto visible del DOM al cambiar ruta/idioma; marcar contenido
  que no se debe traducir con `data-no-translate`.
- El proveedor de idioma debe permanecer bajo `HashRouter`, pues usa
  `useLocation`.
- Estilos globales en `src/index.css` y `src/screens.css`; el plugin de Tailwind
  se configura en `vite.config.js`. Mantener el vocabulario/clases existente y
  revisar el resultado responsive.

## 9. Estado funcional actual: límites importantes

No asumir que un formulario visible está conectado al backend:

- Las únicas llamadas al backend de producto encontradas son las de catálogo
  de juegos y proveedores descritas arriba.
- `Login` y `Register` son formularios visuales sin envío API ni creación de
  sesión implementados.
- `ForgotPassword` solo cambia el estado local para mostrar un mensaje; no
  envía correo.
- Las pantallas de depósito, retiro, transacciones, verificación, límites de
  juego responsable, mensajes e información personal son principalmente
  estáticas. Sus formularios/controles no guardan datos ni invocan endpoints.
- `Promotions` y las promociones de cuenta usan contenido y estado local.
- Los ganadores vienen de datos de ejemplo en `src/data/topWinners.js`; los
  identificadores/importes no se consultan al servidor.
- Los favoritos no persisten ni están asociados a una cuenta.
- La búsqueda de Inicio está conectada al catálogo; los campos de búsqueda de
  algunas páginas de categoría son actualmente solo presentación y no filtran.
- Los balances mostrados son valores fijos de maqueta.

Al implementar una de estas funciones, primero confirmar el contrato disponible
en backend; no inventar endpoints ni mostrar éxito si la operación no se realizó.

## 10. Guía de cambios para agentes

1. Identificar si el cambio es de navegación, catálogo, idioma o pantalla visual.
2. Para el catálogo, preferir `src/api/games.js` y el cliente común en vez de
   introducir nuevos `fetch` en componentes. Pasar `AbortSignal` cuando la
   operación dependa del ciclo de vida de una página.
3. Mantener el modelo normalizado y los estados de carga/error/vacío.
4. Si cambia un componente reutilizable, revisar consumidores de la misma API;
   los catálogos comparten `useGameBrowser`, `GamesGrid` y `GameFilters`.
5. Si se agrega ruta, actualizar `src/App.jsx` y, si corresponde, los enlaces
   del Header, navegación móvil y sidebar de cuenta.
6. Usar `Link`/`NavLink` para navegación SPA; recordar que el router es
   `HashRouter`.
7. Añadir textos a las traducciones si son visibles en ambos idiomas y no
   introducir datos personales, tokens o secretos en código, logs o commits.
8. Ejecutar `npm run lint` y `npm run build` para validar cambios de código.
   Para cambios solo de documentación, no hace falta ejecutar validaciones.

## 11. Mapa de carpetas

```text
src/
  api/          cliente HTTP, catálogo, caché y normalización
  components/   layout, juegos, cuenta, proveedores, loaders y skeletons
  data/         datos locales de ganadores de ejemplo
  hooks/        carga y navegación del catálogo
  i18n/         proveedor, contexto y traducciones
  pages/        rutas públicas, auth y páginas de cuenta
  assets/       recursos importados por Vite
  main.jsx      bootstrap de React
  App.jsx       definición de rutas
```
