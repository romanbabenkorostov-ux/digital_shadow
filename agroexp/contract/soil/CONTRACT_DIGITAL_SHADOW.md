# CONTRACT_DIGITAL_SHADOW — машиночитаемая цифровая тень опыта

**Версия:** 1.1.0 · **Код:** `core/twin/` · **Решение:** `obsidian/Decisions/ADR-017-digital-shadow.md`, ADR-019 (слои НСПД)

## Как получить
- `py .\scripts\export_twin.py <код>` → `html\twin\<код>_twin\` + `<код>_twin.zip`;
- `py .\scripts\export_project.py <код> --case` — то же рядом с HTML кейса (`--no-twin` — без тени);
- `py .\scripts\export_twin.py --verify <папка>` — сверка с manifest.json.

## Гарантии
1. **Только чтение.** Тень не меняет cca.db, soil_lab.db, weather.db.
2. **Без ПДн.** Организации — по роли, ОГРН/ИНН; поля `observer`, `signatory_*`, `contacts`, `phone`, `email`, `recorded_by` не выгружаются. Схема запрещает их у организаций и наблюдений.
3. **Детерминизм.** Одинаковые данные → одинаковые байты файлов и `content_sha256`. Время сборки — только `manifest.generated_at`.
4. **Стабильные IRI:** `https://cca.ekcelo.ru/id/<проект>/<Класс>/<ключ>`; ключ — код (делянка `Д-01`, вариант `V0`, гипотеза `H1`, проход `ПР-04`, разрез, контур). Наблюдения нумеруются в порядке канона.
5. **Словарь** — `ssa:` = `https://cca.ekcelo.ru/ns/ssa#` (`ontology.ttl`), привязки `skos:exactMatch` / `closeMatch` к SOSA/SSN, PROV-O, GeoSPARQL, QUDT, MIAPPE (PPEO); префикс AGROVOC заведён, коды терминов — после сверки с тезаурусом.
6. **Совместимость.** Ломающее изменение ключей `shadow.json` или колонок CSV — новая major-версия (`version`), эталон `contracts/digital_shadow/` обновляется в том же PR.

## Состав комплекта
| файл | содержимое |
|---|---|
| `shadow.json` | канон; проверяется `shadow.schema.json` (эталон — `contracts/digital_shadow/shadow.schema.json`) |
| `entities.jsonl` / `edges.jsonl` | граф: `{iri, type, label, props}` / `{from, rel, to}` |
| `claims.jsonl` | `{subject, predicate, value, unit, time, layer, fraction, method, source, data_origin, observation}` |
| `scenarios.jsonl` | по делянке: события (разбивка, отбор, внесение, сев, уборка) и отборы, доли и затраты |
| `shadow.geojson` | слои `field`, `plot`, `harvest_area`, `sampling_point`, `soil_pit`, `soil_map_contour`, `survey_field`, `weather_point`; свойство `iri` |
| `features_plot_season.csv` | строка = делянка × сезон |
| `features_plot_date.csv` | строка = делянка × дата наблюдения |
| `feature_contract.json` / `.md` | по колонке: `dtype, unit, role, source, window, allowed_use, leakage_risk, cv_group, data_origin, hypotheses, endpoints` |
| `ontology.ttl` / `shadow.ttl` | TBox (эталон — `contracts/digital_shadow/ontology.ttl`) / ABox |
| `quality_report.json` / `.md` | readiness (32 проверки), висячие связи, пропуски, происхождение, шкалы, точки вне полосы, ПДн |
| `SPEC.md` | описание комплекта |
| `cadastre_contract.json` | контракт данных кадастра НСПД: группы, слои, `style_policy`, JSON Schema объекта, поля раздела `cadastre` |
| `cadastre/survey.json` | первый проход (ADR-019): паспорт каждого слоя НСПД под опытом, план второго прохода, ограничения |
| `cadastre/layers/<ключ>.json` / `.geojson` | паспорт и объекты ОДНОГО слоя: `L<layer_id>`; `C<категория>` — слой не определён (кандидаты) |
| `cadastre/coverage.json` | делянка × слой: `checked_objects_found` / `checked_no_objects` |
| `manifest.json` | `files[{path, bytes, sha256}]`, `content_sha256`, `code_rev`, `protocol_hash`, `readiness`, `sources` (SHA-256 баз) |

## Роли колонок
`id` · `design` · `treatment` · `covariate` (до воздействия) · `environment` (общая для опыта) · `intermediate` (после воздействия — признак только для поздних целей) · `target` · `economics` · `meta`. CV — leave-повторность-out (`cv_group = replicate`).

## data_origin
`observed` — измерено; `planned` — запланированная фаза; `modelled` — ERA5 / Open-Meteo; `remote_sensing` — Sentinel-2; `calculated` — вычислено из других колонок; `transcribed` — перенесено из документа.

## Слои НСПД (1.1.0, ADR-019)
- Слой — единица: ключ `L<layer_id>`; одна категория НСПД бывает опубликована несколькими слоями — это разные слои.
- У объекта `layer_resolution`:
  - `measured`, `declared`, `single_in_category`, `base_of_category`;
  - `ambiguous` + `layer_candidates` — слой не угадывается.
- Тематический слой — раскраска (`thematic_coloring`, `display_only`): объектов в нём нет, объект — в базовом слое.
- «Не найдено» — только для слоя, который запрашивал прогон; иначе слой в тень не попадает как проверенный.
- Привязка к тени кадастровой карты abn: её словарь `nspd:` ссылается на `ssa:CadastralLayer`, `ssa:CadastralObject`, `ssa:CadastralParcel`, `ssa:drawnIn` через `skos:exactMatch`.
