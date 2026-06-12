# Spec_Elem_02 — Yarn Tube (Ống Len)

> **Mục tiêu:** Spec triển khai cho agent. Tuân thủ kiến trúc hiện có: **Data → Factory → Domain → Visual**, event-driven (không dùng `IEntityDomain.Tick` — interface này tồn tại nhưng chưa được wire ở đâu cả).
>
> **Level unlock:** 10 · **Element id:** `Spec_Elem_02`

---

## 0. Quyết định đã chốt (KHÔNG được tự đổi)

| # | Hạng mục | Quyết định |
|---|----------|-----------|
| 1 | **Eject trigger** | Auto: khi ô **liền kề trước miệng ống** (`tile + dir`) trống thì tube đẩy ngay 1 ball. Liên tục cho tới khi queue rỗng hoặc miệng bị chặn. |
| 2 | **Placement** | Nhiều tube/level. Mỗi tube = **on-grid, chiếm đúng 1 tile**. Hướng đẩy `dir` ∈ {Up, Down, Left, Right} chỉnh **thủ công** trong editor. |
| 3 | **Color order** | **Nhóm liền nhau, FIFO.** Mỗi nhóm = 1 màu + count. Đẩy hết nhóm đầu rồi tới nhóm sau theo thứ tự author khai báo. |
| 4 | **Slide distance** | Ball được đẩy **trượt thẳng tới ô trống xa nhất** theo `dir` (cho tới khi gặp ball khác / blockage / tube khác / mép board). Lấp đầy từ xa về gần miệng ống. |
| 5 | **Ball nature** | Ball sau khi đẩy ra là **WoolBall thường, 1 tile** (`ColorId`), tham gia đầy đủ train/merge/dispatch. **Tái dùng `WoolBallFactory`.** |
| 6 | **Head counter** | Hiển thị **tổng số ball còn lại** trong ống (cộng mọi nhóm màu). Giảm mỗi lần eject. |
| 7 | **Empty tube** | Khi count = 0, tube **ở lại như vật cản rỗng**: vẫn chiếm tile (blockage), không eject nữa, không biến mất. |
| 8 | **Editor UI** | **Tool mới trong tab YarnBoard** (giống `YarnBall` tool), KHÔNG tạo tab riêng. |

---

## 1. Tổng quan hành vi runtime

```
        dir = Right
   ┌─────┐
   │TUBE │→  (ball)  (ball)  (ball) [■blockage]
   │ [6] │   ← lấp đầy từ xa về gần ───────
   └─────┘
   mouth = tile + dir
```

1. Tube đặt tại `tile`, hướng `dir`. Tile của tube là **static obstacle** suốt vòng đời level (ball không đi qua, pathfinder coi là tường).
2. Queue nội bộ = danh sách `ColorId` đã expand từ các color group, theo thứ tự FIFO. Ví dụ groups `[{Blue,4},{Red,2}]` → queue `[Blue,Blue,Blue,Blue,Red,Red]`.
3. Mỗi frame (Update) — hoặc mỗi khi board occupancy đổi — tube kiểm tra:
   - Queue còn ball? **và** không có eject đang chạy? **và** ô `mouth = tile + dir` đang **trống & walkable**?
   - Nếu đủ điều kiện → **eject 1 ball**:
     - Tính **landing tile** = ô trống xa nhất theo `dir` bắt đầu từ `mouth` (raycast occupancy, xem §3.3).
     - Pop màu ở đầu queue, tạo `WoolBallData` tại landing tile, gọi `WoolBallFactory.Create(...)` → ball được register vào `YarnBoardRuntimeState` (occupancy set ngay tại landing tile).
     - Phát hiệu ứng push của tube + ball slide-in (visual). Giảm counter.
     - Sau khi ball landed: gọi `runtimeState.BuildTrainConnections()` để rebuild train links (ball mới có thể merge/nối với ball cùng màu).
4. Vì ball mới là WoolBall thường, khi player clear ball nằm sát miệng ống → `mouth` trống lại → vòng lặp tiếp tục.
5. Queue rỗng → tube chuyển trạng thái **empty** (vẫn là obstacle, ẩn/đổi counter về 0).

