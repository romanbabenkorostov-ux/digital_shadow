# CONTRACT_DIGITAL_SHADOW — машиночитаемая цифровая тень опыта

**Версия:** 1.4.0 · **Код:** `core/twin/` · **Решение:** `obsidian/Decisions/ADR-017-digital-shadow.md`, ADR-019 (слои НСПД)

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
| `claims.jsonl` | `{subject, predicate, value, unit, time, layer, fraction, method, source, data_origin, observation, status, check_outcome, status_by, document_revision, raw_value, corrected}` |
| `scenarios.jsonl` | по делянке: события (разбивка, отбор, внесение, сев, уборка) и отборы, доли и затраты |
| `shadow.geojson` | слои `field`, `plot`, `harvest_area`, `sampling_point`, `soil_pit`, `soil_map_contour`, `survey_field`, `weather_point`; свойство `iri` |
| `features_soil_summary.csv` | плоская сводка: показатель × глубина × метод → даты или события отбора, число фактических значений, статус проверки |
| `features_plot_season.csv` | строка = делянка × сезон |
| `features_plot_date.csv` | строка = делянка × дата наблюдения |
| `feature_contract.json` / `.md` | по колонке: `dtype, unit, role, source, window, allowed_use, leakage_risk, cv_group, data_origin, hypotheses, endpoints` |
| `ontology.ttl` / `shadow.ttl` | TBox (эталон — `contracts/digital_shadow/ontology.ttl`) / ABox |
| `quality_report.json` / `.md` | readiness (набор проверок), висячие связи, пропуски, происхождение, шкалы, точки вне полосы, ПДн |
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

## Статусы и кто их присваивает (1.2.0)
Две оси у каждого лабораторного значения (`soil_lab` 0008):
- **`status`** — состояние значения: `valid`, `suspect` (сомнительно), `not_measured` (не определялось, не ноль), `invalid` (недостоверно). Флаг `corrected` — значение изменено ревизией документа.
- **`check_outcome`** — итог проверки: `not_checked`, `algo_passed`, `algo_flagged`, `expert_confirmed`, `rejected`.
- **`status_by`** — кто присвоил: `algo:<правило>` — алгоритм; `lab:<документ>-r<ревизия>` — лаборатория переиздала документ; `expert:<роль>` — человек (без ФИО). `invalid`, `expert_confirmed`, `rejected` ставит только эксперт.
- В признаки (`features_*.csv`) не попадают значения `suspect`, `invalid`, `rejected`; в `claims.jsonl` они остаются со статусом.

## Проверки
Справочник — таблица `qc_rule` и лист xlsx «Правила проверок» (правило, что проверяется, уровень, кто присваивает, какой статус ставит, версия). Выполняет алгоритм при загрузке документа: номер документа, план/факт даты отбора, не измерено, сумма слоёв, разрыв профиля, копия профиля другой делянки, нулевое значение без «< ПО», суммы фракций, диапазон, глубина, покрытие вариантов. Решение эксперта — отдельное правило `expert_review`.

## Журнал исправлений
Таблица `measurement_revision`, только дописывается: документ и его ревизия, образец, делянка, глубина, показатель, номер правки, было/стало (значение, текст, статус), причина, кто, когда. Переиздание ведомости не удаляет прежнее значение, а переносит его в журнал; решение эксперта тоже пишется в журнал.

## Исходное значение и документ
У значения: `raw_value_text` (как в документе при первой загрузке, не меняется), `raw_unit`, `source_ref` («таблица 1, строка 5»). Документ: номер, дата, ревизия, `file_sha256`, `scan_path`/`scan_sha256`. Трассировка в xlsx («Данные»): документ → образец → делянка → глубина → показатель → значение → проверка → итоговый статус.

## Паспорт почвы
`plot_soil_passport`: классификационное название и система классификации, положение в рельефе, глубины отбора, исходные характеристики, определённые в опыте; методы — у каждого измерения (`method_code`, стандарт). Незаполненное поле — «не заполнено», значения по умолчанию не подставляются; недостающее попадает в `Техдолг_<код>.xlsx` (`scripts/export_techdebt.py`).

## Плоская сводка
`features_soil_summary.csv` и лист «Сводка показателей»: `Почвенный показатель | Глубина | Метод | Даты или события отбора | Число фактических значений | Статус проверки`. Число считает только `observed` и `transcribed`; плановые, модельные, расчётные и «не измерено» не входят. Статус — худший в группе с разбивкой. CLI: `scripts/export_soil_summary.py <код>`.

## Слои НСПД (1.1.0, ADR-019)
- Слой — единица: ключ `L<layer_id>`; одна категория НСПД бывает опубликована несколькими слоями — это разные слои.
- У объекта `layer_resolution`:
  - `measured`, `declared`, `single_in_category`, `base_of_category`;
  - `ambiguous` + `layer_candidates` — слой не угадывается.
- Тематический слой — раскраска (`thematic_coloring`, `display_only`): объектов в нём нет, объект — в базовом слое.
- «Не найдено» — только для слоя, который запрашивал прогон; иначе слой в тень не попадает как проверенный.
- Привязка к тени кадастровой карты abn: её словарь `nspd:` ссылается на `ssa:CadastralLayer`, `ssa:CadastralObject`, `ssa:CadastralParcel`, `ssa:drawnIn` через `skos:exactMatch`.

## Привязка к Crop Ontology (1.3.0, additive, ADR-020)
Если исход протокола привязан к переменной Crop Ontology, в комплект добавляются: `co_variable` и `co_match` у колонки-цели в `feature_contract.json`; поля `co_variable_id`, `co_match`, `co_snapshot_sha256`, `co_confirmed` у исхода в `shadow.json`; `skos:exactMatch` / `skos:closeMatch` на `https://cropontology.org/rdf/<ID>` в `shadow.ttl`; `sources.ontology[]` в `manifest.json`; файл `ONTOLOGY_ATTRIBUTION.md`. Определения, формулы и русские подписи словаря в комплект не входят. Без привязок комплект отличается от версии 1.2.0 номером версии и разделом `design_issues` (1.4.0, ниже; `CONTRACT_ONTOLOGY.md`).

## Реестр проблем постановки опыта (1.4.0, additive)
`shadow.json` → `design_issues[]`: `{iri, code, title, category, severity (block|warn|info), hypothesis_codes, evidence, consequence, next_experiment, source_rule, rule_version, status (open|mitigated|accepted|closed), status_note, status_by}`. Класс `ssa:DesignIssue`, связи `ssa:issueOf` (опыт) и `ssa:limits` (гипотеза).
- Проблема выводится по данным (`core/experiment/design_audit.py`, правила версионируются): покрытие контрастов гипотез измерениями в скважинах, уровень объединения проб (вариант вместо делянки), ранговая связь номера повторности и варианта с положением, геометрия делянок и учётных полос, MDD и число повторностей по формуле `protocol.mdd_pct`, составной фактор, а также уроки проверок готовности (рандомизация, доза, паспорта партий, NPK-эквивалент, история поля).
- `evidence` - вывод из данных; `consequence` - что нельзя утверждать; `next_experiment` - требование к закладке следующего опыта (`scripts/design_audit.py plan`).
- Проблема не равна ошибке исполнителя. Статус ставит человек (`scripts/design_audit.py status`, роль без ФИО), хранится в таблице `design_issue`, пересчёт его не перезаписывает.
- Схему размещения нового опыта проверяют до закладки: `scripts/design_audit.py layout plots.json --scheme plots_scheme.json` (полнота повторностей, связь номера повторности и варианта с положением).
