# Spec_Elem_04 — Blockage (Hộp Chặn)

> **Mục tiêu:** Spec triển khai cho agent. Tuân thủ kiến trúc hiện có: **Data → Factory → Domain → Visual**, **event-driven** (KHÔNG dùng `IEntityDomain.Tick`).
>
> **Level unlock:** 22 · **Element id:** `Spec_Elem_04`
>
> Prefab lấy từ `PrefabProfile`. Event fill box: `BobbinsConveyor.OnBoxCompletedEvent` (đã có sẵn).

---

## 0. Quyết định đã chốt (KHÔNG được tự đổi)

| # | Hạng mục | Quyết định |
|---|----------|-----------|
| 1 | **Bản chất** | Blockage = **vật cản tĩnh** đặt trên 1 tile, chặn pathfinding (ball không đi qua, không đứng lên). Có **số lần chặn** (`blockCount`, 1..5) hiển thị trên hộp + **màu yêu cầu**. |
| 2 | **Trigger giảm** | Lắng nghe `BobbinsConveyor.OnBoxCompletedEvent` (kiểu `Action<BobbinsBox>`). Khi 1 box **fill đầy** có **màu == màu yêu cầu** → `blockCount -= 1`. |
| 3 | **Phạm vi giảm** ✅ | **Mỗi blockage có counter ĐỘC LẬP.** 1 box màu X hoàn thành → **MỌI blockage màu X đều giảm 1** (mỗi cái trừ độc lập trên counter của mình). |
| 4 | **Khi `blockCount == 0`** ✅ | Blockage **biến mất ngay**: `RemoveStaticObstacle(tile)` → tile thành walkable → gọi `BuildTrainConnections()` (raise `OnBoardChanged` để pathfinding + HiddenYarn re-eval) → play break anim → `Destroy`/disable. **KHÔNG spawn gì** tại ô. |
| 5 | **Màu yêu cầu** ✅ | Lưu **palette index** (`int requiredColorId`, đồng nhất YarnTube/WoolBall). So khớp: `ColorsParamSO.GetColorTypeByPaletteIndex(requiredColorId) == box.CurrentColorType`. **Editor color picker CHỈ hiện các màu mà yarnBalls trong level đang dùng** (xem §5.3). |
| 6 | **Placement** | Nhiều blockage/level. Mỗi cái **on-grid, chiếm đúng 1 tile**. Hard-wall suốt khi `blockCount > 0`. |
| 7 | **Tương tác** | Hoàn toàn **passive**: KHÔNG click được, chỉ bị phá bằng cách fill box đúng màu. |
| 8 | **Obstacle channel** | Tái dùng `BoardObstacleMap` + `YarnBoardRuntimeState.AddStaticObstacle/RemoveStaticObstacle/IsStaticObstacle` (đã có từ YarnTube). KHÔNG set `tileData=false`. |
| 9 | **Editor UI** | **Tool mới trong tab YarnBoard** (giống YarnTube), KHÔNG tạo tab riêng. Inspector chỉnh `blockCount` (1–5) + màu yêu cầu. |
| 10 | **Validation** ✅ | **Chỉ cảnh báo (warning, KHÔNG block save)** cho: chặn hết đường ball→Gate, đặt đè Gate, màu yêu cầu không có box tương ứng. **Chỉ HARD-BLOCK** việc đặt đè tile đã có entity khác (ball/tube/blockage) — đây là integrity, không phải gameplay. |

---

## 1. Tổng quan hành vi runtime

```
   [Gate]                         BobbinsConveyor
     ▲                          ┌───────────────┐
     │   ┌──────┐               │ box fill đầy  │
   ball ─┤ [3]X ├─ ball chặn    │  màu X  ✔     │── OnBoxCompletedEvent(box) ─┐
         │ màu X│               └───────────────┘                            │
         └──────┘                                                            ▼
   blockCount=3, hard-wall      mọi Blockage màu X: blockCount-- ; nếu 0 → phá, thông tile
```