**Lưu ý quan trọng về "auto + slide-to-farthest":** vì occupancy được set ngay tại landing tile khi tạo ball, lần eject kế tiếp tính landing tile mới sẽ tự động dừng ở ô gần hơn 1 bậc. Line lấp dần từ xa về miệng; khi ball chỉ còn dừng được ngay tại `mouth` thì `mouth` bị chiếm → tube ngừng đẩy. Đây là invariant chính, agent phải bảo toàn.

---

## 2. DATA LAYER

### 2.1. Enum mới — `Data/YarnTubeDirection.cs` (hoặc thêm vào `GlobalEnum.cs`)

```csharp
public enum YarnTubeDirection { Up, Down, Left, Right }

public static class YarnTubeDirectionExtensions
{
    public static Vector2Int ToVector(this YarnTubeDirection d) => d switch
    {
        YarnTubeDirection.Up    => Vector2Int.up,
        YarnTubeDirection.Down  => Vector2Int.down,
        YarnTubeDirection.Left  => Vector2Int.left,
        YarnTubeDirection.Right => Vector2Int.right,
        _ => Vector2Int.up
    };

    // Góc Y (độ) để xoay visual ống cho khớp hướng. Agent canh theo prefab thực tế.
    public static float ToYaw(this YarnTubeDirection d) => d switch
    {
        YarnTubeDirection.Up    => 0f,
        YarnTubeDirection.Right => 90f,
        YarnTubeDirection.Down  => 180f,
        YarnTubeDirection.Left  => 270f,
        _ => 0f
    };
}
```

> Lưu ý: trục board ↔ Vector2Int do `BoardSplineDataAdapterInfo.IndexToWorld` quyết định. Agent phải kiểm tra hướng `Up/Down` map đúng chiều world (xem cách `WoolBall`/`Adapter` dùng). `ToYaw` có thể cần hiệu chỉnh theo prefab — đánh dấu là **[CHECK ở scene]**.

### 2.2. `Data/YarnTubeData.cs` (mới)

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

[Serializable]
public sealed class YarnTubeColorGroup
{
    public int colorId;             // PALETTE INDEX — KHỚP hệt WoolBallData.ColorId (xem ⚠️ bên dưới)
    [Min(2)] public int count = 2;  // điều kiện: chẵn & >= 2 (validate ở editor)
}

