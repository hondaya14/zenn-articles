---
title: "Kotlinにおける快適なテスティングエコシステム"
emoji: "🫠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [kotlin, test, oss]
published: true
---

# Open-source tools for your testing ecosystem

持続可能なソフトウェアに欠かせないソフトウェアテスティングにおけるエコシステムにおいて、Kotlinプロジェクトで利用可能で人気のソフトウェアを紹介します。

今回紹介するのは下記です
- [detekt](https://detekt.dev/): Static Code Analyzer (静的解析)
- [kover](https://kotlin.github.io/kotlinx-kover/): Kotlin Coverage Tool (テストカバレッジ)
- [allure](https://allurereport.org/): Testing Report (テストレポートのビジュアライズ)

## Detekt

https://github.com/detekt/detekt

detektの機能は以下の通り
- Code Smell解析
- 豊富なルールセット
- 豊富なレポートフォーマット(HTML, Markdown, SARIF, XML)
- extentionによる拡張性

> What is "Code Smell"?
> 直訳すると、コードの匂い。バグではないが、ソースコードに何か問題が存在する表面的な兆候です.
> 最近ではTidy First?で話題のKent beckが考案したらしいです。 ref. [Code Smell - martinfowler.com](https://martinfowler.com/bliki/CodeSmell.html)


### How to use
```toml:libs.versions.toml
[versions]
detekt = "x.x.x"

[plugins]
detekt = { id = "io.gitlab.arturbosch.detekt:detekt-gradle-plugin", version.ref = "detekt" }
```

```kotlin:build.gradle.kts
plugins {
    alias(libs.plugins.detekt)
}

subprojects {
    apply(plugin = libs.plugins.detekt.get().pluginId)

    dependencies {
        detektPlugins(libs.detekt.formmatting) // formatterを利用する場合
    }

    detekt {
        // detekt configurations
        buildUponDefaultConfig = true
        allRules = true
        autoCorrect = true // formatterを利用する場合
        config.setFrom(file("path/to/config/detekt.yml"))
    }

    tasks.withType<Detekt>().configureEach {
        group = "detekt"
        exclude("**/generated/**") // 自動生成クラスは解析対象にしない設定も可能
    }
}
```

```zsh
 ➤ ./gradlew detekt
or
 ➤ ./gradlew detekt --continue
```
を実行することで、各gradle subprojectごとに、detektのreportが出力されます.

`path/to/root/subproject/build/reports/detekt/`配下に、
- detekt.html
- detekt.md
- detekt.sarif
- detekt.txt
- detekt.xml
が出力されます。(出力形式の指定についてはgradle側の設定で可能)

出力内容は、
- Metrics ... プロジェクト内のproperties/function/class/package/kt fileの数
- Complexity Report ... 
- Findings ... ルール違反したルールとコードが示される.

![](/images/detekt_report.png)
Ref: https://detekt.dev/docs/introduction/reporting


gradle taskでは、detektが解析して、ルール違反するとtask: FAILEDになります。
```zsh
 ➤ ./gradlew detekt

> Task :subproject FAILED
/User/path/to/repository/subproject/path/to/bad/smell/file.kt:27:13: Violate rule message [Rule name]
```

## Kover

https://github.com/Kotlin/kotlinx-kover

koverの機能としては以下の通り (※ Gradle plugin)
- コードカバレッジの収集(ターゲットは`JVM`のみサポート。kotlin/js, kotlin/nativeについては未サポート)
- HTML, XMLレポートの生成
- JVM, KMPプロジェクトのサポート
- Kotlin, Javaが混合していても利用可能
- Jacoco engineも利用可能

### How to use
```diff:libs.versions.toml
[versions]
detekt = "x.x.x"
+kover = "x.x.x"

[plugins]
detekt = { id = "io.gitlab.arturbosch.detekt:detekt-gradle-plugin", version.ref = "detekt" }
+kover = { id = "org.jetbrains.kotlinx.kover", version.ref = "kover" }
```

```kotlin:build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.kover)
}

dependencies {
    // マルチモジュールの場合、レポートをマージするmoduleを依存に追加する
    kover(project(":subproject1"))
    kover(project(":subproject2"))
    kover(project(":subproject3"))
    kover(project(":subproject4"))
    :
}

kover {
    reports {
        filters {
            include {
                classes(
                    "path.to.collect.coverage.package",
                    )
            }
            exclude {
                // 自動生成クラスやアプリケーションのエントリポイントなどテストをそもそも実装しないクラスはカバレッジ収集から除外する
                classes(
                    "*.generated.*",
                    "Application",
                    "ApplicationKt",
                )
            }
        }
    }
}
```

テストカバレッジには種類があります. いかが簡単なサマリです. (IPA試験を思い出しましょう)
| Label | Name(ja) | Name(en) | Kover(CoverageUnit) | Description|
|---|---|---|---|---|
| **C0** | 命令/ステートメント網羅 | Statement coverage | LINE ※default | 命令/行が実行されたか |
| **C1** | 分岐/判定網羅 | Branch/Decision coverage | INSTRUCTION | `if/when/?:/catch` などの**分岐の双方のパターン**（真/偽）を通したか |
| **C2** | 条件網羅 | Condition coverage | BRANCH | 複合条件の**各ブール項目**を真/偽にできたか | 
| **MCC** | 複数条件網羅 | Multiple Condition coverage | N/A | すべての条件**組合せ**を網羅 |
|  | 経路網羅 | Path coverage | N/A | 可能な実行経路すべて |


koverはデフォルトでLINE, つまりC0レベルでのカバレッジの網羅率を計測します。C1(INSTRUCTION), C2(BRANCH)についてもConfigurationで設定可能。
そのほか、AggregationType(集計方法: 件数なのか絶対量なのか率(%)なのか)やGroupingEntitiyType(粒度と境界: ApplicationなのかClassなのかPackageでグルーピングして値を出すか)などの設定項目もあります。


## Allure

https://github.com/allure-framework/allure2
https://github.com/allure-framework/allure-gradle


allureの機能については以下の通り
- testのレポートのビジュアライズ
- 複数testの結果比較
- エラーメッセージやstack trace、そのほかのデバッグ情報の可視化
- テストランタイムデータの提供
- timeline toolによるテストのパフォーマンスや並行問題の発見
- 個々の機能やテストスイートで可視化


### How to use

```diff:libs.versions.toml
[versions]
detekt = "x.x.x"
kover = "x.x.x"
+allure = "x.x.x"
+kotest-allure = "x.x.x" // For kotest extention version

[libraries]
+# kotestと併用する場合, extentionの実装依存を追加する
+kotest-extentions-allure = { module = "io.kotest:kotest-extensions-allure", version.ref = "kotest-allure" }

[plugins]
detekt = { id = "io.gitlab.arturbosch.detekt:detekt-gradle-plugin", version.ref = "detekt" }
kover = { id = "org.jetbrains.kotlinx.kover", version.ref = "kover" }

+# io.qameta.allure pluginのdistributionもあるがマルチモジュール構成の場合、レポートをaggregateしたいのでこちらを利用する
+allure-aggregate-report = { id = "io.qameta.allure-aggregate-report", version.ref = "allure" }

+# aggregateの方を利用する場合、adapter pluginが別途必要
+allure-adapter = { id = "io.qameta.allure-adapter", version.ref = "allure"}
```

```kotlin:build.gradle.kts
plugins {
    alias(libs.plugins.allure.aggregate.report)
    alias(libs.plugins.allure.adapter)
}

subprojects {
    apply(plugin = libs.plugin.allure.aggregate.report.get().pluginId)
    apply(plugin = libs.plugin.allure.adapter.get().pluginId)
}
```

kotestと併用する場合、extentionを別途依存追加する必要がある. [Kotest - Allure](https://kotest.io/docs/extensions/allure.html)

また、kotest側のconfigのextentionsに登録して、allure用のデータを収集できるようにする必要がある.
```kotlin: KotestConfig
class KotestConfig : AbstractProjectConfig() {
    override fun extentsion() = listOf(
        AllureTestReporter()
    )
}
```

report画面

![](/images/kotest-allure.png)
Ref: https://kotest.io/docs/extensions/allure.html


そのほかにもテストの実行時間のduration graphやtimelineでの可視化も可能なので下記を参照してください。
- https://allurereport.org/docs/visual-analytics/
- https://allurereport.org/docs/timeline/


# まとめ
今回はテストにおけるライブラリを紹介しました。今回の3つで

- Linter (Formatter ... detekt同梱)
- Code Smell Detection
- Test Coverage
- Test Report Visualize

が実現でき、プロダクトコードの品質をあげる, ないしはあげる為の可視化がプロジェクトに導入できます。
Github Actionsなどで動作するCIに組み込むことでチーム開発におけるコード品質の向上が期待できます。

最後にテストといえば、t_wada sanを引用しないと終われないので、こちらのslideをば
https://speakerdeck.com/twada/building-automated-test-culture-2024-winter-edition?slide=91

> ...ソフトウェアの成長を持続可能にすること
> by t_wada

自動テストやそのエコシステムを整えて、ソフトウェアの成長を持続可能にしていきましょう。




Embracing Kotlin! Kotlinを愛でよう.