1. **Level load:** mỗi `BlockageData` → `BlockageFactory` tạo GameObject từ `PrefabProfile.BlockagePrefab`, đặt tại `tileId`, `runtimeState.AddStaticObstacle(tileId)`. Visual hiện `blockCount` + tint màu yêu cầu.
2. Suốt khi `blockCount > 0`, tile là **vật cản** (pathfinder coi như tường, ball không đi qua/đứng lên). Giống hệt tube tile.
3. Mỗi khi 1 bobbins box fill đầy → `BobbinsConveyor.OnBoxCompletedEvent(box)`:
   - Mỗi `Blockage` đang subscribe kiểm tra `box.CurrentColorType == màu của mình`?
   - Nếu khớp → `blockCount--`, cập nhật visual (số + hiệu ứng "trúng").
   - Nếu `blockCount == 0` → **phá**: remove obstacle, `BuildTrainConnections()` (raise `OnBoardChanged`), play break, destroy.
4. Vì gỡ obstacle gọi `BuildTrainConnections()` → `OnBoardChanged` → HiddenYarn re-eval + train graph cập nhật + đường ball thông.

**Invariant:** counter mỗi blockage độc lập; nhiều blockage cùng màu cùng giảm 1 trên 1 box. Phá là **một chiều**.

---

## 2. DATA LAYER

### 2.1. `Data/BlockageData.cs` (mới)

```csharp
using System;
using UnityEngine;

[Serializable]
public sealed class BlockageData
{
    public string id;                 // optional, tiện debug
    public Vector2Int tileId;         // ô blockage chiếm
    public int requiredColorId;       // PALETTE INDEX — khớp WoolBallData.ColorId convention
    [Range(1, 5)] public int blockCount = 1;   // số lần chặn, 1..5 (validate editor)
}
```

> ⚠️ **Color convention:** `requiredColorId` là **palette index** (như `WoolBallData.ColorId`). `BobbinsBox` lại expose `CurrentColorType` kiểu `WoolColorType`. Khi so khớp PHẢI convert: `ColorsParamSO.GetColorTypeByPaletteIndex(requiredColorId) == box.CurrentColorType`. KHÔNG cast `(int)WoolColorType`.

### 2.2. Sửa `Data/LevelData.cs`

Thêm field (default rỗng để JSON cũ deserialize an toàn):
```csharp
public List<BlockageData> blockages = new List<BlockageData>();
```

### 2.3. Serialization touchpoints (BẮT BUỘC cả 4 chỗ — giống YarnTube §5.4)
1. `YarnBoardEditorWindow.cs` → `class YarnBoardLevelJson`: `public List<BlockageData> blockages = new();`
2. `Serialization.cs` → `class YarnBoardLevelSaveJson`: `public List<BlockageData> blockages;`
3. `Serialization.cs` → `CreateSaveJson(...)`: `blockages = level.blockages ?? new List<BlockageData>(),`
4. `Bobbins.cs` → `CreateRuntimeLevelData(...)`: `blockages = source.blockages,`
5. `CreateDefaultLevel(...)`: `blockages = new List<BlockageData>()`.

---

## 3. DOMAIN LAYER

### 3.1. `Data/PrefabProfile.cs` (sửa)
Thêm theo pattern `YarnTubePrefab`:
```csharp
public static GameObject BlockagePrefab;
```
> **[CHECK]** cách `PrefabProfile` nạp prefab (Resources / serialized field / SO). Nối `BlockagePrefab` cùng cơ chế.

### 3.2. `System/Creation/BlockageCreateParameters.cs` (thêm vào `YarnBoardCreationParameters.cs`)

```csharp
public sealed class BlockageCreateParameters : ICreateParameters
{
    public BlockageCreateParameters(
        BlockageData data,
        BoardSplineDataAdapterInfo adapter,
        Transform parent,
        YarnBoardRuntimeState runtimeState)
    {
        Data = data; Adapter = adapter; Parent = parent; RuntimeState = runtimeState;
    }

    public BlockageData Data { get; }
    public BoardSplineDataAdapterInfo Adapter { get; }
    public Transform Parent { get; }
    public YarnBoardRuntimeState RuntimeState { get; }
}
```