[Serializable]
public sealed class YarnTubeData
{
    public string id;                                   // optional, tiện debug
    public Vector2Int tileId;                           // ô tube chiếm
    public YarnTubeDirection direction = YarnTubeDirection.Up;
    public List<YarnTubeColorGroup> colorGroups = new();
}
```

> ⚠️ **Color convention — đọc kỹ:** `WoolBallData.ColorId` là **palette index**. Editor map color qua `ColorsParamSO.GetColorTypeByPaletteIndex(colorId)` / `GetColorByPaletteIndex(colorId)` (xem `WoolBallVisual.cs`, `BobbinsEditorUtility.cs`). Index palette **không bảo đảm bằng** `(int)WoolColorType` (palette có thể đảo/đổi). **Do đó YarnTubeColorGroup lưu `int colorId` (palette index)**, gán thẳng vào `WoolBallData.ColorId` khi spawn — KHÔNG cast `(int)WoolColorType`. Editor hiển thị swatch & dropdown qua `ColorsParamSO`/`_colorsParam` giống Bobbins. Runtime: `WoolBall` tự `(WoolColorType)Data.ColorId` như cũ.

### 2.3. Sửa `Data/LevelData.cs`

Thêm field (giữ default rỗng để JSON cũ deserialize an toàn — JsonUtility bỏ qua field thiếu):

```csharp
public List<YarnTubeData> yarnTubes = new List<YarnTubeData>();
```

### 2.4. Serialization touchpoints (editor) — xem chi tiết §5.4

Phải thêm `yarnTubes` vào **cả 3** chỗ trong editor và **1** chỗ runtime:
- `YarnBoardEditorWindow.cs` → class `YarnBoardLevelJson` (in-memory + load).
- `YarnBoardEditorWindow.Serialization.cs` → class `YarnBoardLevelSaveJson` (save) + `CreateSaveJson(...)`.
- `YarnBoardEditorWindow.Bobbins.cs` → `CreateRuntimeLevelData(...)` (preview dựng `LevelData`).
- `Data/LevelData.cs` (runtime, §2.3).

---

## 3. DOMAIN LAYER

### 3.1. Static obstacle registry (chặn ball đi qua tile của tube)

`BoardPathfinder.IsWalkableFor` hiện chỉ check `Level.IsActive(tile)` + reservation + occupancy (occupancy keyed theo `WoolBall`). Tube **không** phải WoolBall nên cần một kênh chặn riêng. **Không** dùng cách set `tileData[index]=false` (sẽ tạo lỗ board, đổi visual & spline).

**Giải pháp:** thêm tập tĩnh `staticObstacles` vào runtime.

1. `System/Runtime/BoardObstacleMap.cs` (mới) — wrap `HashSet<Vector2Int>`:
   ```csharp
   internal sealed class BoardObstacleMap
   {
       private readonly HashSet<Vector2Int> tiles = new();
       public void Add(Vector2Int t) => tiles.Add(t);
       public void Remove(Vector2Int t) => tiles.Remove(t);
       public bool IsObstacle(Vector2Int t) => tiles.Contains(t);
   }
   ```
2. `BoardPathfinder` nhận thêm `BoardObstacleMap` qua constructor; trong `IsWalkableFor` thêm: `if (obstacles.IsObstacle(tile)) return false;` (ngay sau check `Level.IsActive`).
3. `YarnBoardRuntimeState`:
   - khởi tạo `BoardObstacleMap obstacles`, truyền vào `pathfinder`.
   - expose: `public void AddStaticObstacle(Vector2Int t)`, `RemoveStaticObstacle(Vector2Int t)`, `public bool IsStaticObstacle(Vector2Int t)`.
   - trong constructor, sau khi có `context.Level`, **seed** obstacles từ `level.yarnTubes` (nếu muốn block sẵn trước cả khi tube spawn) — HOẶC để tube tự `AddStaticObstacle` lúc `OnCreated`. **Khuyến nghị: tube tự add lúc OnCreated** (đơn giản, 1 nguồn sự thật). Empty tube **không** remove → giữ block (quyết định #7).

> ⚠️ Kiểm tra `reservations.IsReservedByOther(ball, tile)` khi `ball == null` (xem §3.3, ta gọi walkable với ball=null). Nếu `TileReservationService` không handle null an toàn, thêm 1 helper `IsTileFreeForEject(Vector2Int)` riêng thay vì tái dùng `IsWalkableFor(null, ...)`.

### 3.2. `Elements/Domain/YarnTube.cs` (mới)

Theo khuôn `WoolBall`/`BottomBoard`: `MonoBehaviour, IRuntimeCreatable, IPendingCleanup`.

**Fields/Props:**
```csharp
public YarnTubeData Data { get; private set; }
public BoardSplineDataAdapterInfo Adapter { get; private set; }
public YarnTubeVisual Visual { get; private set; }
public bool IsPendingCleanup { get; private set; }
public int RemainingCount => queue.Count;          // tổng còn lại (counter)
public bool IsEmpty => queue.Count == 0;
public Vector2Int Tile => Data.tileId;
public Vector2Int Mouth => Data.tileId + Data.direction.ToVector();

