# AGENTS.md — The Familia fork (Ashtray3250)

Инструкции для агентских сессий, работающих с этим форком. **Соблюдать строго:**
попытки собрать порт «на скорую руку» на Windows уже приводили к бинарникам,
которые падали сразу после загрузки модулей (SIGSEGV в `recomp_ls3df_*`),
хотя исходные исправления были правильными.

## Что уже исправлено в этом форке (по сравнению с upstream c56c051)

1. **Музыка.** `runtime/dsound8.c` + `third_party/pcrecomp/win32hle/dsound.c`
   реализуют `IDirectSoundNotify` (`IID_IDirectSoundNotify = 0xB0210783`):
   `QueryInterface`, `SetNotificationPositions` и взвод событий из SDL-микшера
   при прохождении курсора по DirectSound-буферу. Без этого интерфейса
   `IShow::Open()` в LS3DF (`0x10073ba0`) возвращала ошибку — музыки не было
   ни в меню, ни в катсценах, ни в городе.
2. **Радар и ползунки меню.** `runtime/d3d8_resource.c`: для `SetTexture(NULL)`
   привязывается 1×1 белая текстура. GLES при пустом текстурном объекте
   возвращал чёрный, и vertex-coloured геометрия (отметки радара, красная
   полоса `GM_Slider::Draw` `0xffbe002c`, серые заливки) становилась чёрной.
3. Мелкие forward-declaration в `d3d8_device.c` / `d3d8_resource.c` — чтобы
   код компилировался и более строгими компиляторами.

Эти правки **не проверены на устройстве** в виде готового бинарника.
Проверку должен сделать человек на RG40XXH с muOS.

## Единственный известный правильный процесс сборки

Только **Linux x86-64 + Docker**, как в upstream `docs/BUILDING.md`.
`docker/Dockerfile.cross` — Debian bookworm, glibc **2.36**, GCC **12**
(`gcc-aarch64-linux-gnu`), SDL2/SDL2_ttf arm64 из пакетов Debian 12.
Это в точности соответствует целевой прошивке (muOS на H700).

```sh
# 1. Образ кросс-компилятора (обязательно Debian 12 / GCC 12)
docker build -t familia-cross:bookworm -f docker/Dockerfile.cross docker

# 2. Минимальный FFmpeg 6.1.2 (docker внутри скрипта)
tools/build_ffmpeg.sh

# 3. Трансляция x86 -> C. ВАЖНО: PCRecomp надо брать версией того времени,
#    а не просто main на сегодня. Для релиза v0.2.8 (коммит c56c051,
#    2026-11-11) подходит pcrecomp 8091c57 (11 ноября 2026).
PCRECOMP=/path/to/pcrecomp python3 -P tools/translate.py /path/to/game-files

# 4. Сборка
docker run --rm -u $(id -u):$(id -g) -v $PWD:/src -w /src familia-cross:bookworm \
    make ARCH=aarch64 -j$(nproc) build/aarch64/familia launcher import

# 5. Упаковка
tools/package.sh build/aarch64/familia
```

Входные файлы для шага 3: `Game.exe` 1.2 без SafeDisc (SHA-256
`f49809e2...`), `LS3DF.dll` из патча 1.2 (`e211c850...`), `rw_data.dll`,
`IJoy.dll`. CD-образы и патч распаковываются в `data/` импортером порта.

## CHEATSHEET: как НЕ надо (запрещено повторять)

- **Не собирать Clang/Zig вместо GCC 12.** Бинарники Zig/Clang 22 падали
  на старте с `fault address = guest 0x93B80877` и одинаковым guest-ESP
  `0x105FFFB0` — это была ошибка окружения, а не логики исправлений.
- **Не применять `-DRECOMP_LOCAL_REGS` / `-DRECOMP_NONVOLATILE_MEM` ко всем
  файлам.** В `Makefile` они есть только в `GENFLAGS` (сгенерированный C).
  Если применить их к `runtime/`, guest-память компилируется с другими
  определениями, чем в сгенерированном коде, — результат тот же вылет.
- **Не использовать Windows-хостевой `mingw-w64-i686` GCC от Arm:** 32-битный
  `cc1.exe` падает с `out of memory` на `build/gen/game/f_010.c` и `f_013.c`.
  Гибрид GCC+Clang тоже не годится.
- **Не пользоваться/latest `pcrecomp` main без нужды.** Новые версии lifter'а
  генерируют `RECOMP_TRAP` / `fp_bcd_st`, которых нет во вендорированном
  `recomp32/recomp_types.h`; это маскируется костылями и меняет код.
- Известная мелочь: транслятор может не находить `rw_data!110025C0`
  (`native_rwdata.c` ссылается на `recomp_rw_data_110025C0_orig`). Рабочий
  обход — добавить `config/seeds/rw_data.json` с `"0x110025C0"`. Прежде чем
  добавлять, проверьте, не находится ли функция естественным путём в вашей
  версии транслятора.

## Обязательная проверка перед передачей бинарника

1. Собрать **только** по инструкции выше (Docker + GCC 12).
2. Не перезаписывать рабочий бинарник пользователя: держать
   `familia.aarch64.orig` для отката.
3. Отдать бинарник на тест владельцу устройства RG40XXH/muOS и **дождаться
   `log.txt`**. Признак успеха: в логе есть `[dsound] ... Hz`,
   `[d3d8] N shader variants prebuilt`, игра доходит до меню; музыка идёт,
   отметки радара и ползунков цветные.
4. Если есть вылет — сначала `log.txt`, потом изменения. Не пересобирать
   «вслепую» и не выкладывать непроверенные сборки.
