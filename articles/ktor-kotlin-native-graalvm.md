---
title: "KotlinでNative Buildするときの2パターン —— Kotlin/Native・GraalVM"
emoji: "🫠"
type: "tech"
topics: [Kotlin, GraalVM]
published: true
---

Kotlin は KMP（Kotlin Multiplatform）によって複数のプラットフォームで同一コードを動かせます。多くの文脈では iOS/Android が語られがちですが、**Desktop/Server-side** も対象です。本稿では **[Ktor](https://github.com/ktorio/ktor)** を使って
- **Kotlin/Native** で単体実行可能バイナリ
- AOTコンパイルして**Native ImageをGraalVM**で実行

の2パターンの最小構成をまとめます. 

作成したリポジトリ 
@[card](https://github.com/hondaya14/kotlin-native-server)
@[card](https://github.com/hondaya14/graalvm-ktor)

## Ktor × Kotlin/Native

**Ktor（JetBrains 製の Web フレームワーク）を Kotlin/Native でビルドし、単体実行可能なバイナリを作る**最小構成です。

:::message 
**Kotlin は通常 JVM 向けにバイトコード（JAR）として実行**されますが、**Kotlin/Native を使うと [LLVM](https://llvm.org/) 経由で OS/CPU ネイティブの単体バイナリ**を生成できます。  
参考: <https://kotlinlang.org/docs/native-overview.html>
:::

### ビルド設定

KMPのpluginを入れる。
targetを指定することで、targetに応じたバイナリが生成される。
https://github.com/hondaya14/kotlin-native-server/blob/main/build.gradle.kts

### ビルドと実行

`kmp` プラグインで追加される `nativeBinaries` タスクでビルドします。

```sh
kotlin-native-server ➤ ./gradlew nativeBinaries

BUILD SUCCESSFUL in 27s
3 actionable tasks: 3 executed
```

生成されたバイナリを起動します。

```sh
kotlin-native-server ➤ ./build/bin/native/releaseExecutable/kotlin-native-server.kexe
[INFO] (io.ktor.server.Application): Application started in 0.001 seconds.
[INFO] (io.ktor.server.Application): Responding at http://0.0.0.0:8080
```

### 注意点（ライブラリ互換・cinterop）

:::message alert
Kotlin/Native では **JVM APIを使うライブラリは使えない**。  

そのため、**マルチプラットフォーム対応ライブラリ**、または **C ライブラリとの相互運用（cinterop）** に置き換える必要があります。  
> cinterop を使うとc ライブラリの使用が可能。ref: <https://kotlinlang.org/docs/native-c-interop.html>
:::

## Ktor × GraalVM

次に、Ktor を**GraalVM**上で実行させる最小構成です

:::message
**GraalVM**

従来のJVMは、事前コンパイルでバイトコード(.class/.jar)へ変換したあと、JVMでのJITコンパイルでバイナリコードへ変換され最適化が行われる。  
GraalVMでは、事前コンパイルでNative Image(バイナリコード)へ変換することができる。  
一般に、JVM下では、JITコンパイルが行われるまでパフォーマンスが出ないため、高トラフィックのバックエンドサーバーなどでは十分にJITコンパイルが行われるために、ウォームアップ(暖気運転)を行ったりする。が、GraalVMではAOTコンパイルでNativeまで変換するため、起動時点で高いパフォーマンスを提供でき、メモリのオーバーヘッドも少ない。

また、GraalVM自体はMulti-Language Runtimeであり、Javaなどの旧来のJVM以外にもJS, Python, Ruby, ...など多言語で利用可能。  
Ref: https://www.graalvm.org/
:::

### プロジェクト作成

https://start.ktor.io/settings でプロジェクトを作成・ダウンロード  
※ GraalVMで動かすには、Engineを CIO(Coroutine-based I/O) に選択する必要がある. 

### GraalVM のインストール

```sh
graalvm-ktor ➤ mise use java@oracle-graalvm-23.0.2

graalvm-ktor ➤ java --version
java 23.0.2 2025-01-21
Java(TM) SE Runtime Environment Oracle GraalVM 23.0.2+7.1 (build 23.0.2+7-jvmci-b01)
Java HotSpot(TM) 64-Bit Server VM Oracle GraalVM 23.0.2+7.1 (build 23.0.2+7-jvmci-b01, mixed mode, sharing)
```

### Gradle プラグイン追加

`native-image` CLI を Gradle から使うため、**Native Build Tools** プラグインを追加します。  
（追加後、`nativeCompile` / `nativeRun` などのタスクが使えます）

```toml:libs.versions.toml
[versions]
graalvm-native-build-tools = "0.11.0"
:
[plugins]
graalvm-native-buildtools = { id = "org.graalvm.buildtools.native", version.ref = "graalvm-native-build-tools" }
```

```kotlin:build.gradle.kts
plugins {
    :
    alias(libs.plugins.graalvm.native.buildtools)
    :
}
```
::: details build.gradle.kts - Github
https://github.com/hondaya14/graalvm-ktor/blob/main/build.gradle.kts
:::

### 実行

```sh
graalvm-ktor ➤ ./gradlew nativeRun
```

まず AOT コンパイル（Native Image 生成）が走ります（抜粋）。

```sh
> Task :nativeCompile
[native-image-plugin] GraalVM Toolchain detection is disabled
[native-image-plugin] GraalVM location read from environment variable: JAVA_HOME
[native-image-plugin] Native Image executable path: /Users/nqv/.local/share/mise/installs/java/oracle-graalvm-23.0.2/lib/svm/bin/native-image
========================================================================================================================
GraalVM Native Image: Generating 'graalvm-ktor' (executable)...
...
 Java version: 23.0.2+7, vendor version: Oracle GraalVM 23.0.2+7.1
 Graal compiler: optimization level: 2, target machine: armv8-a, PGO: ML-inferred
 C compiler: cc (apple, arm64, 17.0.0)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
...
  72.02MB in total
...
Build artifacts:
 /Users/nqv/Repositories/Practice/graalvm-ktor/build/native/nativeCompile/graalvm-ktor (executable)
========================================================================================================================
Finished generating 'graalvm-ktor' in 1m 27s.
[native-image-plugin] Native Image written to: /Users/nqv/Repositories/Practice/graalvm-ktor/build/native/nativeCompile
```

1m 27sと少しコンパイルに時間はかかる。
次に、生成されたネイティブバイナリが起動します。

```sh
> Task :nativeRun
[2025-08-21 02:55:39.855][main] INFO  io.ktor.server.Application - Autoreload is disabled because the development mode is off.
[2025-08-21 02:55:39.864][main] INFO  io.ktor.server.Application - Application started in 0.018 seconds.
[2025-08-21 02:55:39.866][DefaultDispatcher-worker-4] INFO  io.ktor.server.Application - Responding at http://0.0.0.0:8080
```

### ハマりポイント

GraalVM Native Image は **リフレクションを部分的にサポート**しており、**実行時にリフレクションで触る要素をビルド時に申告**する必要があります。ref: https://www.graalvm.org/22.1/reference-manual/native-image/Reflection/
そのため、`resources/META-INF/native-image/reflect-config.json` を用意して対象クラスを登録します。以下は **CIO 内部で利用するソケット周りのハンドラ参照**の例です。

```json:reflect-config.json
[
  {
    "name": "io.ktor.network.selector.InterestSuspensionsMap",
    "fields": [
      { "name": "readHandlerReference",  "allowUnsafeAccess": true },
      { "name": "writeHandlerReference", "allowUnsafeAccess": true },
      { "name": "acceptHandlerReference", "allowUnsafeAccess": true },
      { "name": "connectHandlerReference", "allowUnsafeAccess": true }
    ]
  }
]
```

**Agent で収集する方法**もあり、一度jarをbuildし、エージェント`native-image-agent`を設定し実行してリフレクション情報を取得し、出力できる. 

```sh
graalvm-ktor ➤ java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image -jar build/libs/graalvm-ktor-all.jar
[2025-08-21 23:43:43.605][main] INFO  io.ktor.server.Application - Autoreload is disabled because the development mode is off.
[2025-08-21 23:43:43.645][main] INFO  io.ktor.server.Application - Application started in 0.13 seconds.
[2025-08-21 23:43:43.659][DefaultDispatcher-worker-1] INFO  io.ktor.server.Application - Responding at http://0.0.0.0:8080
```

`reachability-metadata.json` に Agent が収集したリフレクション情報が出力されます。

が、実行時リフレクションをagentで回収すると言っても、全てのreflectionを回収できるわけではないのと、手で設定すると言うってもagentと同様に全て回収するのは不可能なので、この辺りどう扱えば良いかが不明。。。。

## まとめ

| | Kotlin/Native | GraalVM Native Image |
|-|---|---|
| pros/cons | - **C資産を cinterop で扱える** <br>- **JVMライブラリは不可**(代替が必要) | - 既存の**JVM資産を使える**<br>- **AOTコンパイル時にリフレクション周りの扱いが難しい**|
| Usecase | - **JVM なしで配りたい単体バイナリ**（CLI/小さな常駐サーバ）<br>- **C 資産を利用**したいシーン | - 既存JVMを**そのままAOT化**し**パフォーマンス向上**(起動時間短縮, メモリ効率)したいシーン<br>- **コンテナ起動頻度が高い**ワークロード |



## 参考リンク

- Kotlin/Native Overview: <https://kotlinlang.org/docs/native-overview.html>  
- LLVM: <https://llvm.org/>  
- cinterop: <https://kotlinlang.org/docs/native-c-interop.html>  
- Ktor: <https://github.com/ktorio/ktor>  
- GraalVM: <https://www.graalvm.org/>  
- GraalVM Native Image（Reflection）: <https://www.graalvm.org/22.1/reference-manual/native-image/Reflection/>