### 3.3. `System/Creation/BlockageFactory.cs` (mới) — khuôn `YarnTubeFactory`

```csharp
public class BlockageFactory : IFactory<Blockage>
{
    public UniTask<Blockage> Create(ICreateParameters parameters, CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        if (parameters is not BlockageCreateParameters p)
            throw new System.ArgumentException($"Expected {nameof(BlockageCreateParameters)}.");

        GameObject instance = PrefabProfile.BlockagePrefab != null
            ? Object.Instantiate(PrefabProfile.BlockagePrefab, p.Parent, false)
            : new GameObject("Blockage");
        if (PrefabProfile.BlockagePrefab == null) instance.transform.SetParent(p.Parent, false);
        instance.name = string.IsNullOrEmpty(p.Data.id) ? $"Blockage_{p.Data.tileId.x}_{p.Data.tileId.y}" : p.Data.id;

        if (!instance.TryGetComponent(out Blockage blockage)) blockage = instance.AddComponent<Blockage>();
        blockage.OnCreated(p);
        return UniTask.FromResult(blockage);
    }
}
```

### 3.4. `Elements/Domain/Blockage.cs` (mới)

`MonoBehaviour, IRuntimeCreatable, IPendingCleanup`. Khuôn `YarnTube`/`HiddenYarn`.

**Fields/Props:**
```csharp
public BlockageData Data { get; private set; }
public BoardSplineDataAdapterInfo Adapter { get; private set; }
public BlockageVisual Visual { get; private set; }
public bool IsPendingCleanup { get; private set; }
public int RemainingCount { get; private set; }      // blockCount hiện tại
public bool IsBroken => RemainingCount <= 0;
public Vector2Int Tile => Data != null ? Data.tileId : Vector2Int.zero;

private YarnBoardRuntimeState runtimeState;
private WoolColorType requiredColorType;              // cache convert từ requiredColorId
private bool subscribed;
private bool breakInFlight;
```

**`OnCreated(ICreateParameters)`** — cast `BlockageCreateParameters`:
- gán Data/Adapter/runtimeState; `RemainingCount = Mathf.Clamp(Data.blockCount, 1, 5);`
- `requiredColorType = ColorsParamSO.GetColorTypeByPaletteIndex(Data.requiredColorId);`
- `transform.localPosition = Adapter.IndexToWorld(Data.tileId);`
- `runtimeState.AddStaticObstacle(Data.tileId);`  *(lưu ý: cũng được seed sớm ở LevelSpawner — idempotent, xem §3.6)*
- `Visual = EnsureVisualChild().GetOrAddComponent<BlockageVisual>(); Visual.Render(Data, Adapter); Visual.SetCount(RemainingCount); Visual.SetColor(Data.requiredColorId);`
- **Subscribe:** `BobbinsConveyor.OnBoxCompletedEvent += OnBoxCompleted; subscribed = true;`

**`OnBoxCompleted(BobbinsBox box)`** (cốt lõi):
```csharp
private void OnBoxCompleted(BobbinsBox box)
{
    if (IsBroken || IsPendingCleanup || box == null) return;
    if (box.CurrentColorType != requiredColorType) return;   // khác màu → bỏ qua

    RemainingCount--;
    Visual?.SetCount(RemainingCount);
    Visual?.PlayHit();

    if (RemainingCount <= 0)
        Break();
}
```

**`Break()`**:
```csharp
private void Break()
{
    if (breakInFlight) return;
    breakInFlight = true;

    UnsubscribeBoxCompleted();
    runtimeState?.RemoveStaticObstacle(Tile);
    runtimeState?.BuildTrainConnections();   // raise OnBoardChanged → pathfinding/HiddenYarn re-eval

    PlayBreakAndDestroy().Forget();           // anim rồi Destroy(gameObject); nếu skip anim thì Destroy thẳng
}
```
> **Quan trọng — thứ tự:** `RemoveStaticObstacle` TRƯỚC `BuildTrainConnections` để khi `OnBoardChanged` raise, tile đã thông ⇒ HiddenYarn/ball thấy đường mới ngay.

