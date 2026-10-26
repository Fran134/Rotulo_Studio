# Rótulo Studio

*v3.7 · antes «Traductor de videos»*

Lee el texto que aparece en pantalla (inglés), lo traduce y genera subtítulos (.srt),
una hoja de revisión (.csv) y, si quieres, el video con los subtítulos pegados.

## Uso

| Archivo | Para qué |
|---|---|
| `instalar.bat` | Una sola vez: instala lo necesario (RapidOCR con GPU AMD, ffmpeg…). |
| **`Rotulo Studio.bat`** | Abre la ventana del programa (`Traductor.bat` sigue funcionando). |
| `crear_acceso_directo.bat` | Pone un acceso directo con ícono en el escritorio. |
| `traducir.bat` | Modo consola: traduce todos los videos de la carpeta. |
| `rehacer_con_correcciones.bat` | Modo consola: rehace subtítulos y video con el CSV corregido. |
| `probar_velocidad.bat` / `probar_traduccion.bat` | Diagnóstico (también están en la ventana). |
| `instalar_ia_local.bat` | Instala Ollama y un modelo para traducir con IA en tu PC (mejor que Google). |
| `actualizar_ytdlp.bat` | Actualiza yt-dlp si una descarga deja de funcionar (los sitios cambian seguido). |
| `trabajador.bat` | **En la laptop / otra PC**: la convierte en ayudante de la PC principal (ver *Trabajo en red*). |
| `instalar_trabajador_nvidia.bat` | Opcional, en una PC con NVIDIA: LaMa y el OCR corren enteros en la tarjeta (CUDA). |

Consola con opciones: `python main.py --help`

Descargar y traducir desde consola: `python main.py --descargar URL1 URL2`
(`--lista` para una lista de reproducción completa, `--solo-descargar` para no traducir).

Los ajustes (idioma, GPU, tamaño de letra, configuración de velocidad…) se guardan en
`ajustes.json` y los usan tanto la ventana como la consola.

## Arquitectura

```
main.py                    punto de entrada: sin opciones abre la ventana, con opciones la consola
app/
├── recursos/              ícono del programa
├── modelos.py             datos del dominio: Segmento, RutasVideo, Evento
├── config.py              Ajustes (ajustes.json)
├── plataforma.py          detalles de Windows (procesos sin ventana, prioridad, abrir archivos)
├── pipeline.py            ORQUESTADOR: une las piezas, salta lo ya hecho, emite eventos, cancela
├── nucleo/                piezas independientes, sin interfaz (reciben callbacks)
│   ├── descarga.py        bajar videos de páginas web con yt-dlp
│   ├── video.py           leer cuadros con ffmpeg (decodificación por hardware)
│   ├── ocr.py             RapidOCR en varios procesos CPU/GPU (DirectML) + medición de velocidad
│   ├── limpieza.py        corregir errores típicos del OCR
│   ├── segmentos.py       agrupar cuadros en textos con tiempo de inicio/fin
│   ├── traduccion.py      servicios (Google, DeepL, MyMemory, IA local) con respaldo automático
│   ├── subtitulos.py      leer/escribir SRT, CSV y caché del OCR
│   ├── superponer.py      modo «encima del original»: zonas, tamaño de letra y archivo ASS
│   ├── borrado.py         plantillas: forma y estilo de cada texto, borrado y visibilidad cuadro a cuadro
│   ├── rotulos.py         dibuja la traducción (tamaño medido con la fuente real, sin salirse de la zona)
│   ├── quemado.py         pegar subtítulos con ffmpeg (GPU AMD/NVIDIA o CPU) con progreso, por tramos
│   └── unir.py            une los tramos de un video hechos en varios equipos y pone el audio
├── red/                   trabajo en red (opcional)
│   ├── protocolo.py       mensajes, archivos verificados (SHA-256), emparejamiento con código (HMAC)
│   ├── descubrimiento.py  encontrar trabajadores en la red local (UDP) sin escribir direcciones
│   ├── trabajador.py      el ayudante por terminal (OCR, crear tramos, relleno con IA)
│   ├── cliente.py         la PC principal hablando con un trabajador
│   ├── coordinador.py     reparte el lote, parte tramos en curso, sumatoria de equipos
│   ├── codigo.py          mantiene el programa del trabajador igual al de la PC principal
│   └── equipo.py          datos del equipo (procesador, tarjeta)
└── interfaz/
    ├── cli.py             consola: traduce eventos a texto
    ├── gui.py             ventana Tkinter: lista de videos, opciones, progreso y editor de revisión
    └── editor_visual.py   ver el cuadro del video y ajustar a mano zonas, modo y letra de cada línea
```