private YarnBoardRuntimeState runtimeState;
private ConveyorEntrance conveyorEntrance;          // truyền xuống ball spawn để dispatch hoạt động
private readonly Queue<int> queue = new();          // ColorId theo FIFO
private Func<WoolBallData, UniTask<WoolBall>> spawnBall;  // delegate do LevelSpawner cấp
private Func<int> allocateBallId;                   // cấp BallId duy nhất
private bool ejectInFlight;
```

**`OnCreated(ICreateParameters)`** — cast sang `YarnTubeCreateParameters`:
- gán Data/Adapter/runtimeState/conveyorEntrance/spawnBall/allocateBallId.
- `transform.localPosition = Adapter.IndexToWorld(Data.tileId)`.
- build `queue` từ `Data.colorGroups` (FIFO, expand count).
- `runtimeState.AddStaticObstacle(Data.tileId)`.
- `Visual = EnsureVisualChild().GetOrAddComponent<YarnTubeVisual>(); Visual.Render(Data, Adapter); Visual.SetCounter(RemainingCount);`
- gọi `TryEjectLoop()` 1 lần để fill ban đầu.

**Re-evaluation:** `private void Update() { TryEject(); }` (poll rẻ, vài tube/level). *Tối ưu tùy chọn:* thay bằng event occupancy-changed nếu sau này thêm; hiện tại Update poll là đủ và đúng pattern event-driven của repo (không phải `IEntityDomain.Tick`).

**`TryEject()`** (cốt lõi):
```
if (ejectInFlight || IsEmpty) return;
if (!IsTileFreeForEject(Mouth)) return;        // miệng bị chặn
var landing = ComputeLandingTile();            // §3.3
if (landing == INVALID) return;                // mouth không inside/active
ejectInFlight = true;
int colorId = queue.Peek();                    // chưa Dequeue vội
EjectAsync(colorId, landing).Forget();
```

**`EjectAsync(colorId, landing)`** (UniTaskVoid):
- `queue.Dequeue();`
- tạo `var data = new WoolBallData { BallId = allocateBallId(), ColorId = colorId, tileId = landing, childrenTileIds = new() };`
- `var ball = await spawnBall(data);` → ball được `runtimeState.Register` (occupancy tại landing).
- (tùy chọn animation slide) `Visual.PlayPush(); await ball.PlaySlideInFromMouth(Mouth, landing, Adapter);` — nếu chưa làm animation thì ball xuất hiện thẳng tại landing + 1 pop scale.
- `Visual.SetCounter(RemainingCount);`
- `runtimeState.BuildTrainConnections();` (rebuild train links sau khi có ball mới).
- nếu `IsEmpty` → `Visual.SetEmptyState();`
- `ejectInFlight = false;`
- gọi lại `TryEject()` (chuỗi đẩy liên tiếp khi line còn trống).

**`IPendingCleanup.CleanupForLevelUnload()`**: `Visual?.StopAllTweens();` — KHÔNG remove obstacle (level đang unload, runtimeState bị bỏ). Set `IsPendingCleanup = true`.

> **Tránh dump tức thì gây giật:** đặt cooldown nhỏ giữa các eject (vd `await UniTask.Delay(ejectIntervalMs)` ~60–120ms) để line lấp đầy mượt thay vì 1 frame. Giá trị để serialize-config hoặc hằng số — đánh dấu **[tunable]**.

### 3.3. Thuật toán landing tile (slide-to-farthest)

```csharp
private const Vector2Int INVALID = ... // dùng sentinel hoặc bool TryComputeLandingTile(out)
private Vector2Int ComputeLandingTile()
{
    var dir = Data.direction.ToVector();
    var cursor = Mouth;
    if (!IsTileFreeForEject(cursor)) return INVALID;   // miệng phải trống
    var landing = cursor;
    while (true)
    {
        var next = cursor + dir;
        if (!IsTileFreeForEject(next)) break;          // gặp obstacle/ball/mép → dừng
        cursor = next;
        landing = cursor;
    }
    return landing;
}
```

`IsTileFreeForEject(t)` = `Level.IsInside(t) && Level.IsActive(t) && !runtimeState.IsStaticObstacle(t) && !runtimeState.TryGetOccupant(t, out _) && !reservedByAnyone(t)`. (Gói gọn trong 1 API trên `YarnBoardRuntimeState` để không lộ internals.)

### 3.4. BallId allocation

Authored balls dùng BallId từ JSON. Ball do tube spawn cần **BallId duy nhất, không đụng**. LevelSpawner cấp `allocateBallId`:
```csharp
int nextBallId = 1 + (level.yarnBalls?.Count > 0 ? level.yarnBalls.Max(b => b.BallId) : 0);
Func<int> allocate = () => nextBallId++;
```
Truyền cùng delegate cho mọi tube (shared counter) để các tube không cấp trùng.

### 3.5. `System/Creation/YarnTubeCreateParameters.cs` (thêm vào `YarnBoardCreationParameters.cs`)

```csharp
public sealed class YarnTubeCreateParameters : ICreateParameters
{
    public YarnTubeCreateParameters(
        YarnTubeData data,
        BoardSplineDataAdapterInfo adapter,
        Transform parent,
        YarnBoardRuntimeState runtimeState,
        ConveyorEntrance conveyorEntrance,
        Func<WoolBallData, UniTask<WoolBall>> spawnBall,
        Func<int> allocateBallId) { ... gán hết ... }

