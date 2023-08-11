---
title: "Androidアプリプロジェクトに独自のLintルールを導入する"
date: 2022-01-23T10:00:00+09:00
description: "Androidアプリを作っていると、標準で提供されていないLintルールでは満足できない状況がたまにある。独自のLintルールを作成し、Androidアプリプロジェクトに適用する方法を記載する。"
slug: android-lint
tags:
  - Android
---

Androidアプリを作っていると、標準で提供されているLintルールだけでは満足できない、プロジェクト固有のLintルールを導入したくなるケースがたまにある。例えば以下が思いつきそうだ。

- プロジェクトが依存するライブラリにおいて、一部クラスが持つメソッドの実装に不備がある。クラスにfinalが付与されているためoverrideもできず、別途実装したwrapperクラスを介して実装してほしい
- 言語設定の変更が適切に反映されない状況を生むので、AndroidViewModelから取得できるApplication Contextを用い、Resourcesから文字列を取得するような実装は避けてほしい
- MutableLiveDataやMutableStateFlowなどをViewModel外に公開することは、必要最小限にしてほしい

新たにメンバが参加した際にプロジェクト固有のコーディングルールを共有し、プロジェクトメンバ間で共通認識を持つことで上記を意識したコードを書けるとは思う。だが、人の記憶は大抵当てにならない。時間が経つにつれてルールの認識があやふやになり、コーディングルールから外れた実装をしてしまい、コードレビューにて指摘を受けることは誰にだって起こりうる、と思う。

個々人の努力によってルールの認識を徹底してもらうような、人に依拠するやり方は筋が悪い。こういったことはシステムに任せるのが望ましく、Lintはその手法の一つとなる。

この記事では、Androidアプリプロジェクトに対し独自のLintルールを実装・導入し、Android StudioにInspectionsの一つとして認識させることで、エディタ上の該当する箇所での警告表示や、Gradle Taskでの検査、またAndroid Studioの `Analyze / Inspect Code...` でのコード検査を可能とするための方法を記載する。

## 独自Lintルールを実装するモジュールの作成

これから実装していくLint関連のコードは、アプリ本体の機能に直接関係しないものであるため、モジュールを分けて実装することが望ましい。以下手順で独自Lintルール実装用のモジュールを作る。

1. 対象とするアプリプロジェクトをAndroid Studioで開く
2. Projectタブで、プロジェクトツリーのルートを左クリックし、 `New / Module` でモジュール作成ウインドウを出す
3. Java or Kotlin Libraryテンプレートを選び、各項目を適宜変更する（Library nameは `lint` とかが良いかも）

作成できたら、モジュール内部の `build.gradle` を以下のように実装する。

```gradle
plugins {
    id 'kotlin'
}

dependencies {
    // 以下二つのライブラリのバージョンは、Android Gradle Pluginのメジャーバージョンに23を足した値を指定すること
    // e.g. AGP: 7.0.4 -> 30.0.4
    // https://github.com/googlesamples/android-custom-lint-rules#lint-version
    compileOnly "com.android.tools.lint:lint-api:30.0.4"
    compileOnly "com.android.tools.lint:lint-checks:30.0.4"
}

sourceCompatibility = JavaVersion.VERSION_1_8
targetCompatibility = JavaVersion.VERSION_1_8
```

上記実装できたら、次はLintの検査ロジックであるDetectorを実装していく。

## Detectorの実装

Detectorクラスの内部に、検査ロジックと、Lintルールそのものの情報をまとめたIssueをそれぞれ実装することとする。Issueを別クラスに分けて実装することももちろん可能なので、Detectorの検査ロジックに要するコード行数が長くなりそうなら、クラスを分けることも検討して良さそうだ。

### 検査ロジックの実装

Detectorを実装するにあたり、どのようなファイルを対象とするかに応じ、継承元を決める必要がある。前述の作業にてGradle Syncを済ませ、Lint関連のライブラリを導入済みであれば、 `com.android.tools.lint-detector.api.Detector` クラスにて、Lintの対象ファイルに応じたInterfaceが定義されている。以下コードを実装する際にコード補完で表示されるだろうし、仮でいずれかのインタフェースを書いておき、そのインタフェース名（以下だと `Detector.UastScanner` ）をCommand + クリックすれば定義元にコードジャンプできるので、目的に応じて選択すると良い。

この記事ではJavaあるいはKotlinのソースコードを対象としたいので、 `UastScanner` を継承したDetectorを実装する。

