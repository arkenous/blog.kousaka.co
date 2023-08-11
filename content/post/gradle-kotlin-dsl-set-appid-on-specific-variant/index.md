---
title: "Gradle Kotlin DSLで、特定のProduct Flavorの組み合わせからなるBuild VariantでのみApplication ID・Version Name・Version Codeを書き換える方法"
date: 2023-01-16T20:03:29+09:00
description: "Gradle Kotlin DSLで記述しているAndroidアプリのビルドスクリプトで、特定のProduct Flavorの組み合わせにより構成されるBuild VariantでのみApplication ID等を変更したい場合の設定方法をログに残す。"
slug: gradle-kotlin-dsl-set-appid-on-specific-variant
tags:
  - Android
keywords:
  - Android
  - Gradle
  - Gradle Kotlin DSL
---

例えば、「Google Playリリース用であるBuild Variantでのみ、本番用に確保されたApplication IDを使って欲しい」という要件があり、なおかつBuild Variantが複数のdimensionから構成されるものである場合、特定のProduct Flavorの組み合わせの場合のみApplication IDを書き換える必要が生じるかもしれない。

Gradle Kotlin DSLで、構成されたBuild Variantを読み取り、条件に合致する・あるいはしない場合にApplication ID・Version Code・Version Nameを書き換えるためには、以下を実装すれば良い。

```kotlin
// app/build.gradle.kts

android {
  ...
  defaultConfig {
    // Google Playリリース用の設定値をデフォルトとする
    applicationId = "本番用のApplication ID"
    versionCode = 本番用のVersion Code（Int値）
    versionName = "本番用のVersion Name"
    ...
  }
}

...

androidComponents {
  onVariants { variant ->
    if (variant.flavorName?.toLowerCaseAsciiOnly() != "novpnproduction") {
      // Google Playリリース用のBuild Variant以外では、開発用の設定値で書き換える
      variant.applicationId.set("開発用のApplication ID") // Application IDの書き換え
      variant.outputs.forEach { output ->
        output.versionCode.set(開発用のVersion Code（Int値）) // Version Codeの書き換え
        output.versionName.set("開発用のVersion Name") // Version Nameの書き換え
      }
    }
  }
}
```

上記コードにより、特定のBuild Variant以外では開発用の設定値でApplication ID等を書き換える挙動となる。

なお、Product Flavorのdimensionが一つのみであるとか、特定のProduct FlavorでのみApplication ID等を書き換えるだけで済むのであれば、該当するProduct Flavorを作成している箇所でApplication ID等を指定すれば良い。

```kotlin
// app/build.gradle.kts

android {
  ...
  defaultConfig {
    // Google Playリリース用の設定値をデフォルトとする
    applicationId = "本番用のApplication ID"
    versionCode = 本番用のVersion Code（Int値）
    versionName = "本番用のVersion Name"
    ...
  }

  flavorDimensions += listOf(
    "environment",
  )
  productFlavors {
    create("development") {
      dimension = "environment"
      isDefault = true
      applicationId = "開発用のApplication ID"
      versionCode = 開発用のVersion Code（Int値）
      versionName = "開発用のVersion Name"
    }
    create("production") {
      dimension = "environment"
    }
  }
}
```