    public YarnTubeData Data { get; }
    public BoardSplineDataAdapterInfo Adapter { get; }
    public Transform Parent { get; }
    public YarnBoardRuntimeState RuntimeState { get; }
    public ConveyorEntrance ConveyorEntrance { get; }
    public Func<WoolBallData, UniTask<WoolBall>> SpawnBall { get; }
    public Func<int> AllocateBallId { get; }
}
```

### 3.6. `System/Creation/YarnTubeFactory.cs` (mới) — khuôn `WoolBallFactory`

```csharp
public class YarnTubeFactory : IFactory<YarnTube>
{
    public UniTask<YarnTube> Create(ICreateParameters parameters, CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        if (parameters is not YarnTubeCreateParameters p)
            throw new ArgumentException($"Expected {nameof(YarnTubeCreateParameters)}.");

        GameObject instance = PrefabProfile.YarnTubePrefab != null
            ? Object.Instantiate(PrefabProfile.YarnTubePrefab, p.Parent, false)
            : new GameObject("YarnTube");
        if (PrefabProfile.YarnTubePrefab == null) instance.transform.SetParent(p.Parent, false);
        instance.name = string.IsNullOrEmpty(p.Data.id) ? $"YarnTube_{p.Data.tileId}" : p.Data.id;

        if (!instance.TryGetComponent(out YarnTube tube)) tube = instance.AddComponent<YarnTube>();
        tube.OnCreated(p);
        return UniTask.FromResult(tube);
    }
}
```
Thêm `public static GameObject YarnTubePrefab` vào `Data/PrefabProfile.cs` theo pattern `WoolBallPrefab` (agent kiểm tra cách `PrefabProfile` load — Resources hay serialized field).

### 3.7. Wiring trong `System/Creation/LevelSpawner.cs`

Trong `SpawnLevelInternalAsync`, **sau** vòng tạo `yarnBalls` và **sau** `runtimeState.BuildTrainConnections()` (để ball authored đã có occupancy/link trước):

```csharp
private readonly YarnTubeFactory yarnTubeFactory = new();
...
// sau khi spawn balls + BuildTrainConnections():
await SpawnYarnTubes(level, adapter, boardRoot, token);
```

`SpawnYarnTubes`:
- nếu `level.yarnTubes == null || count == 0` → return.
- tính `allocateBallId` (§3.4).
- delegate spawn: `Func<WoolBallData, UniTask<WoolBall>> spawn = d => woolBallFactory.Create(new WoolBallCreateParameters(d, adapter, boardRoot, runtimeState, activeConveyorEntrance), token).AsUniTask();` *(kiểm tra signature trả về — `Create` đã trả `UniTask<WoolBall>`)*.
- loop tạo từng tube qua `yarnTubeFactory.Create(new YarnTubeCreateParameters(tubeData, adapter, boardRoot, runtimeState, activeConveyorEntrance, spawn, allocateBallId), token)`.

> Tube đặt dưới `boardRoot` (board-space) để IndexToWorld/scale khớp board, giống WoolBall.

---

## 4. VISUAL LAYER — `Elements/Visual/YarnTubeVisual.cs` (mới)

Theo khuôn `WoolBallVisual`/`BottomBoardVisual` (`MonoBehaviour`, method `Render(YarnTubeData, Adapter)`).

Trách nhiệm:
1. **Mesh ống** đặt tại tile, xoay theo `direction.ToYaw()` (**[CHECK]** khớp prefab & trục board). Dùng `PrefabProfile.YarnTubePrefab` nếu có; nếu không, placeholder primitive để agent ráp art sau.
2. **Head counter:** TextMeshPro (hoặc world-space UI) ở đầu/miệng ống hiển thị `RemainingCount`. API: `SetCounter(int total)`.
3. **Next-color hint (tùy chọn, khuyến nghị):** tint miệng ống theo màu nhóm sắp ra (peek queue) — dùng bảng màu `WoolColorType`→Color hiện có (xem `ColorsParamSO`/`ColorExtensions` trong Helpers). API: `SetNextColor(WoolColorType c)`.
4. **Animation:** `PlayPush()` (ống rung/nhả), `SetEmptyState()` (mờ ống + counter "0"), `StopAllTweens()`.
5. *(Tùy chọn)* hỗ trợ ball slide-in: tube không sở hữu ball, nên animation slide nằm ở WoolBall/ một tween world-space đơn giản; nếu skip, ball pop-scale tại landing là chấp nhận được cho bản đầu.

> Tách bạch domain↔visual: `YarnTube` (domain) KHÔNG truy cập Transform mesh trực tiếp; mọi thứ qua `YarnTubeVisual` API. Đúng pattern `WoolBall.Visual`.

---

## 5. EDITOR — Tool mới trong tab YarnBoard

File: `Editor/C#/YarnBoardEditorWindow.*.cs`. Tạo partial mới **`YarnBoardEditorWindow.YarnTube.cs`** cho logic riêng, đụng tối thiểu vào file chung.

