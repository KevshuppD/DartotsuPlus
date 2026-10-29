# brain.md — Mapa mental de DartotsuPlus

> Documento de orientación para personas e IAs. Explica **qué es el proyecto, de dónde viene y qué
> hace cada parte**. Para el detalle fino de cada subsistema, ver `CLAUDE.md` (la referencia
> técnica completa, en inglés), `CONSISTENCY.md` (reglas de UI compartida) y `MAIN_PARITY.md`
> (qué falta portar de la app antigua).

---

## 0. ⚠️ Esto es un FORK

| | |
|---|---|
| **Este repo** | `DartotsuPlus` — `https://github.com/KevshuppD/DartotsuPlus` |
| **Upstream** | `Dartotsu` de **aayush2622** — `https://github.com/aayush2622/Dartotsu` |
| **Origen del upstream** | Dartotsu es a su vez una reescritura en Flutter de **Dantotsu** (Kotlin, Android) |
| **Historia** | ~2 270 commits heredados del upstream (autores: aayush262, Ankit Grai, Itsmechinmoy, …) |
| **Paquete** | `pubspec.yaml` sigue llamándose `dartotsu`, versión `1.1.1` |

Consecuencias prácticas de ser un fork:

- **Fork independiente.** Los cambios se hacen sólo aquí; de momento no se contribuye (PRs) al
  proyecto de origen. El código heredado se puede modificar libremente. Traer cambios del upstream
  es opcional (`git remote add upstream https://github.com/aayush2622/Dartotsu.git` +
  `git fetch upstream`), y cuanto más diverja el fork, más conflictos tendrá ese merge.
- **Las dependencias git apuntan a los forks de aayush2622** (`media-kit`, `DartotsuExtensionBridge`,
  `screen_brightness`, `dart_discord_presence`, `flutter_web_auth`) y de itsmechinmoy
  (`dartotsu_image_rust`). No son de este repo; si necesitas cambiarlas, tendrás que forkearlas también.
- **Identidad heredada**: nombre de app, `applicationId`/bundle ids, esquema `dartotsu://`, URLs del
  actualizador (`lib/Api/Updater/AppUpdater.dart` busca releases de GitHub), README, badges,
  Firebase (`lib/Core/Analytics/FirebaseOptions.dart`, `firebase.json`), canales Nix
  (`channels.json`) y los **client ids OAuth** de AniList/MAL/Simkl/MangaBaka siguen siendo los
  del upstream. Si el fork se distribuye como app propia, todo esto hay que revisarlo
  (sobre todo el auto-updater: si no se cambia, actualizará a builds del upstream).
- `CLAUDE.md` describe el árbol como la rama `rewrite-re` del upstream y menciona rutas locales de
  otro desarrollador (`/home/aayush/...`). Aquí la rama es `main` y no hay worktree de la app
  antigua; las referencias a "`main` (la app vieja)" se refieren al `main` **del upstream** antiguo.

---

## 1. Qué es la app

Un **cliente de seguimiento y gestión de anime/manga/novelas/películas** multiplataforma
(Android, iOS, Windows, Linux, macOS) hecho en **Flutter/Dart**. Combina varios servicios de
tracking (AniList, MyAnimeList, Simkl, Kitsu, MangaBaka, MangaUpdates) y permite **ver y leer**
contenido mediante **extensiones de terceros** cargadas en tiempo de ejecución (no incluidas aquí).

Incluye: reproductor de vídeo (media_kit/mpv), lector de manga y novelas, descargas y modo
offline, sincronización de listas entre servicios, comentarios, red social, notificaciones,
Discord Rich Presence, casting (Chromecast/DLNA/Roku), "ver juntos" (Syncplay), widget de Android,
bloqueo con PIN/biometría y auto-actualización (se distribuye fuera de las tiendas).

---

## 2. Estructura del repo (raíz)

| Ruta | Qué es |
|---|---|
| `lib/` | Todo el código Dart (~830 archivos). Ver §3. |
| `assets/` | Imágenes, fuentes, `translations/app_*.arb` (43 idiomas), `.env` (obligatorio, contiene `hash`). |
| `android/ ios/ linux/ macos/ windows/ web/` | Proyectos nativos por plataforma (Android incluye el widget en Kotlin, PiP, teclas de volumen). |
| `test/` | Tests unitarios de lógica pura (sync, integraciones, MPRIS, scheduling…). |
| `tool/consistency_check.dart` | Script que verifica las reglas de `CONSISTENCY.md`. |
| `scripts/` | `install.sh` (instalador Linux), scripts de CI. |
| `.github/workflows/` | CI: `dart.yml` (analyze + builds según tags en el commit: `[build.apk]`, `[build.all]`…), `release.yml`, `build-*.yml`, bots de issues. |
| `flake.nix`, `nix/`, `channels.json` | Paquete Nix, módulo NixOS/Home-Manager, dev shell. |
| `fastlane/` | Metadatos de tienda (F-Droid/Obtainium). |
| `docs/` | Notas de release/beta. |
| `CLAUDE.md` / `CONSISTENCY.md` / `MAIN_PARITY.md` / `CHANGELOG.md` | Documentación interna. |
| `pubspec.yaml` | Dependencias (muchas son git forks; **cambios aquí = alto riesgo**). |