**`IPendingCleanup.CleanupForLevelUnload()`**: `IsPendingCleanup = true; UnsubscribeBoxCompleted(); Visual?.StopAllTweens();`
**`OnDestroy()`**: `UnsubscribeBoxCompleted();` (an toàn, tránh leak static event).

`UnsubscribeBoxCompleted()`: `if (subscribed) { BobbinsConveyor.OnBoxCompletedEvent -= OnBoxCompleted; subscribed = false; }`

> ⚠️ **Static event leak:** `OnBoxCompletedEvent` là **static**. Bắt buộc unsubscribe ở cả `CleanupForLevelUnload` và `OnDestroy`, nếu không blockage cũ vẫn nhận event sau khi level unload → NRE/giảm nhầm.

> **Tách domain↔visual:** `Blockage` không đụng mesh/material trực tiếp; mọi thứ qua `BlockageVisual` API.

### 3.5. Wiring trong `System/Creation/LevelSpawner.cs`

Thêm `private readonly BlockageFactory blockageFactory = new();`

**(a) Seed obstacle SỚM** — giống fix YarnTube (tránh HiddenYarn lộ nhầm xuyên blockage). Trong `SeedYarnTubeObstacles`-style, thêm seed blockage **TRƯỚC** `BuildTrainConnections()` đầu tiên:
```csharp
private void SeedBlockageObstacles(LevelData level)
{
    if (level?.blockages == null) return;
    for (var i = 0; i < level.blockages.Count; i++)
        if (level.blockages[i] != null)
            runtimeState.AddStaticObstacle(level.blockages[i].tileId);
}
```
Gọi `SeedBlockageObstacles(level)` cạnh `SeedYarnTubeObstacles(level)` (ngay trước `runtimeState.BuildTrainConnections();`).

**(b) Spawn entities** — sau `BuildTrainConnections()` (cùng chỗ `SpawnYarnTubes`):
```csharp
await SpawnBlockages(level, adapter, boardRoot, token);
```
```csharp
private async UniTask SpawnBlockages(LevelData level, BoardSplineDataAdapterInfo adapter, Transform boardRoot, CancellationToken token)
{
    if (level?.blockages == null || level.blockages.Count == 0) return;
    for (var i = 0; i < level.blockages.Count; i++)
    {
        token.ThrowIfCancellationRequested();
        var data = level.blockages[i];
        if (data == null) continue;
        await blockageFactory.Create(new BlockageCreateParameters(data, adapter, boardRoot, runtimeState), token);
    }
}
```

---

## 4. VISUAL LAYER — `Elements/Visual/BlockageVisual.cs` (mới)

Khuôn `YarnTubeVisual` (`MonoBehaviour`, `Render(BlockageData, Adapter)`).

Trách nhiệm:
1. **Mesh hộp** từ `PrefabProfile.BlockagePrefab` (factory đã instantiate prefab; visual chỉ điều khiển). Nếu prefab đã chứa sẵn mesh/child thì `Render` chỉ canh vị trí/scale theo `Adapter.CellSize`.
2. **Tint màu yêu cầu:** `SetColor(int paletteIndex)` → `var c = ColorsParamSO.GetColorByPaletteIndex(paletteIndex);` apply lên renderer (dùng `VisualColorReviewSO.ApplyToRenderer(...)` nếu phù hợp, hoặc material `_BaseColor`). **[CHECK]** pattern apply màu trong prefab Blockage.
3. **Counter:** `SetCount(int n)` → TextMeshPro hiển thị số lần chặn trên hộp.
4. **Animation:** `PlayHit()` (rung/nháy khi -1), `PlayBreak()` (vỡ + fade), `StopAllTweens()`.

> Nếu prefab chưa có TMP/anim, agent dùng AskUserQuestion xác nhận asset trước khi ráp (xem §7).