### 5.1. Tool enum + nút
- `YarnBoardEditorWindow.cs`: thêm `YarnTube` vào `enum EditorToolMode { Select, Paint, Erase, YarnBall, YarnTube }` + field `_yarnTubeTool` (Button).
- **Wiring tool nằm ở `Workspace.cs`** (KHÔNG phải Inspector.cs): copy theo `_yarnTool`:
  - `_yarnTubeTool.clicked += () => SetTool(EditorToolMode.YarnTube);` (cạnh dòng 15).
  - active-class: `_yarnTubeTool.EnableInClassList("selected", _currentTool == EditorToolMode.YarnTube);` (cạnh dòng 124).
- UXML (`Editor/UXML/YarnBoardEditorWindow.uxml`) thêm `Button name="yarn-tube-tool"` trong board tool group; query trong `cs`; style active theo USS giống các tool khác.

### 5.2. Placement (board cell click) — `Workspace.cs`
Handler click cell xử lý theo `_currentTool` ở **`Workspace.cs`** (xem nhánh `_currentTool == EditorToolMode.YarnBall` ~dòng 277 & 300). Thêm nhánh `EditorToolMode.YarnTube`:
- khi tool = `YarnTube` và click ô `cell`:
  - nếu đã có tube tại `cell` → select nó (set `_selectedYarnTube`).
  - nếu chưa → tạo `YarnTubeData { tileId = cell, direction = Up, colorGroups = { {BrightRed, 2} } }`, add vào `_currentLevel.yarnTubes`, select, `MarkDirty()`.
  - **chặn** đặt tube lên ô: inactive (`tileData=false`), ô đã có yarn ball, ô đã có tube khác, hoặc ô target exit. (validate ngay + báo.)
- State editor mới: `private int _selectedYarnTube = -1;` reset ở `LoadLevelFromFullPath` + khi đổi tool/level (giống `_selectedYarnBall`).

### 5.3. Inspector (`YarnBoardEditorWindow.YarnTube.cs`, gọi từ `Inspector.cs`)
Khi có tube được chọn, panel inspector hiển thị:
- **Direction:** 4 nút toggle Up/Down/Left/Right (hoặc EnumField) → set `data.direction`, `MarkDirty()`, refresh scene preview (vẽ lại mũi tên).
- **Tile:** read-only (đổi bằng cách xóa & đặt lại, hoặc cho sửa Vector2Int — tùy, mặc định read-only).
- **Color groups (list, có thứ tự = FIFO):**
  - mỗi dòng: dropdown `WoolColorType` (dùng `_colorsParam`/`ColorsParamSO` cho swatch giống Bobbins box) + IntegerField `count`.
  - nút ▲▲ / ▼ đổi thứ tự nhóm (vì thứ tự = thứ tự đẩy).
  - nút **Add group**, **Remove group**, **Duplicate group** (khuôn `AddBobbinsBoxConfig`/`DuplicateSelectedBobbinsBox`).
- **Tổng count** hiển thị (read-only) = sum counts.
- Nút **Delete tube**.

