# Whisper на Windows 11: расшифровка аудио и видео в текст

Whisper — бесплатная программа, которая превращает речь из аудио или видео в текст прямо на вашем компьютере, без интернета и без загрузки файлов куда-либо. Настройка — один раз минут на 15–30 (больше всего времени уходит на скачивание), дальше одна команда на каждый файл.

Инструкция для Windows 11 (Pro, Home — неважно) на обычном компьютере с процессором Intel или AMD. Права администратора не нужны.

## Как работать с Терминалом

- Открыть: правой кнопкой по «Пуску» → **«Терминал»**. Откроется окно PowerShell — в нём и работаем.
- Команды из этой инструкции копируйте целиком (каждый серый блок), вставляйте в Терминал **правой кнопкой мыши** или Ctrl + V, затем Enter.
- Если вставляете блок из нескольких строк, Терминал может переспросить «Вставить несколько строк?» — нажмите «Всё равно вставить».
- Пока команда работает, ничего не нажимайте. Готово, когда снова появилась строка вида `PS C:\Users\Имя>`.
- **Путь к файлу не нужно печатать вручную:** перетащите файл из Проводника мышкой прямо в окно Терминала — путь вставится сам. Или: правой кнопкой по файлу → «Копировать как путь» → вставить в Терминал.

Все файлы Whisper будут лежать в одной папке `C:\whisper`. Удалить всё целиком — просто удалить эту папку.

## Шаг 1. Разрешить Терминалу запускать настройки (один раз)

