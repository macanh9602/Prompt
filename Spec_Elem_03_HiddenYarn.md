# Spec_Elem_03 — Hidden Yarn (Len Ẩn Màu)

> **Mục tiêu:** Spec triển khai cho agent. Tuân thủ kiến trúc hiện có: **Data → Factory → Domain → Visual**, **event-driven** (KHÔNG dùng `IEntityDomain.Tick` — interface tồn tại nhưng chưa wire ở đâu).
>
> **Level unlock:** 15 · **Element id:** `Spec_Elem_03`
>
> **Tham chiếu gameplay:** https://www.youtube.com/watch?v=Iw0zHT67pek

---

## 0. Quyết định đã chốt (KHÔNG được tự đổi)

| # | Hạng mục | Quyết định |
|---|----------|-----------|
| 1 | **Bản chất** | Hidden Yarn KHÔNG phải loại ball mới. Nó là **modifier** gắn lên một `WoolBall` thường: ban đầu **che màu** (xám + dấu `?`), khi đủ điều kiện thì **lộ màu thật** và trở lại WoolBall bình thường hoàn toàn. |
| 2 | **Kiến trúc (theo yêu cầu)** | **`HiddenYarnFactory` riêng**. Domain `HiddenYarn` được **AddComponent vào CÙNG GameObject** đang chứa domain `WoolBall` (companion component, KHÔNG tạo GO mới). |
| 3 | **Visual split** | Trong GameObject ball có **2 visual tách biệt**: `WoolBallVisual` (default, màu thật) và `HiddenYarnVisual` (xám + `?`). Domain `HiddenYarn` **bật/tắt** để switch giữa 2 trạng thái — KHÔNG destroy/recreate. |
| 4 | **Điều kiện áp dụng (authoring)** | Chỉ gắn được cho ball **bị chặn (blocked) ngay từ đầu** — lúc load level KHÔNG có path từ ball tới Gate. Ball thông sẵn từ đầu → editor **chặn** gắn element. |
| 5 | **Gate = 1 exit duy nhất** ✅ | **Mỗi level chỉ có DUY NHẤT 1 exit tile** (`Level.targetExitTileId`, `Level.hasTargetExitTileId`). "Thông đường tới Gate" = tồn tại path từ ball tới đúng exit đó. **Dùng thẳng `WoolBall.CanMoveToTarget()`** (đã có sẵn, check `hasTargetExitTileId && CanMoveTo(targetExitTileId)`). KHÔNG loop `GetExitTiles()`. |
| 6 | **Điều kiện lộ màu (runtime)** | Khi path ball→Gate **thông** → lộ màu thật. Reveal là **một chiều** (đã lộ thì không ẩn lại dù sau đó chặn lại). |
| 7 | **Re-evaluate = EVENT** ✅ | KHÔNG Update poll. `HiddenYarn` **đăng ký callback** `YarnBoardRuntimeState.OnBoardChanged` (raise sau mỗi `BuildTrainConnections()` + sau `Unregister`). Mỗi lần board đổi → `TryReveal()`. Xem §3.4. |
| 8 | **Khoá tương tác khi ẩn** ✅ | Trong lúc còn ẩn, ball **KHÔNG cho click/chọn/train/merge/dispatch**. Occupancy & pathfinding (vật cản) **vẫn giữ nguyên** (ball vẫn chặn đường như bình thường). Khoá qua `WoolBall.SetInteractionLock(true)` + loại khỏi train-link building. Reveal → mở khoá. Xem §3.5. |
| 9 | **Sau khi lộ** | Ball thành WoolBall thường 100%: click/train/merge/dispatch như cũ. `HiddenYarn` **tự gỡ vai trò** (`enabled=false` + unsubscribe event), `HiddenYarnVisual` tắt hẳn. |
| 10 | **Asset visual ẩn** ✅ | Thêm `hiddenYarnMaterial` (+ asset dấu `?`) vào `PrefabProfile` để **art kiểm soát**. KHÔNG clone material runtime. Xem §4.1. |
| 11 | **Editor UI** | **Toggle trong inspector của Yarn Ball** đang chọn (KHÔNG tạo tool/tab riêng). Tap chọn ball → bật/tắt "Hidden Yarn". |

> Các quyết định ✅ là 4 điểm vừa chốt qua Q&A; phần còn lại đã chốt từ trước. Mọi điểm tunable nhỏ còn lại nằm ở §7.

---