---

## 5. EDITOR — Tool mới trong tab YarnBoard

Tạo partial mới **`Editor/C#/YarnBoardEditorWindow.Blockage.cs`** cho logic riêng; đụng tối thiểu file chung. Mirror toàn bộ pattern YarnTube (Spec_Elem_02 §5).

### 5.1. Tool enum + nút (`YarnBoardEditorWindow.cs` + `Workspace.cs` + UXML/USS)
- `enum EditorToolMode { ..., YarnTube, Blockage }` + field `_blockageTool` (Button).
- Wiring tool ở **`Workspace.cs`**: `_blockageTool.clicked += () => SetTool(EditorToolMode.Blockage);` + active-class `EnableInClassList("selected", _currentTool == EditorToolMode.Blockage)`.
- UXML thêm `Button name="blockage-tool"` trong board tool group; USS style giống tool khác.
- State: `private int _selectedBlockage = -1;` reset ở `LoadLevelFromFullPath` + khi đổi tool/level.

### 5.2. Placement (cell click) — `Workspace.cs`
Nhánh `EditorToolMode.Blockage`:
- click ô đã có blockage → select (`_selectedBlockage`).
- ô trống → tạo `BlockageData { tileId = cell, requiredColorId = <màu đầu tiên trong palette dùng bởi level>, blockCount = 1 }`, add vào `_currentLevel.blockages`, select, `MarkDirty()`.
- **HARD-BLOCK đặt đè** (integrity): ô inactive (`tileData=false`), ô đã có yarn ball / tube / blockage khác → từ chối + báo.
- **Đặt đè Gate / chặn hết lối ra**: VẪN cho đặt nhưng đưa vào **warning** (xem §5.4).

### 5.3. Inspector (`YarnBoardEditorWindow.Blockage.cs`, gọi từ `Inspector.cs`)
Khi `_selectedBlockage >= 0`:
- **Block count:** slider/IntegerField **1..5** → `data.blockCount`, `MarkDirty()`, refresh preview.
- **Màu yêu cầu (palette index):** dropdown/swatch — ✅ **CHỈ hiện các màu mà yarnBalls trong level đang dùng**:
  ```csharp
  // tập palette index xuất hiện trong level
  var usedColors = new HashSet<int>();
  foreach (var b in _currentLevel.Balls) if (b != null) usedColors.Add(b.ColorId);
  // render swatch cho từng usedColors (dùng ColorsParamSO/_colorsParam như Bobbins/YarnTube)
  ```
  Chọn màu → `data.requiredColorId`, `MarkDirty()`, refresh preview swatch.
  > Nếu `usedColors` rỗng (level chưa có ball) → hiện toàn palette + warning "level chưa có ball nào".
- **Tile:** read-only (đặt lại bằng tool).
- Nút **Delete blockage**.

### 5.4. Validation (`Validation.cs` → `ValidateCurrentLevel`) — ✅ chỉ WARNING
Với mỗi blockage:
- `blockCount` ngoài [1,5] → kẹp lại + warning (hoặc error nhẹ; mặc định warning).
- **Đặt đè Gate** (`tileId == targetExitTileId`) → **warning** (không block save).
- **Màu yêu cầu không có box tương ứng:** nếu trong `level.bobbins` không có box màu == `requiredColorId` → blockage không bao giờ phá được → **warning**.
- **Chặn hết lối ra:** chạy BFS editor (coi mọi blockage là tường): nếu CÓ yarn ball mất hết đường tới Gate → **warning** (không block save).
  - **[CHECK]** dùng BFS tay trên `tileData` + occupancy ball/tube/blockage (giống cách HiddenYarn validation §5.2 của Elem_03), 1 Gate nên nhẹ.
- **HARD-BLOCK (error, chặn save):** chỉ khi đặt đè tile đã có entity khác (ball/tube/blockage) hoặc ô inactive — integrity.