Reglas de la arquitectura:

- **El núcleo no conoce la interfaz.** Nada en `nucleo/` imprime ni abre ventanas; avisa con
  funciones `log` / `al_progreso` y se detiene con un evento `cancelar`.
- **Una sola lógica para consola y ventana.** Ambas crean un `Pipeline` y solo cambian cómo
  muestran los `Evento` que reciben.
- **La ventana nunca se congela:** el trabajo corre en un hilo; los eventos llegan por una cola
  que la ventana lee cada 100 ms.
- **Para añadir un traductor nuevo:** crea una clase que herede de `Servicio` en
  `nucleo/traduccion.py` y agrégala en `servicios_para()`.

## Archivos que se generan junto a cada video

- `video.es.srt`: subtítulos
- `video_revision.csv`: inglés y español lado a lado (se edita en la pestaña *Revisar y corregir*)
- `video_ocr.json`: texto leído (caché, para no repetir el OCR)
- `video_ES.mp4`: video con subtítulos

## Traducción con IA local

1. Ejecuta `instalar_ia_local.bat` (instala Ollama y descarga `qwen2.5:7b`).
2. En la ventana: **Traductor → IA local**, luego **⚙ Configurar IA local** → *Buscar modelos* → *Probar* → *Guardar*.
3. En *Estilo* puedes escribir instrucciones (variante del español, tuteo, cómo traducir ciertas palabras).
4. Para comparar con un video ya hecho: pestaña *Revisar y corregir* → **↻ Retraducir todo** (no repite el OCR
   y conserva las líneas que corregiste a mano).

La IA traduce en grupos de 12 líneas viendo las anteriores como contexto. Si el servidor no responde,
se usa Google como respaldo y el registro lo avisa. También sirve LM Studio (`http://localhost:1234/v1`)
o cualquier servidor compatible con la API de OpenAI.

## Traducción encima del texto original

En *Opciones → Traducción* elige:

- **Abajo (subtítulo clásico)**: como siempre.
- **Encima del original · borrar texto (natural)** ⭐: borra SOLO las letras (y su borde y sombra)
  reconstruyendo el fondo que hay detrás, y escribe el español en el mismo sitio, del mismo tamaño y con
  el mismo color que el original. Cómo lo hace:
  - Antes de crear el video analiza cada texto en varios cuadros: qué píxeles son letras, su color y su
    tamaño real (el alto de las letras, no el de la caja del OCR).
  - Mientras crea el video mide en **cada cuadro** cuánto se ve el original: la traducción aparece,
    desaparece y se funde **al mismo tiempo** que el texto original (aunque el OCR lea un cuadro por segundo).
  - **Fondo real:** guarda cómo se veía ese lugar un momento antes de que apareciera el texto (o justo
    después de que se fue). Si la escena está quieta, pega ese fondo real en vez de inventarlo: no queda
    ninguna mancha. Si la **cámara** se mueve (paneo o zoom lento sobre un dibujo), primero acomoda el
    fondo limpio a la nueva posición. Si lo que se mueve es el personaje (animación), reconstruye el
    fondo (clásico o con IA).
  - Si un logo o menú entra volando y luego se queda quieto, usa la posición final; si se mueve todo el
    tiempo, no lo tapa (va abajo; si es una sola palabra, como un logo, no se traduce).
  - Si dos traducciones que están a la vez se pisan, achica un poco las dos.
  - El Registro avisa cuántos textos tienen «poco contraste»: en esos se usa el tiempo del OCR tal cual;
    conviene revisarlos en el Editor visual.
- **Borrado con IA (experimental)**: casilla debajo de *Traducción*. Rellena el hueco de las letras
  con un modelo de IA que continúa pelo, tela y líneas del dibujo en vez de dejar una mancha lisa.
  Usa **MI-GAN** (~28 MB, rápido, funciona en GPU AMD) y, si no está, **LaMa** (~200 MB; en GPU AMD
  solo corre en la CPU, lento). La primera vez descarga MI-GAN a la carpeta `modelos` (o ejecuta
  `descargar_modelo_ia.bat`). Si el modelo corre en la CPU, la IA solo se usa en imágenes quietas.
  Si falla algo, el programa usa el borrado clásico.
