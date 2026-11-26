# AGENTS.md — The Familia fork (Ashtray3250)

Инструкции для агентских сессий, работающих с этим форком.

## Исправления в форке (относительно upstream c56c051)

1. **Музыка.** `runtime/dsound8.c` + `third_party/pcrecomp/win32hle/dsound.c`
   реализуют `IDirectSoundNotify` (`0xB0210783`), которого не было в порту.
   Без него `IShow::Open()` в LS3DF (`0x10073ba0`) возвращала ошибку — музыки
   не было ни в меню, ни в катсценах, ни в городе.
2. **Радар и ползунки меню.** `runtime/d3d8_resource.c`: для `SetTexture(NULL)`
   привязывается 1×1 белая текстура. GLES давал чёрный для пустого текстурного
   объекта, поэтому vertex-coloured геометрия становилась чёрной.
3. Мелкие forward-declaration в `d3d8_device.c` / `d3d8_resource.c`.

**Больше ничего в runtime менять не надо.** Все предыдущие «костыли»
(пропуск `DllMain`, `FAMILIA_UNRESOLVED_CONTINUE`, seed'ы для `rw_data`) были
убраны: они требовались только из-за битых входных DLL и только ухудшали дело.

## КРИТИЧЕСКИ ВАЖНО: входные DLL

`rw_data.dll` и `IJoy.dll` надо брать **из установленной игры / из `data/`**
или извлекать из CD **правильно**. CD — MODE1/2352: каждый сектор занимает
2352 байта, полезные данные — 2048 байт начиная со смещения 16. Обязательно
читать **каждый сектор отдельно**. Ошибка ( readable contiguous `count*2048`)
даёт DLL того же размера, но с другим содержимым: трансляция тогда produces
неверный код, и игра падает на старте.

Проверочные хеши файлов, на которых работает устройство:

- `rw_data.dll` `67f72c30cc7fe941f5526b8602a0bd2785baa0f22617bbcfd4a71bacd79b8d49`
- `IJoy.dll`    `4e7441a927ccc7e7609c11ff6f829bf236c98360e392201611b348300a31fc2d`
- `Game.exe`    `f49809e22e0a927af670c19cc81ccd9bcab443abfc0e7737eceb8a7996dff11c`
- `LS3DF.dll`   `e211c85009440d6c032f2074d79184f074a9a046574648745c2594b4cef178a6`

## Правильная сборка

Только Linux + Docker, как в `docs/BUILDING.md`:

```sh
docker build -t familia-cross:bookworm -f docker/Dockerfile.cross docker
tools/build_ffmpeg.sh
PCRECOMP=/path/to/pcrecomp python3 -P tools/translate.py /path/to/game-files
docker run --rm -v /absolute/path/to/TheFamilia:/src -w /src \
    familia-cross:bookworm make ARCH=aarch64 -j1 build/aarch64/familia launcher import
tools/package.sh build/aarch64/familia
```

- Debian 12 (glibc 2.36), GCC 12, SDL2 arm64 — это муOS/H700.
- `-j1`: при `-j8` `cc1` убивается по памяти.
- Docker-монтировать **абсолютным** путём; `$PWD` через оболочку может
  смонтировать не ту папку.
- pcrecomp — версия времени релиза (проверено: `8091c57`).

## Что НЕ делать

- Не Clang/Zig: собранные ими бинарники падали на старте.
- Не применять `-DRECOMP_LOCAL_REGS` / `-DRECOMP_NONVOLATILE_MEM` к runtime —
  только к сгенерированному C (как в Makefile).
- Не добавлять seed'ы «на всякий случай»: с правильными DLL все функции
  находятся сами, а seed создаёт функцию в середине другого кода.
- Не выдавать бинарник без проверки на RG40XXH; всегда хранить `.orig` для отката.