---

## 3. `lib/` — qué hace cada carpeta

### 3.1 Arranque
- **`main.dart`** — `runZonedGuarded` → `init()`: abre preferencias (Isar), inicia rhttp, `DI.init()`,
  logger, el bridge de extensiones, ventana de escritorio, media_kit; luego `runApp(MyApp)`.
  Tareas post-arranque: deep links, comprobación de updates, alertas, sync automático, etc.
- **`DI.dart`** — el único sitio donde se registran los controladores globales (`put`/`lazyPut`).
- **`Logger.dart`** — log rotativo a `appLogs.txt` (+ logcat nativo en Android).

### 3.2 `Core/` — infraestructura (sin UI de pantallas)
| Carpeta | Responsabilidad |
|---|---|
| `State/` | **Motor de estado propio**: `Live<T>` (valor reactivo), `Watch(() => widget)` (reconstruye al cambiar), `find<T>()`/`put` (localizador), `AppController`. GetX sólo existe en `GetBridge.dart` porque el bridge lo usa. **No se usa `setState`.** |
| `Preferences/` | Almacén clave-valor tipado sobre Isar (`Pref<T>`, `PrefName`, `.rx` reactivo), backup/restore cifrado, importación desde Dantotsu, rutas de disco (`StorageManager`), incógnito. |
| `ThemeManager/` | Temas (`AppTheme` enum), modo glass/OLED/Material You, tipografía, idioma (`LocaleController`, `getString`). |
| `NetworkManager/` | HTTP con **rhttp** (Rust), DNS-over-HTTPS, cookies, logs, puente de red para extensiones. |
| `Services/` | **Abstracción de servicios** (`MediaService`, `Queries`, `Mutations`, `ServiceAuth`, `*ScreenView`), modelos de dominio (`Model/`: `Media`, `Anime`, `Manga`, `Character`…), caché de secciones, `IdMapper` (traducción de ids entre servicios), integraciones externas (AnimeThemes, SauceNAO, trace.moe, AniTrendz, ISBN), login por navegador. |
| `Downloads/` | Motor de descargas: vídeo directo, HLS, torrents, capítulos de manga/novela, archivos cbz/epub, límites de red/velocidad. |
| `Offline/` | Detección de conexión, modo offline, escrituras pendientes que se reenvían al volver la red. |
| `SourceAlerts/` | Suscripciones a fuentes de extensiones → notificaciones de nuevos episodios/capítulos. |
| `Cast/` | Chromecast (CASTV2 propio), DLNA, Roku + proxy HTTP local. |
| `HomeWidget/` | Sincroniza datos con el widget de Android. |
| `NotificationManager/` | Notificaciones locales del sistema. |
| `Analytics/` | Firebase Crashlytics. |
| `Security/` | `AppLock` (PIN 4–8 dígitos) y biometría. |
| `Fonts/` | Fuentes de la app. |

### 3.3 `Api/` — backends concretos
- **`Api/Services/<Servicio>/`** — una carpeta por servicio, con el mismo patrón:
  `Client` (HTTP), `Auth` (login/tokens), `Queries` (lecturas), `Mutations` (escrituras),
  `Screens/` (implementaciones de las vistas de datos), `Sync.dart` (adaptador de sincronización),
  `Prefs.dart`, `Data/` (mapeo JSON → modelos).
  - `Anilist/` — GraphQL, el servicio más completo (social, historias, notificaciones).
  - `MyAnimeList/` — API oficial + Tenrai (datos públicos estilo Jikan).
  - `Simkl/` — películas, series y anime; OAuth PKCE; cuotas estrictas.
  - `Kitsu/` — JSON:API, login email/contraseña, social y grupos.
  - `MangaBaka/` — manga/novelas, stacks comunitarios, noticias.
  - `MangaUpdates/` — manga/novelas, login usuario/contraseña.
  - `Extension/` — "servicio" que navega directamente las extensiones instaladas.
- **`Api/Discord/`** — Rich Presence (IPC en escritorio, token en móvil).
- **`Api/Updater/`** — auto-actualizador desde GitHub Releases (⚠️ apunta al upstream).

**Regla clave**: añadir un servicio = una subclase de `MediaService` sin tocar el núcleo. Los
servicios devuelven **datos, nunca widgets**; si una vista es `null`, la pantalla muestra
`NotImplemented`.