### 5.5. Scene/Board preview (`ScenePreview.cs` / `Preview.cs`)
- Cell blockage: vẽ ô đặc trưng (icon hộp) + **số `blockCount`** + **viền/fill theo màu yêu cầu** (swatch palette).
- 3D preview (tùy chọn, phase sau): dựng `BlockageVisual` preview.

---

## 6. Edge cases (agent phải xử lý)

1. **Static event leak:** PHẢI unsubscribe `OnBoxCompletedEvent` ở `CleanupForLevelUnload` + `OnDestroy`. Level reload mà quên → blockage zombie nhận event.
2. **Nhiều blockage cùng màu:** 1 box hoàn thành → mọi blockage màu đó cùng -1 (mỗi cái counter độc lập) — đúng quyết định #3.
3. **Màu không có box:** blockage không bao giờ phá → tường vĩnh viễn. Editor cảnh báo; runtime không crash.
4. **Phá xong mở đường cho HiddenYarn/ball:** `Break()` gọi `BuildTrainConnections()` ⇒ `OnBoardChanged` ⇒ HiddenYarn re-eval + train graph cập nhật. KHÔNG bỏ bước này.
5. **HiddenYarn lộ nhầm xuyên blockage lúc spawn:** đã chặn bằng `SeedBlockageObstacles` TRƯỚC `BuildTrainConnections` đầu tiên (§3.5a).
6. **Box hoàn thành dồn dập:** mỗi event -1; `PlayHit` phải robust khi gọi liên tiếp (kill tween cũ). Nếu nhiều event đưa count<0 trong 1 frame → `Break()` guard `breakInFlight` chống phá 2 lần.
7. **Level unload giữa lúc break:** `PlayBreakAndDestroy` bọc `try/catch OperationCanceledException`; cleanup dừng tween.
8. **`blockCount` sửa tay JSON > 5 hoặc < 1:** runtime `Clamp(1,5)` khi OnCreated; editor validate.
9. **Đặt blockage đè Gate (JSON tay):** tile Gate thành tường → có thể không hoàn thành level; chỉ cảnh báo theo quyết định #10.

---

## 7. Điểm tunable (đã có default — agent xác nhận khi chạm tới)

| Q | Default | Ghi chú |
|---|---------|--------|
| Anim khi -1 / khi phá | **PlayHit rung + PlayBreak vỡ/fade** | Tối thiểu: cập nhật số + destroy tức thì. |
| Counter render | **TMP world-space trên hộp** | Theo prefab; nếu prefab có sẵn slot text thì dùng. |
| Apply màu lên prefab | **`VisualColorReviewSO.ApplyToRenderer` / `_BaseColor`** | [CHECK] theo cấu trúc prefab Blockage. |
| Phá xong: Destroy hay disable | **Destroy(gameObject)** | Không còn vai trò; obstacle đã remove. |
| 3D scene preview editor | **Phase sau**, ưu tiên 2D cell (số+màu) | |

> Gặp điểm cần asset/quyết định lớn ngoài bảng → dùng AskUserQuestion trước khi code.

---

## 8. Acceptance criteria / Test plan

**Gameplay**
- [ ] Blockage hiện đúng `blockCount` + màu yêu cầu; là tường (ball/pathfinder không đi qua/đứng lên).
- [ ] Fill đầy 1 box đúng màu → mọi blockage cùng màu giảm 1 (counter độc lập).
- [ ] Fill box khác màu → blockage không đổi.
- [ ] `blockCount` về 0 → blockage biến mất, tile thông ngay, ball đi qua được.
- [ ] Phá blockage mở đường → HiddenYarn cạnh đó re-eval/lộ đúng (tích hợp với Elem_03).
- [ ] Nhiều blockage/level + nhiều màu hoạt động độc lập.
- [ ] Level reload nhiều lần KHÔNG bị giảm nhầm (event đã unsubscribe sạch).

