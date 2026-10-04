# Задача: магический портал (способность, LVL 200)

Источник: прошлый чат с claude.ai (перенос в Claude Code) + `src (2).zip`.
Пакет мода: `com.invfix`, modid `sable_compat`, NeoForge 1.21.1.

## Суть
Способность, открывающая портал в духе MCU («Шан-Чи»): ободок из искр, вид сквозь
портал (рендер на базе Immersive Portals), проход сущностей, снарядов способностей
и построек Sable, если они влезают в рамку.

## Требования (решено)
- Уровень: 200.
- Настройки на Shift + клавиша способности, по образцу пушки Рика
  (`rick/portalgun`): все 5 режимов `PortalMode` — FIFO(cycle), LIFO(override),
  MULTI_PAIR, ROOT(hub), POINT; лимит проходов (открытий/закрытий); точки.
- Размер портала 1–5 блоков, выбирается.
- Плоскость — любая из трёх:
  - начальный портал: на грани блока — плоскость грани; в воздухе — по углам взгляда;
  - режим координат (POINT): выбор в настройках, по умолчанию «авто» = как у ведущего портала.
- Дальность: настраивается, максимум 60 блоков. Если цель-блок ближе дальности —
  портал на блоке, иначе в воздухе на заданном расстоянии. Снаряда нет, портал просто открывается.
- Проход только с лицевой стороны. Задняя сторона тёмная и твёрдая (как стена).
- Проходит всё, что влезает в рамку:
  - постройки Sable — целиком через `RigidBodyHandle.teleport(pos, rot)`;
  - способности по размеру: Heat Vision, Neutron Beam (1 блок) — проходят; Heat Disruption — нет.
- Синхронизация сущностей с той стороны (мобы видны и при дальнем выходе) — обязательна.
- Портал в портале: до 10 уровней вложенности, оптимизированно; под шейдерами
  глубина может урезаться по порогу площади на экране.
- Наши VFX: по эту сторону — как обычно, поверх всего; по ту сторону — рисовать
  в проходе портала с камерой прохода, внутри рамки. Если это слишком дорого,
  допустимо не показывать их сквозь портал, но рядом с игроком они пропадать не должны.
- Шейдеры (Iris) поддерживаются.
- Другие измерения: пока без вида внутрь (только телепорт). Мультимировой рендер IP — позже.
- Анимация открытия/закрытия плавная. Звуков пока нет.

## Совместимость (перепроверено в Claude Code по присланным jar)
| Мод | Что нужно |
|---|---|
| Sodium 0.8.12 | 3 миксина IP переписать: `OcclusionCuller.findVisible` (RenderSectionVisitor), `Viewport.isBoxVisible` (int,int,int → testSection), `RenderSectionManager` (sectionCollector, дерево секций). Остальные 6 совпадают. |
| Iris 1.8.14 | Миксины и API IP совпадают. |
| Flywheel 1.0.6 | Compat IP под 0.6 не подходит; добавить плоскость отсечения в шейдеры Flywheel. |
| Sable 2.0.1 | Конфликт `@Redirect` на `collide` в `Entity.move` — переписать коллизию без конфликта; корабли в виде портала нужно отсекать плоскостью. |
| Veil 4.1.4 (jarjar внутри Sable) | Миксины на LevelRenderer/GameRenderer/RenderTarget — проверить логику по декомпайлу. |
| EntityCulling 1.10.5 | На время прохода портала отключать отсечение. |
| Vista 4.3.1 | Защита от рендера порталов внутри её трансляций. |
| C2ME 0.4.0 (notickvd) | Проверить отправку чанков для дальних порталов. |
| Chunky | Конфликтов нет. |

## Конфликты миксинов IP с модами (сверено автоматически по всем миксинам)
Владелец способности — Metroman. IP: 6.0.7, исходники (Apache-2.0), в сборку не ставится; переносим только нужную часть.

Жёсткие (при переносе писать иначе):
- `Entity.move` → `collide`: `@Redirect` у IP и у Sable. Делаем `@Inject` в `Entity.collide`, как в `EntityFastCollideMixin`.
- `ChunkMap$TrackedEntity.updatePlayer`: IP делает `@Overwrite` пустым, а Sable там же `@Redirect` на `Entity.position()` с require=1 → краш при загрузке. Синхронизацию сущностей пишем своей, без overwrite.
- `PlayerList.broadcast`: `@Overwrite` и у IP, и у Sable — работает только один. Не переносим (нужно только для мультимира).
- `ChunkMap.getPlayers` (overwrite IP, inject Sable), `PlayerChunkSender.onChunkBatchReceivedByClient` (overwrite IP, inject C2ME notickvd) — часть чанковой синхронизации IP; не переносим, пока нет дальних порталов и других измерений.
- `Player.canPlayerFitWithinBlocksAndEntitiesWhen`: overwrite IP; WrapOperation Sable на `noCollision` в нём сохраняется (overwrite вызывает `noCollision`). Лучше заменить на inject.

Мягкие (работают вместе): `Entity.getInBlockState` (overwrite Sable + inject IP), `Frustum.offsetToFullyIncludeCameraCube` (overwrite IP + inject Vista), `LevelRenderer.setupRender/isSectionCompiled/renderSectionLayer` (overwrite Sodium, redirect IP с require=0), `GameRenderer.renderLevel` (WrapOperation IP + Redirect Iris на rotation).

Наш мод: пересечение только `FxLevelRendererMixin` (inject HEAD `LevelRenderer.renderLevel`) — нужна защита от повторной отрисовки VFX в проходе портала.

## Открытые вопросы
- Название способности.