### 3.4 `Screen/` — pantallas (UI)
Cada pantalla son **dos archivos**: `FooScreen.dart` (widgets) + `FooState.dart` (lógica, `Live`s).
Los componentes propios van en `Components/`.

| Carpeta | Pantalla |
|---|---|
| `Splash/`, `Onboarding/`, `Login/`, `Lock/` | Arranque, bienvenida, login del servicio, bloqueo PIN. |
| `MainScreen.dart`, `Navbar.dart` | Shell con pestañas dinámicas según el servicio (`[anime] Home [manga]`). |
| `Home/`, `Feed/` | Inicio (dashboard, Resume shelf) y pestañas de navegación (carrusel + estanterías). |
| `Detail/` | Ficha de un título: info, pestaña **Watch/Read** (fuentes, episodios, metadatos), **Comments**. |
| `Player/` | Reproductor (media_kit/mpv, saltos AniSkip, PiP, MPRIS, cast, shaders Anime4K). |
| `Reader/` | Lector de manga (paginado/webtoon) y novelas (TTS, traducción). |
| `Search/` | Búsqueda con filtros, búsqueda por imagen y por código de barras ISBN. |
| `MediaList/` | Editor de entrada de lista (estado, progreso, nota). |
| `Downloads/` | Cola, completados, almacenamiento. |
| `Extension/` | Gestor de extensiones y tester de fuentes. |
| `Social/`, `Entity/`, `Review/` | Perfiles, actividad, grupos; personajes/staff/estudios; reseñas. |
| `Notifications/`, `Calendar/`, `Charts/`, `News/`, `MangaBaka/` | Notificaciones, calendario, rankings AniTrendz, noticias, stacks. |
| `Settings/` | Ajustes data-driven (incluye `Sync/`: sincronización de listas entre servicios). |
| `Together/`, `Share/`, `Update/`, `Webview/`, `Error/` | Ver juntos, tarjetas para compartir, hoja de actualización, navegador interno, pantalla de crash. |

### 3.5 Resto
- **`Widgets/`** — componentes compartidos: `SectionCard`, `PosterCard`, `MediaSection`,
  `ShelfFrame`, `BaseScreen`, `ThemedContainer`/glass, `AppControls`, `AniHtml`, gráficos.
- **`Utils/`** — extensiones (`Dimens`, responsive), animaciones, navegación D-pad/TV, snackbars,
  deep links, LRU.
- **`Model/`** — tipos de app (`CardStyle`, `MediaType`, `SearchResults`, `Setting`).
- **`l10n/`** — localización generada (no editar a mano; editar los `.arb`).

---

## 4. Flujo de datos (en una frase cada uno)

1. **Ver un título**: `openDetail` → `DetailScreenView` del servicio actual → `Media` → la UI pinta.
2. **Ver un episodio**: pestaña Watch → `SourceMatcher` busca el título en una extensión →
   lista de episodios → `StreamSheet` → `PlayerScreen` → al 80 % `Mutations.setProgress`.
3. **Editar la lista**: `ListEditorSheet` → `Mutations.editList` (o `PendingWrites` si offline) →
   `RefreshController` avisa a los feeds → (opcional) `SyncController.pushEdit` a otros servicios.
4. **Ids cruzados**: `IdMapper` (lista Fribb + lo aprendido) traduce AniList ↔ MAL ↔ Kitsu ↔ Simkl…
   para sync, comentarios (keyed por id de AniList), metadatos de episodios y AniSkip.

---

## 5. Comandos esenciales

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs   # tras tocar modelos JSON/Isar
flutter gen-l10n                                           # tras tocar .arb
flutter analyze                                            # gate de CI
flutter run -d linux
flutter build apk --release --split-per-abi
```

Requiere `assets/.env` (clave `hash`). Flutter objetivo: `>=3.44.0` (el CI pinea versiones más antiguas).

---

## 6. Reglas de oro (para no romper nada)

- Nada de `setState`, `ValueNotifier` ni `package:get` fuera de `Core/State/GetBridge.dart`.
- Pantallas extienden `BaseScreen` e implementan `buildContent`; lógica en `*State.dart`.
- Strings visibles → `app_en.arb` + `getString.key`. Espaciados → `Dimens.*`.
- Usa los componentes compartidos (`SectionCard`, `PosterCard`, `AppSegmented`…) — ver `CONSISTENCY.md`.
- Imports relativos dentro de `lib/`; carpetas y archivos en PascalCase.
- Estado por título → un campo nuevo en `MediaRecord`, nunca una clave de preferencia nueva.
- Servicios: datos, nunca widgets. Nuevos controladores → `DI.init()`.
- Fork independiente: los commits/PRs van a `KevshuppD/DartotsuPlus`, nunca al upstream.
