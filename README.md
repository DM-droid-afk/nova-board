# Nova Board — Pizarra digital nativa para Windows 11

Aplicación de escritorio nativa (Tauri 2 + Rust + WebView2 + React + Fabric.js)
con diseño **Liquid Glass** estilo iOS, motor vectorial, exportación nativa y
auto-recuperación offline.

## Características principales

- **Ventana frameless** con barra de título personalizada (arrastre nativo, doble
  clic para maximar, minimizar, cerrar).
- **Animación de apertura**: splash con logo animado (pop + glow) que se
  desvanece automáticamente al iniciar.
- **Ajustes (engranaje en el encabezado del Dashboard)**: cambio de idioma ES/EN
  y buscador de actualizaciones.
- **Actualizaciones por internet** vía `tauri-plugin-updater`: comprueba,
  descarga, instala y reinicia desde el diálogo de Ajustes.
- **Snap Layouts de Windows 11**: implementado con subclase Win32
  (`WM_NCHITTEST` → `HTMAXBUTTON`) sobre la barra HTML para que el panel de
  distribuciones aparezca al arrastrar la ventana al borde.
- **Liquid Glass**: refracción real (recreación en JS puro de la técnica de los
  repos de la comunidad: SDF del rectángulo redondeado → `feDisplacementMap`
  en `backdrop-filter`, ver `src/lib/liquidGlass.ts`), más material translúcido
  con `blur + saturate`, borde refractivo de 1px, esquinas squircle y sombras
  ambientales profundas. Paleta de sistema Apple (`#0A84FF`, `#64D2FF`,
  `#5E5CE6`, `#30D158`, `#FF9F0A`, `#FF453A`, `#FFD60A`...).
- **Motor vectorial**: Fabric.js 5 con pincel por presión (Windows Ink), texturas
  (suave, rotulador, lápiz, acuarela), opacidad y grosor ajustables.
- **Herramientas**: lápiz, pincel, borrador (trazo/capa), líneas, flechas,
  rectángulos, elipses, texto con fuentes instaladas y estilos (negrita/italica).
- **Navegación**: pan con `Espacio` + arrastre, zoom con `Ctrl` + rueda, pinch
  nativo de trackpad Windows (WheelEvent), Ctrl+Z / Ctrl+Y / Ctrl+S / Ctrl+C / Ctrl+V.
- **Capas**: crear, reordenar, mostrar/ocultar, bloquear.
- **Persistencia**: `%APPDATA%\Nova Board\boards\*.board` (gzip + JSON con
  vectores) y `%APPDATA%\Nova Board\recovery\*.json` (autosave cada 30 s y
  ante cierre).
- **Exportación**: SVG vectorial puro, PDF vectorial (jsPDF + svg2pdf), PNG/JPG de
  alta densidad (hasta 8×) con viewport de exportación y guardado vía diálogo nativo.
- **Dashboard**: crear, renombrar, eliminar, reordenar por drag & drop, tamaño
  finito/infinito, fondo, cuadrícula o puntos.

## Arquitectura

```
whiteboard/
├── src/                        # Frontend React + TypeScript (WebView2)
│   ├── components/
│   │   ├── Canvas/             # Motor Fabric.js, toolbar, propiedades, capas, export
│   │   ├── Dashboard/          # Gestión de pizarras
│   │   └── Glass/               # GlassCard, GlassButton, Modal
│   ├── lib/                     # Export (SVG/PDF/PNG/JPG), wrappers Tauri
│   ├── store/                   # Zustand (dashboard)
│   ├── hooks/                   # useBoardLoader
│   └── types/board.ts          # Contrato frontend/backend
├── src-tauri/                  # Backend Rust nativo
│   ├── src/
│   │   ├── main.rs             # App + registro de comandos + Snap Layouts
│   │   ├── board.rs            # Modelo Board (finito/infinito, capas)
│   │   ├── storage.rs          # Persistencia gzip, recuperación, export dialog
│   │   └── win_snap.rs        # WM_NCHITTEST / HTMAXBUTTON (Snap Layouts)
│   └── tauri.conf.json
├── package.json                 # Tauri CLI, React, Fabric, jsPDF, Zustand, Tailwind
└── vite.config.ts
```