## 1. Tổng quan hành vi runtime

```
   [Gate/Exit duy nhất] ──X── (tường/ball chặn) ────   ● = HiddenYarn (xám, "?", KHOÁ tương tác)
                              │
   ●(?)  ●(?)  ●(?)  ←── các ball bị chặn, màu bị giấu, không click được
                              │
   player clear bớt vật cản (ở các ball KHÁC) → mở đường tới Gate
                              ▼
   ●(Red) ●(Blue) ●(Green)  ←── lộ màu thật + mở khoá ngay khi path thông
```

1. **Level load:** với mỗi `WoolBallData` có cờ `isHiddenYarn = true`, sau khi `WoolBallFactory` tạo ball xong, `HiddenYarnFactory` **gắn `HiddenYarn` lên cùng GameObject** và **bật chế độ ẩn**: hiện `HiddenYarnVisual` (xám + `?`), ẩn pieces màu thật của `WoolBallVisual`, **khoá tương tác** ball.
2. Ball ẩn **vẫn register occupancy & là vật cản** trong pathfinding (KHÔNG đổi luật đi). `ColorId` thật vẫn nằm trong `WoolBallData`, chỉ bị **giấu khỏi hiển thị** và **khoá thao tác**.
3. **Mỗi khi board đổi** (player clear/move/dispatch ball khác → `BuildTrainConnections()` hoặc `Unregister` chạy → `OnBoardChanged` raise), `HiddenYarn` re-evaluate: ball này hiện có path tới Gate chưa (`woolBall.CanMoveToTarget()`)?
   - **Chưa thông** → giữ ẩn + giữ khoá.
   - **Đã thông** → **PlayReveal()**: tắt `HiddenYarnVisual`, hiện pieces màu thật, **mở khoá tương tác**, hiệu ứng lộ màu. Sau đó `HiddenYarn` ngừng vai trò (disable + unsubscribe).
4. Sau reveal, ball là WoolBall thường hoàn toàn.

**Invariant chính:** reveal **một chiều** & dựa **thuần** vào `CanMoveToTarget()` tại thời điểm board ổn định (sau `BuildTrainConnections`). Agent KHÔNG để trạng thái nhấp nháy ẩn↔hiện.

---

## 2. DATA LAYER

### 2.1. Sửa `Data/WoolBallData.cs`

Thêm cờ (default `false` để JSON cũ deserialize an toàn — JsonUtility bỏ field thiếu → `false`):

```csharp
[System.Serializable]
public class WoolBallData
{
    public int BallId;
    public int ColorId;
    public Vector2Int tileId;
    public List<Vector2Int> childrenTileIds;
    public bool isHiddenYarn;   // ← MỚI: ball khởi tạo ở trạng thái ẩn màu
}
```

> ⚠️ **Color convention:** `WoolBallData.ColorId` là **palette index** (xem `WoolBallVisual.cs`: `ColorsParamSO.GetColorByPaletteIndex(data.ColorId)`). Hidden Yarn **KHÔNG đổi** `ColorId` — chỉ giấu hiển thị. Khi reveal, `WoolBallVisual` dùng đúng `ColorId` đó.

### 2.2. Serialization — ✅ KHÔNG cần sửa thêm

Đã verify: `YarnBoardEditorWindow.Serialization.cs` gán **nguyên list** `yarnBalls = level.Balls` (dòng ~169) chứ KHÔNG copy field-by-field. JsonUtility serialize cả object `WoolBallData` ⇒ `isHiddenYarn` tự được lưu/đọc. **Không cần** thêm list mới vào `LevelData` (khác cố ý với YarnTube). Chỉ cần đảm bảo không có chỗ nào dựng `new WoolBallData{...}` thủ công làm rớt field — hiện không có.

---

## 3. DOMAIN LAYER

### 3.1. `Elements/Domain/HiddenYarn.cs` (mới) — companion component

Khuôn `WoolBall`/`YarnTube`: `MonoBehaviour, IRuntimeCreatable, IPendingCleanup`. **Sống chung GameObject với `WoolBall`.**

```csharp
[RequireComponent(typeof(WoolBall))]
public class HiddenYarn : MonoBehaviour, IRuntimeCreatable, IPendingCleanup
{
    public bool IsHidden { get; private set; }
    public bool IsRevealed => !IsHidden;
    public bool IsPendingCleanup { get; private set; }

    private WoolBall woolBall;                       // cùng GO
    private YarnBoardRuntimeState runtimeState;
    private BoardSplineDataAdapterInfo adapter;
    private HiddenYarnVisual hiddenVisual;           // visual xám + "?"
    private bool revealInFlight;
    private bool subscribed;
}
```

