# План Rust-оптимизаций в Firefox/Gecko

Репозиторий: `x7717/gecko-dev` (форк mozilla/gecko-dev). Цель проекта — находить и
измеримо улучшать горячие пути браузера там, где код уже на Rust, и фиксировать
результаты числами (до/после). Документ честный: описывает и то, что реально на Rust,
и то, что переписать на Rust в обозримой перспективе не получится.

## Что уже написано на Rust в Gecko

- **Stylo** (`servo/components/style`, `selectors`, `style_traits`, `servo_arc`,
  `malloc_size_of`, `to_shmem`) — весь CSS-парсинг, matching селекторов и вычисление
  стиля. Клей с C++ — `layout/style/`. Один из самых горячих Rust-компонентов: любая
  загрузка страницы и динамические классы проходят через него.
- **WebRender** (`gfx/wr`: `webrender`, `webrender_api`, `webrender_build`,
  `wr_glyph_rasterizer`, `wr_malloc_size_of`, `wrench`) — GPU-рендерер UI и контента.
  Рендер-потоки (RenderBackend, SceneBuilder) — чистый Rust. `swgl` и `glsl-to-cxx` —
  софтверный GL и генератор C++ из GLSL (собираются из того же Rust-воркспейса,
  но сами не Rust).
- **HTTP/3 + QUIC — neqo**: Rust-стек (`third_party/rust/neqo-*`, клей
  `netwerk/protocol/http/neqo_glue.cpp`). Остальной Necko (HTTP/1.x, HTTP/2, сокеты) — C++.
- **encoding_rs** (`intl/encoding_glue`) — конвертация кодировок, уже активно
  SIMD-оптимизирован. Хороший образец методологии, а не цель.
- **Локализация — fluent** (`fluent-rs`, `fluent-fallback`) — движок L10n.
- **Медиа**: MP4-демукс — `mp4parse` (`media/mp4parse-rust`); аудио-бэкенды cubeb на
  Rust: `cubeb-pulse` (Linux), `cubeb-coreaudio` (macOS); `audioipc` — вынос аудио в
  отдельный процесс поверх Rust-крейта `ipc-channel` (это и есть «Rust-часть IPC»).
- **WebGPU** — на базе `wgpu` (`dom/webgpu`); на Windows включён по умолчанию
  (с Firefox 141), на остальных платформах — за флагом.
- **Прочее**: WebAuthn (`authenticator`), key-value хранилище `rkv`
  (`toolkit/components/kvstore`), XPCOM-байндинги (`xpcom/rust`: `xpcom`, `nsstring`,
  `nserror`), shared arc/атомы (`servo_arc`, `to_shmem`).

Честно: **большинство ядерных подсистем — C++**: layout (frame tree, блоки/строки),
SpiderMonkey (JS), большая часть Necko, GPU-процесс (контейнер вокруг WR),
виджет/GTK. Переписать их «большим взрывом» нереально; работа возможна только через
точечные оптимизации существующего Rust-кода и локальные интерфейсы.

## Реалистичный цикл оптимизации

1. **Выбрать горячий путь по профилю реального браузинга**, а не по интуиции.
   Кандидаты: селектор-матчинг и инкрементальный рестайл Stylo; frame building и
   батчинг в WebRender; растеризация глифов (`wr_glyph_rasterizer`); аллокации в
   горячем цикле neqo (handshake, обработка пакетов).
2. **Профилирование**:
   - встроенный Gecko Profiler (`profiler.firefox.com`) — для всего браузера;
   - `perf record -g` и `samply` (`cargo install samply; samply record ./mach run`)
     на Linux — нативные профили с символами Rust;
   - фоновые потоки WR смотреть отдельно: они вне главного треда.
3. **Микробенчмарки на criterion**. В самостоятельном воркспейсе `gfx/wr` бенчей
   сейчас нет — добавляем локально (в upstream-репозитории servo они есть для
   style). Правило: бенч до оптимизации и после, числа — в worklog проекта.
4. **Оптимизация + измерение** до/после на одном железе, минимум 3 прогона,
   фиксировать медиану и разброс.
5. **Апстрим**: баги в Bugzilla (Core :: CSS Parsing and Computation / Graphics:
   WebRender), патчи по Firefox Contributor Guide (firefox-source-docs.mozilla.org).
   Без апстрима выигрыш теряется при каждом ребейзе с mozilla-central.

## Как вписывается webrender-rust-ci.yml

- Воркспейс `gfx/wr` собирается отдельно от всего дерева Gecko — полный билд Firefox
  в GitHub Actions публичного раннера не помещается ни по времени, ни по диску.
- Воркфлоу делает `cargo check --workspace` и `cargo test -p webrender --lib`
  (ubuntu-latest, stable Rust, кеш Swatinem/rust-cache, таймаут джобы 40 минут);
  запуск вручную (`workflow_dispatch`) и по понедельникам 05:00 UTC.
- Роль: базовый гейт перед любыми экспериментами с `gfx/wr` в этом форке. Если
  `gfx/wr/Cargo.toml` исчез или переехал — джоба падает с диагностикой (`ls gfx | head`),
  а не «молча зелёная».
- Дальше можно добавить шаг с criterion-бенчами и публикацией чисел в артефакты,
  чтобы отслеживать регрессии по расписанию.

## Первые конкретные кандидаты

1. `wr_glyph_rasterizer` — повторная растеризация глифов, порядок выгрузки кеша.
2. Stylo: инкрементальный рестайл, bloom-фильтр предков в матчинге селекторов.
3. neqo: аллокации буферов пакетов, копирование на пути шифрования.
4. encoding_rs — не трогать без необходимости (уже SIMD), использовать как эталон
   подхода: бенч → SIMD/таблицы → числа → апстрим.

## Ограничения и риски

- Сборка целого Firefox в Actions невозможна (taskcluster, сотни минут, десятки ГБ) —
  CI держим только на самостоятельных Rust-воркспейсах.
- Vendored-крейты (`third_party/rust`) перезаписываются `mach vendor rust` — патчи
  туда проводить через Bugzilla/апстрим, а не правками в форке.
- Профилирование требует локального билда (`./mach build`) — GitHub Actions годится
  только для компиляции/тестов, не для реальных профилей браузера.
