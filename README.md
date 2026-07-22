# Cryobiology V — Cell Mask Annotation Pipeline

> Human-in-the-loop інструмент для побудови **ground-truth датасетів** сегментації
> клітинної мікроскопії: ML дає чорнові маски → людина у браузері їх відбирає,
> виправляє й групує → система запікає узгоджений датасет для аналізу.
>
> *Human-in-the-loop pipeline to turn rough ML cell-segmentation masks into a clean,
> grouped ground-truth dataset (instance + semantic masks + cell groups).*

**Версія 2.0.0** · Apache-2.0 · pytest **237/237** · Python 3.10+

```
 apps/segmentation  ─▶  data/<dataset>/output/  ─▶  apps/mask_picker  ─▶  deliverable
   (ML-моделі:           (чорнові instance-маски)    (Cleanup·Polygons·    (dense npy +
    Cellpose/InstanSeg/                               Groups + bake)        semantic + groups)
    YOLO)
```

---

## Що це
Сегментаційні моделі помиляються на складних знімках (пропуски, хибні обʼєкти, злиплі
клітини). Розмічати з нуля — дорого. Mask Picker бере **найкращий ML-вихід** і дає
анотатору швидко довести його до ідеалу та структурувати у клітини («1 ядро + N везикул»).

### Три інструменти в одному редакторі
- **Cleanup** — відхилити погані маски + позначити пропуски.
- **Polygons** — домалювати/виправити маски (draw / pick / seed-from-mask).
- **Groups** — обʼєднати інстанси у клітини, класифікувати за правилами.

### Особливості
- Робота з виходами **7 моделей** одночасно (відбір найкращої per-фото).
- Стабільні ID між запіканнями (reserved-range) → групи не «дрейфують».
- Глобальний хронологічний undo/redo на всі інструменти.
- Lazy-bake + автозбереження без гонок; фінальний deliverable у dense `1..N`.
- **237 автотестів**, повна архітектурна документація, browser-верифіковані UI-фікси.

## Складові
| Шлях | Що |
|---|---|
| `apps/mask_picker/` | Flask-редактор (Cleanup/Polygons/Groups) + bake |
| `apps/segmentation/` | драйвер ML-моделей → `output/` |
| `shared/cellsegkit/` | запозичений тулкіт рендеру/експорту (Cryobiology III, MIT — див. `NOTICE`) |
| `cryobiology4/` | reference-код моделей + config (ваги — окремо, не в git) |
| `tools/` | `bake_all.py` (`--pack` deliverable), інваріант-верифікатори |
| `docs/` | технічна документація |

## Швидкий старт
Повна інструкція (встановлення, ваги, чистий ПК) → **[`INSTALL.md`](INSTALL.md)**.

> **Windows:** спершу встанови [Microsoft Visual C++ Redistributable (x64)](https://aka.ms/vs/17/release/vc_redist.x64.exe) — без нього PyTorch не стартує (`OSError: WinError 126`, `c10.dll`).
```bash
pip install -r apps/mask_picker/requirements.txt
pip install -e ./shared/cellsegkit

# Крок 1 — ML-сегментація заповнює output/ (фото класти у data/my_dataset/images/).
# Перший запуск качає модель Cellpose-SAM ~1.2 ГБ → потрібен інтернет (деталі — INSTALL.md §3).
pip install cellpose                                           # + instanseg-torch / ultralytics за потреби
python apps/segmentation/run_segmentation.py --data-dir data/my_dataset

# Крок 2 — відбір і доразмітка у браузері:
python apps/mask_picker/app.py --workspace data/my_dataset    # → http://127.0.0.1:5000
```
Фінальний датасет: `python tools/launchers/bake_all.py --data-dir data/my_dataset --pack`.

### Що класти в `--data-dir` і що на виході
На вході потрібна лише папка `images/` з фото (`.jpg/.jpeg/.png/.tif/.tiff/.bmp`) —
або `.zip`-архів прямо в корені `data-dir`, який розпакується туди автоматично
при першому запуску (кириличні імена та префікси `Копія `/`Copy of ` обробляються самі).
Ніяких анотацій/лейблів на вхід не потрібно — їх генерує сегментація.

```
data/my_dataset/
├── (my_photos.zip)   ← або одразу...
└── images/           ← ...фото сюди (.jpg/.png/.tif/...)
```

Після `run_segmentation.py` з'являється:

```
data/my_dataset/output/
├── cyto2/            ← по підпапці на кожну модель (більше моделей — з `--all`)
│   ├── overlay/      ← PNG-візуалізація з масками
│   ├── png/          ← маски як PNG
│   ├── npy/          ← маски як numpy (.npy)
│   └── yolo/         ← анотації YOLO (.txt)
└── instanseg/ ...
```

Це і є вхід для Кроку 2 (`apps/mask_picker`). Повторний запуск з новими фото в
`images/` — безпечний: вже оброблені фото (усі 4 формати присутні) пропускаються.

### Крок 2 (`apps/mask_picker`) — який вхід і як підставити свою папку
Це браузерний редактор: Cleanup (відбір), Polygons (доправка масок), Groups
(групування в клітини). Вхід — `--workspace <папка>` з тією самою структурою,
що виходить із Кроку 1:

```
<workspace>/
├── images/     ← ті самі фото, що й у Кроці 1
└── output/     ← результат run_segmentation.py (моделі автовиявляються)
```

Усередині `<workspace>` автоматично створюються/використовуються `selected/`,
`skipped/`, `polygons/`, `groups/`, `labels.json`, `group_classes.json`.

Щоб працювати з іншим датасетом — просто вкажи іншу папку (той самий шлях,
що був `--data-dir` у Кроці 1):
```bash
python apps/mask_picker/app.py --workspace data/other_dataset
```

Для тонкого контролю (наприклад, фото і маски лежать у різних місцях) є окремі
прапорці замість `--workspace`: `--images-dir`, `--output-root`, `--selected-dir`,
`--skipped-dir`, `--polygons-dir`, `--groups-dir`. У цьому режимі `--output-root`
має вже містити підпапки моделей з `overlay/` (інакше — помилка при старті).

## Документація
- **[`docs/TECHNICAL_REPORT.md`](docs/TECHNICAL_REPORT.md)** — цілісна технічна записка.
- **[`docs/architecture/`](docs/architecture/README.md)** — детальна архітектура (14 підсистем, front↔back).
- **[`docs/PROJECT_JOURNEY.md`](docs/PROJECT_JOURNEY.md)** — шлях проєкту й складнощі.

## Дані та ваги моделей
Не зберігаються в репо (великі/чутливі), доступні окремо:
- **Розмічені датасети** (vesicles, nuclei): [Google Drive](https://drive.google.com/drive/folders/1jWNuxl-E7uaGRc3ubgug4RKz8cem_QTA).
- **Ваги моделей** (~1.8 ГБ): у **GitHub Releases** — завантажуються **автоматично** при першому
  запуску сегментації (`tools/download_weights.py`). Built-in моделі (cyto2/instanseg) ваг **із Release**
  не потребують, але cellpose 4.x при першому запуску сам качає свою модель **Cellpose-SAM (~1.2 ГБ)** —
  поради для повільної мережі/ВМ у [`INSTALL.md`](INSTALL.md) §3.

## Ліцензія
Apache License 2.0 (`LICENSE`). Запозичені компоненти й ML-моделі — `NOTICE`.