**Editor**
- [ ] Tool Blockage đặt/chọn/xóa; chỉnh count 1–5; màu chỉ hiện các màu ball trong level dùng.
- [ ] HARD-BLOCK đặt đè ball/tube/blockage/ô inactive.
- [ ] WARNING (không chặn save): đè Gate, chặn hết lối ra, màu không có box tương ứng.
- [ ] Save → load round-trip giữ `blockages`. JSON cũ (không field) load ok.
- [ ] Play Level từ preview dựng đúng blockage.

**Verify cuối:** level test có Gate + 2 blockage cùng màu + 1 blockage khác màu + bobbins box các màu; play, fill box, kiểm 7 tiêu chí gameplay; round-trip save/load + diff JSON.

---

## 9. File-by-file checklist

**Mới**
- `Data/BlockageData.cs`
- `Elements/Domain/Blockage.cs`
- `Elements/Visual/BlockageVisual.cs`
- `System/Creation/BlockageFactory.cs`
- `Editor/C#/YarnBoardEditorWindow.Blockage.cs`

**Sửa**
- `Data/LevelData.cs` → field `blockages`
- `Data/PrefabProfile.cs` → `BlockagePrefab`
- `System/Creation/YarnBoardCreationParameters.cs` → `BlockageCreateParameters`
- `System/Creation/LevelSpawner.cs` → `blockageFactory` + `SeedBlockageObstacles` (trước BuildTrainConnections) + `SpawnBlockages`
- `Editor/C#/YarnBoardEditorWindow.cs` → enum tool, `_blockageTool`, query nút, `YarnBoardLevelJson.blockages`, `_selectedBlockage`
- `Editor/C#/YarnBoardEditorWindow.Workspace.cs` → wiring tool + cell-click placement nhánh `Blockage`
- `Editor/C#/YarnBoardEditorWindow.Serialization.cs` → save JSON + `CreateSaveJson` + `CreateDefaultLevel`
- `Editor/C#/YarnBoardEditorWindow.Bobbins.cs` → `CreateRuntimeLevelData`
- `Editor/C#/YarnBoardEditorWindow.Inspector.cs` → gọi panel inspector blockage (render khi `_selectedBlockage >= 0`)
- `Editor/C#/YarnBoardEditorWindow.Validation.cs` → rule blockage (warning)
- `Editor/C#/YarnBoardEditorWindow.ScenePreview.cs` → vẽ count + màu
- `Editor/UXML/YarnBoardEditorWindow.uxml` + `Editor/USS/YarnBoardEditorWindow.uss` → nút tool

**KHÔNG cần sửa:** `YarnBoardRuntimeState` obstacle API + `OnBoardChanged` đã có sẵn (từ YarnTube/HiddenYarn).

**Thứ tự build đề xuất:** Data (`BlockageData`+LevelData) → Domain `Blockage`+Factory+CreateParameters (subscribe event, decrement, break) → Spawner wiring (seed sớm + spawn) → test bằng JSON tay (1 blockage + bobbins box đúng màu) → Visual (count+màu+anim) → Editor tool/inspector(màu lọc theo level)/serialization/validation/preview.

---

## 10. Khác biệt so với Spec_Elem_02 (YarnTube) & Elem_03 (HiddenYarn)

| | YarnTube | HiddenYarn | **Blockage** |
|---|----------|-----------|--------------|
| Loại entity | GO riêng | Modifier trên WoolBall | **GO riêng (1 tile)** |
| Data | List `yarnTubes` | Cờ `isHiddenYarn` | **List `blockages`** |
| Trigger | Auto eject khi mouth trống | Path tới Gate thông | **Box đúng màu fill đầy (`OnBoxCompletedEvent`)** |
| Counter | Tổng ball còn lại | — | **Số lần chặn (1–5), giảm theo box** |
| Obstacle | `AddStaticObstacle` (giữ khi rỗng) | Không (chỉ giấu màu) | **`AddStaticObstacle`; Remove khi count=0** |
| Kết thúc | Empty tube ở lại làm tường | Reveal một chiều | **Phá → biến mất → tile thông** |
| Editor | Tool + color groups | Toggle inspector ball | **Tool + count(1–5) + màu (lọc theo màu ball level)** |