```kotlin
@Suppress("UnstableApiUsage")
internal class SampleDetector : Detector(), Detector.UastScanner {
    // TODO: Applicableを定義し、何を検査対象とするか、より具体的に絞り込む（デフォルトだと、何も対象としない）

    // TODO: 定義したApplicableに適合するイベントコールバックメソッドをoverrideし、検査ロジックを実装する
}
```

クラスを作ったら、まずはじめに何を検査対象とするかを、Applicableを実装して定義する。

```kotlin
// 特定の型のコンストラクタを呼び出している箇所を対象に含める
override fun getApplicableConstructorTypes(): List<String> {
    return listOf(
        SAMPLE_CLASS,
    )
}

companion object {
    private const val SAMPLE_CLASS = "com.example.sampleapp.SampleClass"
}
```

次に、定義したApplicableに適合するイベントコールバックメソッドを実装し、そこで検査ロジックを組んでいく。

```kotlin
override fun visitConstructor(context: JavaContext, node: UCallExpression, constructor: PsiMethod) {
    // Applicableで検査対象を絞っているので、ここが呼ばれる == SampleClassのコンストラクタが呼ばれたとみなす
    context.report(
        // Lintルールに関する情報をまとめたもの。後ほど実装する
        issue = ISSUE_SAMPLE,
        // どの範囲を警告として視覚化するか。大抵はUASTあるいはPSIのクラスに応じてよしなに解釈してくれる
        // LocationTypeを明示して、レシーバまで含めるとか、構文全てで表示する、とかもできる
        location = context.getLocation(node),
        // 以下で指定したメッセージが、Inspectionや、あるいはエディタで波線をマウスホバーした際のポップアップにて表示される
        message = "We shouldn't use SampleClass.",
    )
}
```

このように、Applicableを定義する -> 適合するイベントコールバックメソッドで検査ロジックを実装する、というのが一連の流れになる。

他に、いくつか実装してみたものがあるので、それらを以下に例として載せる。

```kotlin
// 検査するUastTypeを列挙する
// ここで列挙したものを、それぞれ対応するUElementHandlerのイベントコールバックメソッドで検査する
override fun getApplicableUastTypes(): List<Class<out UElement>> {
    return listOf(
        UBinaryExpression::class.java, // 二項演算の構文を検査対象に含める
        UCallExpression::class.java, // メソッド・コンストラクタ・配列初期化呼び出しを検査対象に含める
        UField::class.java, // フィールドを検査対象に含める
        UParameter::class.java, // パラメータを検査対象に含める
    )
}

override fun createUastHandler(context: JavaContext): UElementHandler {
    return object : UElementHandler() {

        // 二項演算の構文を検査する
        override fun visitBinaryExpression(node: UBinaryExpression) {
            val leftOperandType = node.leftOperand.getExpressionType()?.canonicalText ?: return
            if (leftOperandType.contains(SAMPLE_CLASS).not()) return

            val operator = node.operator
            if (operator == UastBinaryOperator.EQUALS || operator == UastBinaryOperator.NOT_EQUALS) {
                context.report(
                    issue = ISSUE_SAMPLE,
                    location = context.getLocation(node),
                    message = "Since lacks of implementing equals of SampleClass, we may lead unexpected behavior if we check equality around SampleClass.",
                )
            }
        }

        // メソッド・コンストラクタ・配列初期化の呼び出しを検査する
        override fun visitCallExpression(node: UCallExpression) {
            val receiverType = node.receiverType?.canonicalText
            if (receiverType?.contains(SAMPLE_CLASS) == false) return
            if (node.methodName == "equals") {
                context.report(
                    issue = ISSUE_SAMPLE,
                    location = context.getCallLocation(
                        call = node,
                        includeReceiver = true,
                        includeArguments = false,
                    ),
                    message = "Since lacks of implementing equals of SampleClass, we may lead unexpected behavior if we check equality around SampleClass.",
                )
            }
        }

        // フィールドを検査する
        override fun visitField(node: UField) {
            if (node.type.canonicalText.contains(SAMPLE_CLASS)) {
                context.report(
                    issue = ISSUE_SAMPLE,
                    location = context.getLocation(node),
                    message = "We should use FixedSampleClass rather than SampleClass.",
                )
            }
        }

        // パラメータを検査する
        override fun visitParameter(node: UParameter) {
            if (node.type.canonicalText.contains(SAMPLE_CLASS)) {
                context.report(
                    issue = ISSUE_SAMPLE,
                    location = context.getLocation(node, type = LocationType.DEFAULT),
                    message = "We should use FixedSampleClass rather than SampleClass.",
                )
            }
        }
    }
}
```

