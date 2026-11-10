# avatar_bot

A bot can change telegram group avatar, support image,sticker,gif, video and link.   

**Notice, everyone in group can use this bot to change group avatar.**

if you want to use anime face detcet function, you need to download a lbpcascade_animeface.xml and place it in workdir.

----
## Depends

- telethon
- opencv
- asyncio
- yaml
- filemagic
- numpy
- request
- BeautifulSoup
-He revisado el archivo que compartiste: https://github.com/MarvelousBlack/avatar2/blob/master/avatar_bot.py

El problema principal no es solo “un bug puntual”, sino varios problemas de configuración y robustez que hacen que el bot falle o no arranque bien.

Lo más importante:
- `config.yaml` tiene valores vacíos:
  - `bot_token: ''`
  - `bot_name: ''`
- Eso hace que `TelegramClient` no pueda autenticarse correctamente.
- También `artworks_root = ""` deja rota la funcionalidad de Pixiv.
- El código usa rutas fijas como `/tmp/...`, que no son compatibles con Windows.
- Hay varios `if`/`elif` que deberían ser exclusivos, pero en `get_img()` no lo son.
- `get_telegram_img()` no verifica si `replymsg` existe y puede producir `NoneType` al acceder a `replymsg.file`.

Solución real:

1) Configura `config.yaml`
Cópialo así:

```yaml
app_id: 12345678
api_hash: "tu_api_hash_aqui"
bot_token: "tu_token_de_botfather"
bot_name: "tu_bot_username_sin_@"
chat_settings:
  - id: -1001234567890
    time_limit: 100
```

Importante:
- `bot_name` debe ser el nombre del bot, por ejemplo: `miavatar_bot`
- no uses `@`, no uses espacios
- el `id` del grupo debe ser el real del chat

2) Corrige la inicialización del cliente
Tu código actual:

```python
client = TelegramClient(bot_config['bot_name'],
                        bot_config['app_id'],
                        bot_config['api_hash'],
                        )
client.start(bot_token=bot_config['bot_token'])
```

Debe estar protegido:

```python
if not bot_config.get("bot_token") or not bot_config.get("bot_name"):
    raise RuntimeError("Falta bot_token o bot_name en config.yaml")

client = TelegramClient(
    bot_config['bot_name'],
    bot_config['app_id'],
    bot_config['api_hash'],
)

client.start(bot_token=bot_config['bot_token'])
```

3) Evita rutas `/tmp` para compatibilidad multiplataforma
Cambia esto:

```python
file_name = '/tmp/avatar{}.jpg'.format(event.chat_id)
```

por algo más seguro:

```python
import tempfile
import os

file_name = os.path.join(tempfile.gettempdir(), f"avatar_{event.chat_id}.jpg")
```

Haz lo mismo para video e imágenes.

4) Corrige `get_img()`
Ahora mismo tienes:

```python
async def get_img(event,args,auto_detect):
    if auto_detect == 'pixiv':
        img = await get_pixiv_image(args['pixiv_id'],args['pic_num'])
    if auto_detect == 'link':
        img = await get_link_image(args['link'],args['pic_num'])
    if auto_detect == None:
        ...
    return img
```

Debe ser:

```python
async def get_img(event, args, auto_detect):
    if auto_detect == 'pixiv':
        return await get_pixiv_image(args['pixiv_id'], args['pic_num'])
    if auto_detect == 'link':
        return await get_link_image(args['link'], args['pic_num'])
    if auto_detect is None:
        if args['link'] is None and args['has_replymsg']:
            return await get_telegram_img(event)
        if args['link'] is not None:
            return await get_link_image(args['link'], args['pic_num'])
        return 1
    return 1
```

5) Protección en `get_telegram_img()`
Esto evita errores de `NoneType`:

```python
async def get_telegram_img(event):
    replymsg = await event.message.get_reply_message()
    if replymsg is None or replymsg.file is None:
        await event.reply("No hay una imagen o video adjunto para procesar.")
        return -1

    try:
        if replymsg.file.size > 15 * 1024**2:
            await event.reply("¡Archivo demasiado grande!")
            return 0

        if replymsg.file.mime_type == "image/gif" or "video" in replymsg.file.mime_type:
            file_name = os.path.join(tempfile.gettempdir(), f"video_{event.chat_id}.tmp")
            await client.download_media(message=replymsg, file=file_name)
            return video2img(file_name)

        if 'image' in replymsg.file.mime_type:
            file_name = os.path.join(tempfile.gettempdir(), f"image_{event.chat_id}.tmp")
            buf_bytes = await client.download_media(message=replymsg, file=bytes)
            img_buf = np.asarray(bytearray(buf_bytes), dtype="uint8")
            return cv2.imdecode(img_buf, cv2.IMREAD_UNCHANGED)

        await event.reply(f"No se soporta este tipo: {replymsg.file.mime_type}")
        return -1

    except Exception as e:
        logger.error(e)
        await event.reply("Error al procesar la imagen.")
        return -1
```

6) `artworks_root` para Pixiv
Si usas Pixiv, define una URL base válida o deja la función deshabilitada:

```python
artworks_root = "https://example.com/pixiv/"
```

o mejor:
- no uses Pixiv si aún no tienes un host funcional
- o valida antes de intentar una descarga

7) Mejoras adicionales
- usa `logging.exception` para ver el traceback real
- evita `except Exception as e: logger.error(e)` sin contexto
- usa `os.path.exists` antes de borrar archivos
- en `update_channel_avatar`, haz `try/finally` para eliminar el archivo temporal

Si quieres, te puedo dejarte una versión corregida completa del archivo `avatar_bot.py` ya arreglada y lista para ejecutar en Linux, Windows o macOS.


