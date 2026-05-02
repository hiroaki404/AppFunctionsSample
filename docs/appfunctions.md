# AppFunctions 実装計画

## 目的

DroidKaigi 2026 セッション「複数アプリを横断して動くエージェンティックワークフローの構築」のデモ検証として、AppFunctions の基本的な動作を確認する。

## デモシナリオ（案）

**タスク管理アプリ**として AppFunctions を公開する。
Gemini（または ADB）から自然言語でタスクを操作できることを確認する。

例:
- 「明日の MTG をタスクに追加して」→ `addTask()` が呼ばれる
- 「今日のタスク一覧を見せて」→ `getTasks()` が呼ばれる
- 「タスクを完了にして」→ `completeTask()` が呼ばれる

## 公開する AppFunctions

| 関数名 | 説明 | 引数 | 戻り値 |
|---|---|---|---|
| `addTask` | タスクを追加する | `title: String`, `dueDate: String?` | `TaskResult` |
| `getTasks` | タスク一覧を取得する | なし | `List<TaskResult>` |
| `completeTask` | タスクを完了状態にする | `taskId: String` | `TaskResult` |

## 実装手順

### 1. 依存関係の追加

`libs.versions.toml` と `app/build.gradle.kts` に AppFunctions ライブラリと KSP を追加する。

```toml
# libs.versions.toml
appfunctions = "1.0.0-alpha08"
ksp = "2.3.20-2.0.1"  # kotlin バージョンに合わせる

[libraries]
appfunctions-service = { module = "androidx.appfunctions:appfunctions-service", version.ref = "appfunctions" }
appfunctions = { module = "androidx.appfunctions:appfunctions", version.ref = "appfunctions" }
appfunctions-compiler = { module = "androidx.appfunctions:appfunctions-compiler", version.ref = "appfunctions" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.ksp)
}

dependencies {
    implementation(libs.appfunctions)
    implementation(libs.appfunctions.service)
    ksp(libs.appfunctions.compiler)
}
```

### 2. AppFunctionService の実装

`@AppFunctionService` アノテーションを付けたクラスを作成し、各関数を `@AppFunction` で定義する。

```kotlin
@AppFunctionService
class TaskAppFunctionService(context: Context) : AppFunctionService(context) {

    @AppFunction
    suspend fun addTask(
        context: Context,
        /** 追加するタスクのタイトル */
        title: String,
        /** 期日（例: "2025-05-10"）*/
        dueDate: String? = null
    ): AddTaskResponse { ... }

    @AppFunction
    suspend fun getTasks(context: Context): GetTasksResponse { ... }

    @AppFunction
    suspend fun completeTask(
        context: Context,
        /** 完了にするタスクのID */
        taskId: String
    ): CompleteTaskResponse { ... }
}
```

### 3. AndroidManifest への登録

```xml
<service
    android:name=".TaskAppFunctionService"
    android:exported="true"
    android:permission="android.permission.EXECUTE_APP_FUNCTIONS">
    <intent-filter>
        <action android:name="androidx.appfunctions.AppFunctionService" />
    </intent-filter>
</service>
```

### 4. タスクデータの管理

初期実装ではインメモリの `MutableList` で管理する（Room は後回し）。

### 5. ADB での動作確認

```bash
# AppFunctions の一覧を確認
adb shell cmd app_function list-app-functions com.example.appfunctionssample

# addTask を実行
adb shell cmd app_function execute-app-function \
  com.example.appfunctionssample \
  addTask \
  --string title="DroidKaigi の発表準備"

# getTasks を実行
adb shell cmd app_function execute-app-function \
  com.example.appfunctionssample \
  getTasks
```

## 確認したいこと

- [ ] KDoc コメントがスキーマとして正しく反映されるか
- [ ] ADB から関数を呼び出して結果が返るか
- [ ] optional 引数（`dueDate`）の扱いはどうなるか
- [ ] 関数の粒度（細かい vs まとめた）でエージェントの挙動が変わるか

## 参考

- [AppFunctions 公式ドキュメント](https://developer.android.com/ai/appfunctions?hl=ja)
- [App Functions によるアプリの将来（Shreyas Patil ブログ）](https://blog.shreyaspatil.dev/the-future-of-android-apps-with-appfunctions)
- [FilipFan/AppFunctionsPilot: A sample that showcases Android App Functions use cases\.](https://github.com/FilipFan/AppFunctionsPilot)
- `docs/context.md` — 調査メモ