Applicableとイベントコールバックメソッドの関連付けなどは、以下の記事も参照すると良いかと思う。

[Enforcing Team Rules with Lint: Detectors 🕵️](https://zarah.dev/2020/11/19/todo-detector.html)

さて、ここまでで検査ロジックの実装ができた。次に、横に置いていたIssueの作成を行う。

### Lintルールの情報をまとめた、Issueの作成

先の検査ロジック実装例でも含めていた、ISSUE_SAMPLEの実装を行う。ここでは、検査ロジックを実装したDetector継承クラスのcompanion objectとして実装する。

```kotlin
@Suppress("UnstableApiUsage")
internal class SampleDetector : Detector(), Detector.UastScanner {
    // Applicable実装済み
    // イベントコールバックメソッドでの検査ロジック実装済み

    companion object {
        private const val SAMPLE_CLASS = "com.example.sampleapp.SampleClass"

        private val ISSUE_SAMPLE = Issue.create(
            id = "SampleClass",
            briefDescription = "Using SampleClass",
            explanation = "Since lacks of implementing equals of SampleClass, we may lead unexpected behavior if we check equality around SampleClass.",
            category = Category.CORRECTNESS,
            priority = 5,
            severity = Severity.WARNING,
            implementation = Implementation(SampleDetector::class.java, Scope.JAVA_FILE_SCOPE),
        )

        val issue = ISSUE_SAMPLE
    }
}
```

Issue.createの各パラメータについては、メソッドのJavadocを見てもらえれば理解できると思う。また、Android Studioの環境設定で、Editor / Inspectionsに列挙されている項目を眺めると、idやbriefDescription, explanationに記載する内容のあたりを付けられると思う。

これで、Detectorの実装は完了となる。次に、作成したIssueをAndroidアプリプロジェクト上で利用するために必要となるIssueRegistryの実装と、lintモジュールのbuild.gradleにて、生成するJarのManifestへのIssueRegistry参照先定義を実装する。

## IssueRegistryの実装

### 独自LintルールをAndroidアプリプロジェクトで利用するための、IssueRegistryの実装

IssueRegistryを継承したクラスを実装し、Androidアプリプロジェクト上で独自Lintルールを扱えるよう対応する。

```kotlin
@Suppress("UnstableApiUsage")
internal class IssueRegistry : IssueRegistry() {
    override val api: Int
        get() = CURRENT_API

    override val issues: List<Issue>
        get() = listOf(
            SampleDetector.issue,
        )
}
```

上記でIssueRegistryの実装は完了となる。

### lintモジュールのbuild.gradleにて、生成するJarのManifestへのIssueRegistry参照先定義を実装する

lintモジュールのbuild.gradleにて、Lintモジュールから生成するJarのManifestに `Lint-Registry` attributeを定義することにより、後述する `lintChecks` を指定した各モジュールにてLintを効かせられる。

本記事の冒頭で作成し、前述の各実装を行ったlintモジュールのbuild.gradleに、以下を追加する。

```gradle
plugins {
    id 'kotlin'
}

dependencies {
    // 以下二つのライブラリのバージョンは、Android Gradle Pluginのメジャーバージョンに23を足した値を指定すること
    // e.g. AGP: 7.0.4 -> 30.0.4
    // https://github.com/googlesamples/android-custom-lint-rules#lint-version
    compileOnly "com.android.tools.lint:lint-api:30.0.4"
    compileOnly "com.android.tools.lint:lint-checks:30.0.4"
}

sourceCompatibility = JavaVersion.VERSION_1_8
targetCompatibility = JavaVersion.VERSION_1_8

// 本記事冒頭の作業により、ここから上は既に実装済みのはず。以下を追加する。

jar {
    manifest {
        // 先ほど実装したIssueRegistryの完全修飾名（Fully Qualified Name）を書く
        attributes 'Lint-Registry': 'com.example.sampleapp.lint.IssueRegistry'
    }
}
```

ここまでの作業により、独自Lintルールの実装は完了した。あとは、次のように独自Lintルールを追加したいモジュールのbuild.gradleにてlintChecksを有効にする。

## 独自のLintルールを適用する

実装が完了した独自Lintルールを適用する場合、適用先モジュールのbuild.gradleにおいて、dependenciesに以下を追加すると良い。

```gradle
dependencies {
    // projectに渡すモジュール名は、本記事冒頭で作成したLintモジュールのモジュール名で実装する
    lintChecks project('lint')
}
```

以上で導入が完了した。プロジェクトをビルドすれば、Detectorの実装に従って、Lintが効くようになるはずだ。

## 検査方法

### エディタ上での警告表示

エディタ上での警告表示に関しては、前述の各モジュールに対する `lintChecks` の指定により、動作していると思う。Android Studioの環境設定において、 Editor / Inspections に実装したIssueがリストアップされているはずだ。

### Gradle Taskでの検査

Gradle Taskとして定義があるので、例えば以下のようにTaskを実行することで、コードを検査できる。

```
// プロジェクトのルートディレクトリに gradlew があると思うので、これを用いる
// 以下は、appモジュールに対して検査する場合
// 最後にLintで検査した時点からソースコードに変更が無い場合は、以下を実行しても検査結果は再度出力されない
$ ./gradlew app:lintDebug
```

Lintに関する設定、例えばテキスト形式で検査結果を出力して欲しいとか、HTMLでも出力して欲しいとかがあれば、build.gradle内に `android` ブロックを持つモジュールに対し、以下のように設定を調整すると良い。

```gradle
android {
    lintOptions {
        textReport true
        htmlReport true
        xmlReport false
    }
}
```

### Android Studioの Analyze / Inspect Code での検査

Android Studioの Analyze / Inspect Code で検査する場合、作成し実装したlintモジュールから生成されるJARファイルを `~/.android/lint/` の中に配置すると良い。lintモジュールがビルドされた際に、以下ディレクトリにJARファイルが生成されているはずだ。

```
$ lint/build/libs/
```

## Detectorの実装Tips

### デバッグの方法

Applicableに適合するイベントコールバックメソッドがどれかを調べる場合、 `kotlin.io.print(message: Any?)` や `kotlin.io.println(message: Any?)` といったコンソールログ出力用関数を用いたprintデバッグが使える。Android Studioのログは、macOSの場合 `~/Library/Logs/Google/AndroidStudio2020.3/idea.log` のようなログファイルに出力される。

printやprintlnでの出力は標準出力となり、ログファイル内で `STDOUT` という文言が付与されるようだ。なので、以下のように `tail` のfオプションでファイルの更新を監視し、 `grep` で `STDOUT` が含まれる行に絞ってターミナルエミュレータに表示すると良いだろう。

```
$ cd ~/Library/Logs/Google/AndroidStudio2020.3/
$ tail -f idea.log | grep ".*STDOUT.*"
```

### 検査ロジックの実装にあたって

検査ロジックの実装にあたって、本記事ではごく簡単な実装事例を挙げた。より複雑な、あるいはより厳密なLintルールを作る場合、イベントコールバックメソッドで受け取るUASTクラスやPSIクラスから抽象構文木を解析し、JavaContextから得られるJavaEvaluetorも活用して検査していくことになると思われる。前述のprintデバッグや、UASTクラス・PSIクラスの定義元を見て各クラスが持つ情報を確認し、[Android Code SearchでAndroidアプリ開発時にいつもお世話になっている既存のLintの内部実装](https://cs.android.com/android-studio/platform/tools/base/+/mirror-goog-studio-master-dev:lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks/)も参考にしつつ、実装を進めることになるだろう。

# 参考
- [Enforcing Team Rules with Lint: Detectors 🕵️](https://zarah.dev/2020/11/19/todo-detector.html)
- [Get started with Android Lint - Custom Lint Rules](https://jayrambhia.com/blog/android-lint)
- [GitHub - googlesamples/android-custom-lint-rules: This sample demonstrates how to create a custom lint checks and corresponding lint tests](https://github.com/googlesamples/android-custom-lint-rules)
- [GitHub - alexjlockwood/android-lint-checks-demo: A demo project that shows how to setup and write some basic custom lint checks.](https://github.com/alexjlockwood/android-lint-checks-demo)
[社内Android勉強会でAndroid Lintを実装して得た知見 | BLOG - DeNA Engineering](https://engineering.dena.com/blog/2020/12/getting-started-with-android-lint/)
- [Custom Lint Rules - Qiita](https://qiita.com/hotchemi/items/9364d54a0e024a5e6275)