По умолчанию Windows не даёт PowerShell запускать даже собственный файл настроек. Эта команда разрешает это только для вашего пользователя:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned -Force
```

## Шаг 2. Установить конвертер звука ffmpeg (один раз)

ffmpeg достаёт звук из видео и переводит любой формат (mp3, m4a, ogg, mp4, mov, голосовые из Telegram и WhatsApp) в нужный Whisper. Ставится встроенным в Windows 11 установщиком winget:

```powershell
winget install --id Gyan.FFmpeg -e --accept-source-agreements --accept-package-agreements
```

После установки **закройте Терминал и откройте заново**, затем проверьте — должна появиться версия, а не ошибка:

```powershell
ffmpeg -version
```

## Шаг 3. Скачать Whisper (один раз)

Команда создаёт папку `C:\whisper`, находит свежую версию Whisper для Windows на GitHub, скачивает и распаковывает её. Скопируйте блок целиком:

```powershell
$ProgressPreference = 'SilentlyContinue'
New-Item -ItemType Directory -Force C:\whisper\bin, C:\whisper\models, C:\whisper\tmp | Out-Null
$url = (Invoke-RestMethod "https://api.github.com/repos/ggml-org/whisper.cpp/releases?per_page=10").assets | Where-Object name -eq "whisper-bin-x64.zip" | Select-Object -First 1 -ExpandProperty browser_download_url
curl.exe -L -o C:\whisper\whisper.zip $url
Expand-Archive C:\whisper\whisper.zip C:\whisper\bin -Force
Get-ChildItem C:\whisper\bin -Recurse -Filter whisper-cli.exe | Select-Object FullName
```

В конце должна появиться строка с путём к `whisper-cli.exe`. Если её нет — повторите блок.

## Шаг 4. Скачать модель (один раз)

Модель — это «мозг» Whisper, отдельный большой файл. Рекомендованная — **large-v3-turbo** (1,6 ГБ, хорошее качество для русского):

```powershell
curl.exe -L -o C:\whisper\models\ggml-large-v3-turbo.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-turbo.bin
```

### Чем отличаются модели

| Модель | Размер | Скорость | Качество русского | Когда брать |
|---|---|---|---|---|
| tiny | 75 МБ | очень быстро | плохо | только проверить, что всё работает |
| base | 142 МБ | очень быстро | слабо, много ошибок | черновик чистой речи |
| small | 466 МБ | быстро | средне | слабый или старый компьютер |
| medium | 1,5 ГБ | средне | хорошо | если turbo ошибается на терминах |
| **large-v3-turbo** | **1,6 ГБ** | **средне** | **очень хорошо** | **основной вариант** |
| large-v3 | 3 ГБ | медленно, в 3–5 раз дольше turbo | лучшее | важные записи, плохой звук, несколько собеседников |

Модели с `.en` в названии понимают только английский — для русского их не берите.

Скачать другую модель — та же команда, поменяйте название в двух местах. Например, small:

```powershell
curl.exe -L -o C:\whisper\models\ggml-small.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-small.bin
```

Посмотреть, какие модели уже скачаны:

```powershell
Get-ChildItem C:\whisper\models
```

## Шаг 5. Создать короткую команду transcribe (один раз)

Эта команда добавляет в Терминал слово `transcribe`, которое делает всё сразу: достаёт звук, расшифровывает и сохраняет текст рядом с исходным файлом. Скопируйте блок целиком:

```powershell
if (-not (Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force | Out-Null }
@'
function transcribe {
  $model = if ($env:WHISPER_MODEL) { $env:WHISPER_MODEL } else { "C:\whisper\models\ggml-large-v3-turbo.bin" }
  $lang = if ($env:WHISPER_LANG) { $env:WHISPER_LANG } else { "ru" }
  $exe = Get-ChildItem C:\whisper\bin -Recurse -Filter whisper-cli.exe | Select-Object -First 1
  if (-not $exe) { Write-Host "ERROR: whisper-cli.exe not found - repeat step 3"; return }
  if (-not (Test-Path $model)) { Write-Host "ERROR: model not found: $model - repeat step 4"; return }
  foreach ($arg in $args) {
    foreach ($f in (Resolve-Path -Path $arg).Path) {
      Write-Host ">> $f"
      $wav = "C:\whisper\tmp\in.wav"; $out = "C:\whisper\tmp\out"
      Remove-Item "$out.txt", "$out.srt" -ErrorAction SilentlyContinue
      ffmpeg -loglevel error -y -i "$f" -ar 16000 -ac 1 -c:a pcm_s16le $wav
      & $exe.FullName -m $model -l $lang -otxt -osrt -of $out $wav *> $null
      $base = Join-Path (Split-Path $f -Parent) ([IO.Path]::GetFileNameWithoutExtension($f))
      if (Test-Path "$out.txt") {
        Move-Item "$out.txt" "$base.txt" -Force
        Move-Item "$out.srt" "$base.srt" -Force
        Write-Host "OK: $base.txt"
      } else { Write-Host "ERROR: could not transcribe $f" }
      Remove-Item $wav -ErrorAction SilentlyContinue
    }
  }
}
'@ | Add-Content $PROFILE
. $PROFILE
```

Настройка закончена. Дальше — только шаг 6.

## Шаг 6. Расшифровать файл (каждый раз)

1. Откройте Терминал.
2. Напечатайте `transcribe` и **пробел**.
3. Перетащите файл из Проводника в окно Терминала.
4. Нажмите Enter.

В итоге строка выглядит примерно так:

```powershell
transcribe "C:\Users\Имя\Downloads\встреча.mp4"
```

Пока идёт работа, видна строка `>> путь к файлу`. Когда появится `OK: …`, рядом с исходным файлом будут два файла с тем же названием:

- `встреча.txt` — сплошной текст;
- `встреча.srt` — текст с таймкодами (видно, на какой минуте что сказано; открывается Блокнотом, подходит как субтитры к видео).

Ориентир по времени: расшифровка идёт на процессоре, поэтому дольше, чем на Mac. На современном ноутбуке с моделью turbo часовая запись — примерно от 15 минут до часа. Если слишком медленно — возьмите модель small или medium.

### Готовые шаблоны

Несколько файлов за раз — перетащите их по очереди после слова `transcribe`, через пробел:

```powershell
transcribe "ФАЙЛ1" "ФАЙЛ2" "ФАЙЛ3"
```

Все mp3 из папки «Загрузки»:

```powershell
transcribe "$HOME\Downloads\*.mp3"
```

Все файлы из отдельной папки — например, заведите на рабочем столе папку «Расшифровать» и складывайте туда записи:

```powershell
New-Item -ItemType Directory -Force "$HOME\Desktop\Расшифровать" | Out-Null
```

```powershell
transcribe "$HOME\Desktop\Расшифровать\*"
```

Если Рабочий стол синхронизируется с OneDrive, папка окажется в `$HOME\OneDrive\Desktop\Расшифровать` — тогда используйте этот путь или просто перетащите файлы мышкой.

Запись на английском (или `auto` — Whisper определит язык сам). Настройка действует до закрытия окна Терминала:

```powershell
$env:WHISPER_LANG = "en"
```

Другая модель — например, самая точная large-v3 (сначала скачайте её, шаг 4). Тоже до закрытия окна:

```powershell
$env:WHISPER_MODEL = "C:\whisper\models\ggml-large-v3.bin"
```

Открыть получившийся текст в Блокноте:

```powershell
notepad "$HOME\Downloads\встреча.txt"
```

## Если что-то пошло не так

| Сообщение | Что делать |
|---|---|
| «выполнение сценариев отключено в этой системе» | Не выполнен шаг 1. Выполните его и откройте Терминал заново. |
| `ffmpeg` не распознано как имя командлета | Закройте Терминал и откройте заново. Не помогло — повторите шаг 2. |
| `transcribe` не распознано как имя командлета | Закройте Терминал и откройте заново. Не помогло — повторите шаг 5. |
| `ERROR: whisper-cli.exe not found` | Повторите шаг 3. Если ссылка не скачивается — включите или выключите VPN и повторите. |
| `ERROR: model not found` | Команда ищет модель large-v3-turbo. Скачайте её (шаг 4) или укажите ту, что уже есть: `$env:WHISPER_MODEL = "C:\whisper\models\ИМЯ.bin"`. |
| `ERROR: could not transcribe` | Проверьте, что файл открывается в обычном плеере и в нём есть звук. Если модель скачивалась с обрывом — удалите её из `C:\whisper\models` и скачайте заново. |
| Антивирус ругается на `whisper-cli.exe` | Это официальная сборка проекта whisper.cpp с GitHub. Добавьте папку `C:\whisper` в исключения. |
| Текст с ошибками | Возьмите large-v3. Сильно помогает и чистый звук без музыки и шума. |
| В тексте повторяется одна и та же фраза или «Продолжение следует…» | Так Whisper «додумывает» тишину в конце записи — просто удалите эти строки. |

Ничего никуда не отправляется: и модели, и расшифровка работают только на этом компьютере, поэтому подходит и для конфиденциальных записей.

Инструкция для Mac: [fdn11.github.io/whisper-mac-guide](https://fdn11.github.io/whisper-mac-guide/)