## Requisitos

- Windows 11 (64 bits)
- [Node.js 18+](https://nodejs.org/)
- [Rust](https://rustup.rs/) (toolchain `stable`)
- WebView2 (incluido en Windows 11)

## Desarrollo

```bash
npm install
npm run tauri dev        # arranca la ventana nativa en modo debug
```

Verificación y build:

```bash
npm run typecheck        # TypeScript
npm run build            # Vite (frontend → dist/)
cargo check --manifest-path src-tauri/Cargo.toml
npm run tauri build -- --debug --no-bundle   # binario nativo debug
npm run tauri build                           # instalador (.msi / .exe NSIS)
```

Salida del instalador: `src-tauri/target/release/`.

## Almacenamiento local

- Pizarras: `%APPDATA%\Nova Board\boards\<id>.board` (gzip + JSON)
- Recuperación: `%APPDATA%\Nova Board\recovery\<id>.json`

Nunca se sube nada a la nube: 100 % offline.

## Exportaciones

| Formato | Contenido |
|---|---|
| SVG | Vectorial puro, sin pérdida. |
| PDF | Vectorial con tamaño exacto del lienzo. |
| PNG/JPG | Raster a escala 2× / 4× / 8× con viewport de exportación. |

## Actualizaciones por internet

El botón de **Ajustes → Actualizaciones** ejecuta
`check()` del plugin `tauri-plugin-updater`; si hay una versión más reciente
firmable (`pubkey` ya configurada en `tauri.conf.json`), se descarga, se instala
en silencio y la app se reinicia con `tauri-plugin-process`.

Para publicar una actualización:

1. Incrementa `version` en `src-tauri/tauri.conf.json`, `package.json` y
   `src-tauri/Cargo.toml`.
2. Construye firmado (la clave privada **nunca** se sube al repo):

   ```powershell
   $env:TAURI_SIGNING_PRIVATE_KEY = "C:\ruta\a\tauri-updater.key"
   # Solo si la clave tuviera contraseña:
   # $env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD = "..."
   npm run tauri build
   ```

3. Sube los artefactos del instalador NSIS a tu endpoint HTTPS
   (`bundle/nsis/*-setup.exe` y `*-setup.exe.sig`) y publica un
   `latest.json` en la URL configurada en `tauri.conf.json` →
   `plugins.updater.endpoints`, con el formato:

   ```json
   {
     "version": "1.1.0",
     "notes": "Mejoras y correcciones",
     "pub_date": "2026-11-11T12:00:00Z",
     "platforms": {
       "windows-x86_64": {
         "signature": "<contenido del archivo .exe.sig>",
         "url": "https://<tu-host>/Nova%20Board_1.1.0_x64-setup.exe"
       }
     }
   }
   ```

4. La app comparará versiones con `semver` y verificará la firma minisign con
   la `pubkey` incluida; si no coincide, rechaza la actualización.

⚠️ La `endpoint` configurada en `tauri.conf.json`
(`https://github.com/DM-droid-afk/nova-board/releases/latest/download/latest.json`)
sirve el `latest.json` desde GitHub Releases del repo público `nova-board`.
La clave pública ya está en el config; **guarda `tauri-updater.key` en un lugar
seguro y fuera del control de versiones** (está cubierta por `.gitignore`).

## Limitaciones conocidas

- Exportación SVG/PDF usa el **tamaño finito** de la pizarra; en modo infinito
  se exporta el **viewport actual**.
- Snap Layouts: la intercepción se implementa sobre el área del botón
  maximizar del título HTML; no se intercepta el arrastre por toda la barra
  (solo el botón), que es lo requerido por el sistema para activar el panel.
- La detección de fuentes usa la Web API `queryLocalFonts` cuando está disponible;
  si no, se usan fuentes del sistema por defecto.

## Licencia / uso

Uso personal offline. Sin telemetría, sin cuentas, sin nube.