### 5.4. Serialization (BẮT BUỘC cả 4 chỗ)
1. `YarnBoardEditorWindow.cs` → `class YarnBoardLevelJson`: thêm `public List<YarnTubeData> yarnTubes = new List<YarnTubeData>();`
2. `Serialization.cs` → `class YarnBoardLevelSaveJson`: thêm `public List<YarnTubeData> yarnTubes;`
3. `Serialization.cs` → `CreateSaveJson(...)`: thêm `yarnTubes = level.yarnTubes ?? new List<YarnTubeData>(),`
4. `Bobbins.cs` → `CreateRuntimeLevelData(...)`: thêm `yarnTubes = source.yarnTubes,` (để preview/PlayLevel dựng đúng).
5. `CreateDefaultLevel(...)` (Serialization.cs): khởi tạo `yarnTubes = new List<YarnTubeData>()`.

> JSON cũ không có field → JsonUtility để list rỗng/null. Đảm bảo mọi nơi đọc đều null-guard.

### 5.5. Validation (`Validation.cs` → `ValidateCurrentLevel`)
Thêm rule cho mỗi tube:
- tile phải `IsInside` + active.
- không trùng tile với yarn ball / tube khác / target exit.
- mỗi color group: `count >= 2` **và** `count % 2 == 0` → nếu sai, thêm `_validationErrors` (block save, giống các lỗi khác).
- phải có ≥ 1 color group với tổng count ≥ 2.
- **Warning (không block):** nếu ô `mouth = tile+dir` ngay lập tức inactive/ngoài board → tube không bao giờ eject được (thêm vào `_conveyorWarnings` hoặc cơ chế warning hiện có).

### 5.6. Scene/Board preview
- Vẽ ô tube trên grid editor: ô tô màu đặc trưng + **mũi tên hướng** `dir` + số tổng count (tái dùng cách vẽ cell `_cellViews` / `ScenePreview.cs`).
- Nếu có 3D scene preview (`ApplyFullLevelScenePreview`) — tùy chọn dựng `YarnTubeVisual` preview giống cách Bobbins/board preview; có thể để phase sau, ưu tiên grid 2D arrow trước.

---

## 6. Edge cases (agent phải xử lý)