**`OnCreated(ICreateParameters)`** — cast sang `HiddenYarnCreateParameters`:
- `woolBall = GetComponent<WoolBall>();` (BẮT BUỘC tồn tại — gắn sau khi ball tạo xong).
- gán `runtimeState`, `adapter` từ params (hoặc `woolBall.Adapter`).
- Tạo/lấy `HiddenYarnVisual` ở **child riêng `"HiddenVisual"`** (KHÔNG dùng chung child `"Visual"` của WoolBall — xem §4.2).
- `EnterHiddenState()`:
  - `IsHidden = true;`
  - `woolBall.Visual.SetAllPiecesVisible(false);` (ẩn màu thật — API có sẵn, dòng ~427 `WoolBallVisual`).
  - `hiddenVisual.Render(woolBall.Data, adapter); hiddenVisual.SetActive(true);`
  - **`woolBall.SetInteractionLock(true);`** (khoá click/train/merge — xem §3.5).
- **Subscribe event:** `runtimeState.OnBoardChanged += OnBoardChanged; subscribed = true;` (xem §3.4).
- Re-evaluate lần đầu: `TryReveal();` (an toàn — nếu author đặt đúng thì vẫn ẩn vì blocked).

**KHÔNG có `Update()`.** Re-eval hoàn toàn event-driven (quyết định #7).

```csharp
private void OnBoardChanged() => TryReveal();
```

**`TryReveal()`** (cốt lõi):
```csharp
private void TryReveal()
{
    if (revealInFlight || IsRevealed || IsPendingCleanup) return;
    if (woolBall == null || runtimeState == null) return;
    if (!IsReachableToGate()) return;          // chưa thông → giữ ẩn
    revealInFlight = true;
    PlayRevealAsync(this.GetCancellationTokenOnDestroy()).Forget();
}
```

**`IsReachableToGate()`** — ✅ dùng API có sẵn (đã verify), KHÔNG tự viết BFS:
```csharp
private bool IsReachableToGate()
{
    // Level chỉ 1 exit → tái dùng nguyên CanMoveToTarget() của WoolBall.
    // (nội bộ: hasTargetExitTileId && CanMoveTo(targetExitTileId) → TryFindPath head/tail)
    return woolBall.CanMoveToTarget();
}
```
> `CanMoveToTarget()` xét cả endpoint root & tail giống lúc ball di chuyển ⇒ nhất quán với luật move. KHÔNG cần xử lý multi-tile riêng.

**`PlayRevealAsync(ct)`** (UniTaskVoid):
- `IsHidden = false;`
- `await hiddenVisual.PlayRevealOut(ct);` (fade dấu `?` + ball xám) rồi `hiddenVisual.SetActive(false);`
- `woolBall.Visual.SetAllPiecesVisible(true);` (hiện màu thật — pieces vẫn còn, chỉ bị ẩn; KHÔNG cần Render lại).
- **`woolBall.SetInteractionLock(false);`** (mở khoá → ball thành thường).
- (tùy chọn) `woolBall.Visual.PlayEntranceScale();` hoặc pop-scale nhấn "lộ màu".
- `runtimeState.BuildTrainConnections();` (rebuild links phòng reveal mở ra merge mới — idempotent).
- `revealInFlight = false;`
- **Gỡ vai trò:** `UnsubscribeBoardChanged(); enabled = false;` (disable, KHÔNG Destroy — an toàn nếu còn reference; tunable §7).
- Bọc `try/catch (OperationCanceledException)`.

**`IPendingCleanup.CleanupForLevelUnload()`**:
```csharp
IsPendingCleanup = true;
UnsubscribeBoardChanged();
hiddenVisual?.StopAllTweens();
```
`UnsubscribeBoardChanged()`: `if (subscribed && runtimeState != null) { runtimeState.OnBoardChanged -= OnBoardChanged; subscribed = false; }`. Gọi cả trong `OnDestroy`.

> **Tách bạch domain↔visual:** `HiddenYarn` KHÔNG truy cập mesh/material trực tiếp; mọi thứ qua `HiddenYarnVisual` API + API show/hide của `WoolBallVisual`.

### 3.2. `System/Creation/HiddenYarnCreateParameters.cs` (thêm vào `YarnBoardCreationParameters.cs`)

```csharp
public sealed class HiddenYarnCreateParameters : ICreateParameters
{
    public HiddenYarnCreateParameters(
        WoolBall woolBall,
        BoardSplineDataAdapterInfo adapter,
        YarnBoardRuntimeState runtimeState)
    {
        WoolBall = woolBall;
        Adapter = adapter;
        RuntimeState = runtimeState;
    }

    public WoolBall WoolBall { get; }
    public BoardSplineDataAdapterInfo Adapter { get; }
    public YarnBoardRuntimeState RuntimeState { get; }
}
```

### 3.3. `System/Creation/HiddenYarnFactory.cs` (mới) — ✅ AddComponent, KHÔNG Instantiate

```csharp
public class HiddenYarnFactory : IFactory<HiddenYarn>
{
    public UniTask<HiddenYarn> Create(ICreateParameters parameters, CancellationToken ct = default)
    {
        ct.ThrowIfCancellationRequested();
        if (parameters is not HiddenYarnCreateParameters p)
            throw new System.ArgumentException($"Expected {nameof(HiddenYarnCreateParameters)}.");

        var go = p.WoolBall.gameObject;                 // CÙNG GameObject với WoolBall
        if (!go.TryGetComponent(out HiddenYarn hidden))  // chống double-create
            hidden = go.AddComponent<HiddenYarn>();

        hidden.OnCreated(p);
        return UniTask.FromResult(hidden);
    }
}
```

### 3.4. ✅ Event hook trong `YarnBoardRuntimeState` (re-evaluate)

Thêm 1 event công khai + raise tại các điểm board ổn định:

```csharp
public event System.Action OnBoardChanged;
private void RaiseBoardChanged() => OnBoardChanged?.Invoke();
```

Raise **cuối** các API làm thay đổi cục diện board (BẮT BUỘC đủ **3** điểm):
- Cuối `BuildTrainConnections()` — tín hiệu chính khi graph được rebuild.
- Cuối `Unregister(WoolBall)` — khi 1 ball hoàn tất/biến mất (mở đường).
- **Cuối `EndTrainSession(session)`** ✅ — gọi `BuildTrainConnections()` ở đây. ĐÂY LÀ ĐIỂM DỄ SÓT NHẤT.

> ⚠️ **Tại sao bắt buộc `EndTrainSession`:** `BuildTrainConnections()` chỉ chạy lúc **bắt đầu** 1 nước đi (`WoolBall.OnInteract → RebuildTrainLinks`, tức TRƯỚC khi ball di chuyển), và `Unregister` chỉ chạy khi ball **Complete/biến mất**. Một nước đi mà ball **reposition hoặc đang chờ dispatch ở conveyor** (chưa Complete) sẽ giải phóng ô cũ qua `EndMoveOccupancy`/`SetOccupancy` mà **KHÔNG** raise `OnBoardChanged`. Hệ quả: ball ẩn bị chặn bởi ô đó **không bao giờ lộ** dù đường đã thông. `TrainMoveSession.Run()` luôn gọi `runtimeState.EndTrainSession(this)` trong `finally` (sau khi occupancy đã commit) ⇒ thêm `BuildTrainConnections()` tại `EndTrainSession` phủ trọn mọi nước đi. **KHÔNG** raise trong `SetOccupancy`/`Register` vì sẽ gây lộ nhầm giữa lúc spawn (occupancy chưa đầy đủ).
>
> ⚠️ **Tránh reentrancy:** `PlayRevealAsync` gọi `BuildTrainConnections()` → lại raise `OnBoardChanged`. Lúc đó `IsRevealed == true` nên `TryReveal()` return ngay ⇒ không vòng lặp. Vẫn nên set `IsHidden=false` **trước** khi gọi `BuildTrainConnections`.

### 3.5. ✅ Khoá tương tác — sửa nhẹ `WoolBall`

Khoá thuộc **domain WoolBall** (1 nguồn sự thật), `HiddenYarn` chỉ bật/tắt cờ.

Thêm vào `Elements/Domain/WoolBall.cs`:
```csharp
public bool IsInteractionLocked { get; private set; }

public void SetInteractionLock(bool locked)
{
    IsInteractionLocked = locked;
    SetInteractionCollidersEnabled(!locked && !IsCompleted && !IsMoving);  // method private sẵn có
}
```
Chèn guard:
- `OnInteract()`: thêm `IsInteractionLocked` vào đầu điều kiện return:
  `if (IsInteractionLocked || IsMoving || IsCompleted || IsPendingCleanup || isMovingIntoEntrance || isDispatchingAtWait) return;`
- `CanBeClickedHead`: `=> !IsInteractionLocked && !trainCycleUnsafe && !HasPreviousTailChain();`

**Loại khỏi train/merge — phải chặn ở CẢ HAI tầng (nếu thiếu 1 → ball ẩn bị teleport theo train):**

1. **Static train-link graph** — `TrainGraphBuilder.BuildTrainConnections()`: khi dựng tập `eligible` để nối link, skip ball `IsInteractionLocked` ⇒ ball ẩn không có train link nào.
2. **Session join gate** ✅ — `TrainSessionCoordinator.CanJoinTrainSession(ball)`: thêm `!ball.IsInteractionLocked`. ĐÂY LÀ ĐIỂM DỄ SÓT. Khi 1 ball cùng màu kề bên di chuyển, `TrainMoveSession.EnsureFullColorComponentCollected()` quét **toàn bộ** ball cùng màu kề footprint và **ép** chúng vào session (kể cả tạo synthetic SideMerge link), chỉ chặn bằng `CanJoinTrainSession`. Tầng (1) KHÔNG đủ vì path này bỏ qua static graph. Nếu thiếu, ball ẩn bị kéo vào train của ball kia và **teleport tới conveyor entrance** khi ball kia dispatch.

> Vì ball ẩn luôn blocked (không path tới gate) nên thường không tự thao tác được; nhưng khoá tường minh ở cả 2 tầng đảm bảo **đúng quyết định #8** khi ball ẩn đứng cạnh ball cùng màu đã thông và ball đó được click di chuyển/dispatch.

### 3.6. Wiring trong `System/Creation/LevelSpawner.cs`

Trong vòng tạo `yarnBalls` (đoạn ~dòng 120-129): **capture** `WoolBall` trả về, nếu `isHiddenYarn` thì gắn Hidden Yarn.

```csharp
private readonly HiddenYarnFactory hiddenYarnFactory = new();
...
WoolBall ball = await woolBallFactory.Create(new WoolBallCreateParameters(
    ballData, adapter, boardRoot, runtimeState, activeConveyorEntrance), token);

if (ballData.isHiddenYarn)
{
    await hiddenYarnFactory.Create(
        new HiddenYarnCreateParameters(ball, adapter, runtimeState), token);
}
```

> Hiện `await woolBallFactory.Create(...)` **không capture** return (dòng ~122). Đổi thành `WoolBall ball = await ...`.
>
> **Thứ tự:** gắn Hidden Yarn xong, `BuildTrainConnections()` cuối cùng (dòng ~132) sẽ raise `OnBoardChanged` ⇒ `TryReveal` lần đầu chạy đúng (và bị loại khỏi train vì đã khoá). Ball đã register occupancy trong `WoolBall.OnCreated`.

---

## 4. VISUAL LAYER

### 4.1. ✅ Asset qua `PrefabProfile` + `HiddenYarnVisual.cs` (mới)

**Sửa `Data/PrefabProfile.cs`** — thêm (theo pattern `WoolBallMaterial`/`WoolBallMesh`):
```csharp
public static Material HiddenYarnMaterial;   // material xám neutral cho ball ẩn
public static Sprite   QuestionMarkSprite;   // dấu "?" (hoặc dùng TMP nếu chưa có sprite)
```
> **[CHECK]** cách `PrefabProfile` nạp asset (Resources/serialized field/SO). Art cấp `HiddenYarnMaterial` (xám) + `QuestionMarkSprite`. Nếu sprite chưa có → tạm TMP `"?"`, để TODO nối sprite sau.

**`Elements/Visual/HiddenYarnVisual.cs`** — khuôn `WoolBallVisual` (`MonoBehaviour`, `Render(WoolBallData, Adapter)`). Đặt trên **child riêng `"HiddenVisual"`**.

Trách nhiệm:
1. **Mesh ball xám:** tái dùng `PrefabProfile.WoolBallMesh`, material = `PrefabProfile.HiddenYarnMaterial`. Render **mọi piece** theo footprint (`tileId` + `childrenTileIds`) — copy logic dựng piece + `TileToLocalPosition` của `WoolBallVisual` (không kế thừa, copy pattern).
2. **Dấu `?`:** overlay world-space (`QuestionMarkSprite` trên quad, hoặc TMP `"?"`) gắn trên đỉnh ball (head piece). Multi-tile: 1 dấu `?` ở head là đủ (tunable).
3. **API:** `Render(data, adapter)`, `SetActive(bool)`, `UniTask PlayRevealOut(CancellationToken)` (fade/scale dấu `?` + ball xám), `StopAllTweens()`.

### 4.2. Cấu trúc GameObject ball (sau khi gắn Hidden Yarn)

```
WoolBall (GameObject)
├─ [WoolBall]        (domain, sẵn có)
├─ [HiddenYarn]      (domain companion, MỚI — cùng GO)
├─ Visual            (child sẵn có) → [WoolBallVisual]  → Pieces (màu thật, bị ẩn khi hidden)
└─ HiddenVisual      (child MỚI)     → [HiddenYarnVisual] → Pieces xám + "?"
```

Switch trạng thái = bật/tắt 2 child + flag khoá (quyết định #3, #8). KHÔNG destroy để switch nhanh, giữ state.

### 4.3. Show/hide màu thật — tái dùng `WoolBallVisual.SetAllPiecesVisible`

- Vào hidden: `woolBall.Visual.SetAllPiecesVisible(false);`
- Khi reveal: `woolBall.Visual.SetAllPiecesVisible(true);`

> ⚠️ `SetAllPiecesVisible(false)` cũng được `WoolBall.Complete()` dùng để ẩn ball hoàn tất. Hidden Yarn chỉ thao tác khi ball **chưa** Complete (ẩn→reveal xảy ra trước mọi dispatch hoàn tất) ⇒ không xung đột. Nếu muốn tách ngữ nghĩa, thêm `SetColorVisualHidden(bool)` riêng — **[tunable §7]**.

---

## 5. EDITOR — Toggle trong Inspector của Yarn Ball

KHÔNG tạo tool/tab mới. Dùng tool `YarnBall` sẵn có để **tap chọn** ball (`_selectedYarnBall`), bật/tắt trong inspector.

### 5.1. Inspector (`Editor/C#/YarnBoardEditorWindow.Inspector.cs`)
Khi `_selectedYarnBall != null`, thêm vào panel inspector ball:
- **Toggle "Hidden Yarn"** ↔ `_selectedYarnBall.isHiddenYarn`.
- Khi tick **ON**: chạy validate (§5.2). Nếu ball **không** blocked-from-start → **từ chối bật**, cảnh báo inline ("Chỉ áp dụng cho ball bị chặn từ đầu"), revert toggle về OFF.
- Đổi giá trị hợp lệ → `MarkDirty()` + refresh preview (overlay `?` trên cell).

### 5.2. Validation (`Editor/C#/YarnBoardEditorWindow.Validation.cs` → `ValidateCurrentLevel`)
Rule cho mỗi ball có `isHiddenYarn == true`:
- **Phải blocked-from-start tới Gate duy nhất:** dựng board tĩnh ban đầu (active tiles + occupancy mọi ball/obstacle/tube authored) và kiểm tra **KHÔNG** có path từ ball tới `targetExitTileId`. Có path → `_validationErrors` (block save) message rõ ràng.
  - **[CHECK]** editor có pathfinder không. Ưu tiên: (a) BFS tay trên `tileData` + tập occupied tiles (gọn, không phụ thuộc runtimeState); fallback (b) reuse `BoardPathfinder` chế độ editor. Vì chỉ 1 exit nên BFS tay rất nhẹ.
- **Block nếu level KHÔNG có Gate** (`!hasTargetExitTileId`) mà vẫn có hidden ball → hidden ball không bao giờ lộ → `_validationErrors` (đổi từ warning thành **error**, vì single-exit là tiền đề bắt buộc).
- Ball `isHiddenYarn` vẫn phải hợp lệ như ball thường (rule sẵn có).

### 5.3. Scene/Board preview (`Editor/C#/YarnBoardEditorWindow.ScenePreview.cs` / `Preview.cs`)
- Cell có hidden ball: vẽ **overlay xám + ký hiệu `?`** đè lên swatch màu thật (author vẫn thấy màu nền: ví dụ viền màu thật + fill xám + `?`).
- 3D scene preview (`ApplyFullLevelScenePreview`): tùy chọn dựng `HiddenYarnVisual` preview — phase sau, ưu tiên 2D overlay trước.

---

## 6. Edge cases (agent phải xử lý)

1. **Reveal một chiều:** đã lộ → dù board đổi khiến lại bị chặn → KHÔNG ẩn lại. `IsRevealed` chốt cứng; component đã `enabled=false` + unsubscribe.
2. **Ball multi-tile (`childrenTileIds`):** hidden visual phủ xám toàn thân + `?` ở head; điều kiện reveal dùng `CanMoveToTarget()` (đã xét cả root/tail endpoint) ⇒ nhất quán, không cần code riêng.
3. **Level KHÔNG có Gate:** hidden ball ở mãi trạng thái ẩn — editor đã chặn (5.2) nên runtime hiếm gặp; nếu lọt (JSON tay) thì ball ẩn vĩnh viễn, KHÔNG crash.
4. **Khoá tương tác khi ẩn (quyết định #8):** ball ẩn KHÔNG được click/kéo train/merge; bị loại khỏi `RebuildTrainLinks`. Khi reveal mới mở khoá. Đảm bảo `SetInteractionLock(false)` chạy đúng 1 lần lúc reveal.
5. **Level unload giữa lúc reveal:** `PlayRevealAsync` bọc `try/catch OperationCanceledException`; `CleanupForLevelUnload`/`OnDestroy` unsubscribe event + dừng tween, không để visual nửa vời.
6. **Gắn nhầm cho ball thông sẵn (JSON tay vượt editor):** runtime KHÔNG crash — `OnBoardChanged` đầu tiên (sau `BuildTrainConnections` lúc spawn) thấy `CanMoveToTarget()==true` → lộ luôn. Editor mới là nơi chặn authoring.
7. **Double-create:** `HiddenYarnFactory` dùng `TryGetComponent` → không gắn 2 lần/GO.
8. **Reentrancy event:** `PlayRevealAsync` gọi `BuildTrainConnections()` → raise `OnBoardChanged` lại; vì `IsRevealed==true`, `TryReveal` return ngay (xem §3.4). Không vòng lặp.
9. **Hiệu năng:** event-driven, chỉ chạy `CanMoveToTarget()` khi board đổi (không mỗi frame). Số hidden ball ít ⇒ rẻ.

---

## 7. Điểm tunable nhỏ còn lại (đã có default — agent xác nhận khi chạm tới)

| Q | Default | Ghi chú |
|---|---------|--------|
| Reveal: disable hay Destroy `HiddenYarn`? | **Disable** (`enabled=false` + unsubscribe) | An toàn nếu còn reference; Destroy cũng được. |
| Reveal effect | **Pop-scale + fade `?`** | Tối thiểu là switch tức thì nếu gấp. |
| `SetColorVisualHidden(bool)` riêng cho WoolBallVisual? | **Không** — tái dùng `SetAllPiecesVisible` | Thêm method riêng nếu muốn tách ngữ nghĩa khỏi `Complete()`. |
| Dấu `?` cho multi-tile | **1 dấu ở head** | Có thể mỗi piece 1 dấu nếu art muốn. |
| Editor blocked-from-start: BFS tay hay reuse pathfinder | **BFS tay** trên tileData+occupancy (1 exit nên nhẹ) | Reuse `BoardPathfinder` nếu tách được. |
| Sprite `?` chưa có asset | **TMP `"?"` tạm** | Nối `PrefabProfile.QuestionMarkSprite` khi art cấp. |

> Các điểm này nhỏ; nếu phát sinh lựa chọn lớn ngoài bảng, agent dùng AskUserQuestion trước khi code.

---

## 8. Acceptance criteria / Test plan

**Gameplay**
- [ ] Ball gắn Hidden Yarn hiển thị **xám + `?`** lúc load, màu thật bị giấu.
- [ ] Ball ẩn **KHÔNG click/chọn/merge được**; nhưng vẫn chặn đường/occupancy như ball thường (pathfinder không đi xuyên).
- [ ] Khi player clear vật cản mở đường ball→Gate (1 exit) → ball **lộ màu thật + mở khoá ngay**, đúng `ColorId`.
- [ ] Reveal **một chiều**: chặn lại sau đó không ẩn lại.
- [ ] Sau lộ: click/train/merge/dispatch hoạt động y như WoolBall thường.
- [ ] Nhiều hidden ball/level độc lập; ball nào thông tới Gate trước lộ trước.
- [ ] Ball multi-tile: phủ xám + `?`; reveal đúng theo endpoint (`CanMoveToTarget`).
- [ ] Re-eval chạy theo **event** (đặt breakpoint/log ở `OnBoardChanged`), KHÔNG poll mỗi frame.

**Editor**
- [ ] Tap chọn ball → toggle Hidden Yarn bật/tắt; lưu vào `isHiddenYarn`.
- [ ] Bật toggle cho ball **thông sẵn** → bị từ chối + cảnh báo; chỉ ball **blocked-from-start** mới bật được.
- [ ] Validate chặn save khi: hidden ball không blocked-from-start, hoặc level thiếu Gate mà có hidden ball.
- [ ] Save → load round-trip giữ nguyên `isHiddenYarn`. JSON cũ (không field) load ra `false`, không lỗi.
- [ ] Play Level từ editor preview dựng đúng hidden ball.

**Verify cuối:** mở 1 level test có Gate (1 exit) + vài hidden ball (1 single-tile, 1 multi-tile) bị chặn; play, clear dần để mở đường, quay clip kiểm 8 tiêu chí gameplay; round-trip save/load + diff JSON xác nhận `isHiddenYarn`.

---

## 9. File-by-file checklist

**Mới**
- `Elements/Domain/HiddenYarn.cs` (companion component, cùng GO với WoolBall, event-driven)
- `Elements/Visual/HiddenYarnVisual.cs` (visual xám + `?`)
- `System/Creation/HiddenYarnFactory.cs` (AddComponent, KHÔNG Instantiate)

**Sửa**
- `Data/WoolBallData.cs` → field `bool isHiddenYarn`
- `Data/PrefabProfile.cs` → `HiddenYarnMaterial` + `QuestionMarkSprite`
- `System/Creation/YarnBoardCreationParameters.cs` → `HiddenYarnCreateParameters`
- `System/Creation/LevelSpawner.cs` → capture `WoolBall` return + gắn `HiddenYarnFactory` khi `isHiddenYarn`
- `System/Runtime/YarnBoardRuntimeState.cs` → **event `OnBoardChanged`** (raise cuối `BuildTrainConnections` + `Unregister`) + **skip ball `IsInteractionLocked`** trong build train-link
- `Elements/Domain/WoolBall.cs` → `IsInteractionLocked` + `SetInteractionLock(bool)` + guard `OnInteract`/`CanBeClickedHead`
- `Editor/C#/YarnBoardEditorWindow.Inspector.cs` → toggle Hidden Yarn cho ball đang chọn
- `Editor/C#/YarnBoardEditorWindow.Validation.cs` → rule blocked-from-start (tới 1 exit) + bắt buộc có Gate
- `Editor/C#/YarnBoardEditorWindow.ScenePreview.cs` / `Preview.cs` → overlay xám + `?`

**Không cần sửa** (đã verify): `Serialization.cs`/`Bobbins.cs` (gán nguyên list `yarnBalls = level.Balls`, JsonUtility tự lo `isHiddenYarn`).

**Thứ tự build đề xuất:** Data (`isHiddenYarn`) → WoolBall lock API + RuntimeState event → Domain `HiddenYarn` + Factory + CreateParameters → Spawner wiring → test bằng JSON tay (1 ball blocked `isHiddenYarn=true` + 1 Gate) → Visual xám + `?` (PrefabProfile asset) → Editor toggle/validation/preview.

---

## 10. Khác biệt cố ý so với Spec_Elem_02 (Yarn Tube)

| | Yarn Tube | Hidden Yarn |
|---|-----------|-------------|
| Loại entity | Entity độc lập, GO riêng | **Modifier gắn lên WoolBall, cùng GO** |
| Data | List `yarnTubes` trong `LevelData` | **Cờ `isHiddenYarn` trong `WoolBallData`** (không sửa serialization) |
| Factory | Instantiate prefab mới | **AddComponent vào GO ball có sẵn** |
| Visual | 1 visual riêng | **2 visual toggle** (default WoolBall + hidden xám/`?`) |
| Editor | Tool mới trong tab | **Toggle trong inspector ball** |
| Re-eval | Update poll (eject) | **Event `OnBoardChanged`** (không poll) |
| Trigger | Auto eject khi mouth trống | **Reveal khi path tới Gate (1 exit) thông — một chiều** |
| Tương tác | Ball đẩy ra là ball thường | **Khoá click/train/merge tới khi lộ** |