- **Encima del original · difuminar**: borra el inglés rellenando con los colores de alrededor y escribe
  el español en el mismo sitio. Se ve natural en fondos parejos; si algo se mueve detrás, se nota una mancha.
- **Encima del original · caja de color**: tapa el inglés con un recuadro del color del fondo de esa zona.

Consejo: con *borrar texto* basta 1 cuadro por segundo (el tiempo exacto se mide al crear el video).
Los videos traducidos antes de esta versión no guardaron la posición de los textos: marca
*Volver a leer el texto (OCR)* una vez; tus traducciones y correcciones se conservan.
Las posiciones quedan en el CSV (columnas x1, y1, x2, y2), así que *Regenerar* también las usa.

## Trabajo en red (dos o más PCs)

Otra PC de la casa (por ejemplo una laptop) puede ayudar a la principal: las dos trabajan como si
fueran una sola. Es **opcional**: si no hay ningún equipo, todo funciona igual que siempre.

**Preparar la laptop (una vez):**
1. Copia la carpeta del programa a la laptop y ejecuta `instalar.bat`.
2. (Opcional, si tiene NVIDIA) `instalar_trabajador_nvidia.bat` para LaMa en la tarjeta.
3. Conecta las dos PCs al módem, mejor por **cable**.

**Cada vez:** en la laptop abre `trabajador.bat` y déjalo abierto (es solo una terminal). Muestra un
**código de 6 cifras**: la primera vez, en la PC principal pulsa **Emparejar equipo…** (abajo, en
*Equipos en red*) y escríbelo. Windows puede preguntar por el firewall en la laptop: elige
*Redes privadas → Permitir*. Desde ahí la PC principal la encuentra sola.

**Modos** (en *Equipos en red*):
- **Automático** ⭐: cada equipo hace videos completos y, cuando uno se queda sin trabajo, se queda con
  el final de lo que hace el otro (así nadie espera y el último video también se reparte).
- **Repartir videos**: cada equipo hace videos completos.
- **Todo a un video**: cada video se parte entre todos según su velocidad (para uno urgente o largo).

**Qué hace cada uno:** la PC principal tiene la lista, traduce (Google o tu IA local) y guarda los
resultados en tu carpeta. La laptop lee el texto (OCR) y crea tramos del video con su procesador y
su tarjeta. Los tramos se cortan donde no hay texto y se unen sin que se note; el audio original se
pone al final. Si la laptop se apaga o se desconecta, lo que estaba haciendo lo termina la principal.
Si actualizas solo la PC principal, la laptop recibe la versión nueva y se reinicia sola.

**Sumatoria:** mientras trabajan, el panel *Equipos en red* muestra cada equipo (qué hace, cuadros por
segundo, aporte) y una fila **TOTAL** con la velocidad de los dos juntos, cuánto más rápido vas
(×1.75, por ejemplo) y cuánto falta (y cuánto faltaría solo con esta PC). Al terminar, el Registro
muestra el reparto y el tiempo ahorrado. La terminal de la laptop también muestra el total.

**Opciones del trabajador** (se pueden poner al final en `trabajador.bat` o desde la consola):
`--nombre Laptop` · `--hilos 10` (deja libres algunos hilos) · `--sin-gpu` · `--solo-ia` ·
`--emparejar` (código nuevo) · `--al-iniciar` (arranca solo con Windows).

**Consola de la PC principal:** `--sin-red`, `--modo-red auto|videos|un_video`, `--codigo-red 123456`,
`--red-ip 192.168.1.20` (si el módem no deja encontrarla sola).

**Seguridad:** solo acepta equipos de tu red local y con el código de emparejamiento (el código no
viaja por la red: se usa para firmar cada conexión).

## Editor visual (control manual)

*Revisar y corregir* → **🖼 Editor visual**. Muestra el cuadro real del video con las zonas encima:

- **Arrastra** una zona para moverla y **sus esquinas** para cambiar el tamaño. **✎ Dibujar zona** crea una nueva.
- **Modo** de cada línea: *Automático*, *Encima del original*, *Abajo (subtítulo)* u *Ocultar*
  (para logos, nombres de canal, botones…).
- **Letra**: tamaño fijo para esa línea (0 = automático).
- **⇤ aquí / aquí ⇥**: usa el momento actual del video como inicio o fin de la línea.
- **▶ Vista previa (6 s)**: crea solo ese pedazo del video con tus cambios, sin guardar nada.
- **Guardar y crear video**: guarda y genera el video completo.

Atajos: ← → para la línea anterior/siguiente, Ctrl+S para guardar.