1. **Mouth ngoài board / inactive ngay từ đầu:** tube không eject (đứng yên). Editor cảnh báo (5.5).
2. **Toàn bộ line đầy + queue còn ball:** tube chờ; khi player clear ball ở mouth → eject tiếp. Không drop ball.
3. **2 tube đẩy vào cùng 1 ô:** vì occupancy set ngay khi ball tạo + `ejectInFlight` guard + dùng `IsTileFreeForEject` (gồm reservation), ô không bị double-claim. Nếu vẫn lo race async, **reserve landing tile** trước khi await spawn rồi release sau khi register.
4. **Level unload giữa lúc eject:** `CancellationToken` từ LevelSpawner + `CleanupForLevelUnload` → ngừng tween, không tạo ball mồ côi. `EjectAsync` phải `try/catch OperationCanceledException`.
5. **Ball mới landed cạnh ball cùng màu authored:** sau eject gọi `BuildTrainConnections()` để link/merge đúng — không được bỏ.
6. **Empty tube:** vẫn obstacle (quyết định #7) — KHÔNG `RemoveStaticObstacle`. Pathfinder vẫn coi là tường.
7. **Counter:** luôn = `queue.Count` (tổng). Cập nhật ngay sau mỗi `Dequeue`.
8. **Color group count lẻ lọt qua editor (JSON sửa tay):** runtime không cần enforce chẵn (chỉ editor validate); nhưng đừng crash — cứ expand theo count.

---

## 7. Điểm cần implementer chốt (đã có default — agent xác nhận khi làm)

| Q | Default đề xuất | Ghi chú |
|---|----------------|--------|
| Eject có animation slide-in hay pop tại chỗ? | **Pop-scale tại landing** cho bản đầu, slide-in là enhancement | Đẹp hơn nhưng tốn công; tách phase. |
| `ejectInterval` giữa các ball | **~80ms** | tunable; tránh dump 1 frame. |
| Tile của tube có cho sửa trong inspector? | **Read-only** (đặt lại bằng tool) | giảm phức tạp. |
| Next-color tint ở miệng ống | **Có** | UX rõ ràng; nếu thiếu màu lookup thì skip. |
| 3D scene preview cho tube trong editor | **Phase sau**, ưu tiên grid arrow 2D | |
| Prefab art ống | placeholder primitive nếu chưa có | nối `PrefabProfile.YarnTubePrefab`. |

> Khi gặp các điểm này, agent dùng AskUserQuestion (select option / other) trước khi code phần đó.

---

## 8. Acceptance criteria / Test plan

**Gameplay**
- [ ] Tube đặt ở mép, `dir` hướng vào board: balls auto đẩy, lấp đầy line từ xa về gần, dừng khi mouth bị chiếm.
- [ ] Clear ball sát miệng → tube đẩy ball kế tiếp; landing tile tính lại đúng (xa nhất).
- [ ] Color FIFO đúng thứ tự nhóm (hết Blue mới tới Red).
- [ ] Ball đẩy ra là WoolBall thường: click chọn, train/merge/dispatch hoạt động.
- [ ] Counter = tổng còn lại, giảm đúng; về 0 thì tube thành obstacle rỗng, vẫn chặn đường.
- [ ] Ball/pathfinder không bao giờ đi xuyên tile tube.
- [ ] Nhiều tube cùng level chạy độc lập, BallId không trùng.

**Editor**
- [ ] Tool YarnTube đặt/chọn/xóa tube; direction 4 phía; color groups add/remove/reorder.
- [ ] Validate chặn count lẻ hoặc < 2; chặn đặt chồng ball/tube/exit/ô inactive.
- [ ] Save → load round-trip giữ nguyên `yarnTubes`. JSON cũ (không field) vẫn load ok.
- [ ] Play Level từ editor preview dựng đúng tube.

**Verify cuối:** mở 1 level test có 2 tube (1 ngang, 1 dọc, đa màu), chạy play, quay video/clip kiểm tra 8 tiêu chí gameplay; chạy round-trip save/load + diff JSON.

---

## 9. File-by-file checklist

**Mới**
- `Data/YarnTubeDirection.cs` (+extensions) *(hoặc nhét vào GlobalEnum.cs)*
- `Data/YarnTubeData.cs`
- `System/Runtime/BoardObstacleMap.cs`
- `Elements/Domain/YarnTube.cs`
- `Elements/Visual/YarnTubeVisual.cs`
- `System/Creation/YarnTubeFactory.cs`
- `Editor/C#/YarnBoardEditorWindow.YarnTube.cs`

**Sửa**
- `Data/LevelData.cs` → field `yarnTubes`
- `Data/PrefabProfile.cs` → `YarnTubePrefab`
- `System/Creation/YarnBoardCreationParameters.cs` → `YarnTubeCreateParameters`
- `System/Creation/LevelSpawner.cs` → `yarnTubeFactory` + `SpawnYarnTubes`
- `System/Runtime/BoardPathfinder.cs` → check obstacle trong `IsWalkableFor`
- `System/Runtime/YarnBoardRuntimeState.cs` → obstacle map + API `AddStaticObstacle/RemoveStaticObstacle/IsStaticObstacle` + `IsTileFreeForEject`
- `Editor/C#/YarnBoardEditorWindow.cs` → enum tool, field `_yarnTubeTool`, query nút, `YarnBoardLevelJson.yarnTubes`, `_selectedYarnTube`
- `Editor/C#/YarnBoardEditorWindow.Workspace.cs` → wiring tool (clicked + EnableInClassList) + **cell-click placement** nhánh `YarnTube`
- `Editor/C#/YarnBoardEditorWindow.Serialization.cs` → save JSON + `CreateSaveJson` + `CreateDefaultLevel`
- `Editor/C#/YarnBoardEditorWindow.Bobbins.cs` → `CreateRuntimeLevelData`
- `Editor/C#/YarnBoardEditorWindow.Inspector.cs` → gọi panel inspector tube (render khi `_selectedYarnTube >= 0`)
- `Editor/C#/YarnBoardEditorWindow.Validation.cs` → rule tube
- `Editor/C#/YarnBoardEditorWindow.ScenePreview.cs` → vẽ arrow + count
- `Editor/UXML/YarnBoardEditorWindow.uxml` + `Editor/USS/YarnBoardEditorWindow.uss` → nút tool

**Thứ tự build đề xuất:** Data → RuntimeState/Pathfinder obstacle → Domain+Factory → Spawner wiring → chạy test bằng JSON sửa tay → Visual → Editor tool/inspector/serialization/validation → preview.
