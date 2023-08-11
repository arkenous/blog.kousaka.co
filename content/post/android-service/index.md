---
title: "Android Serviceについてのメモ"
date: 2017-08-19T22:00:00+09:00
description: "AndroidのService周りについて、調べたことをメモする。"
slug: android-service
tags:
  - Android
---

AndroidのService周りについて、調べたことをメモする。

Android Serviceはいくつかの種類があり、呼び出し方によってその形態が変わる。

- startService
- bindService

startServiceを用いた場合、呼び出し元のActivity等が死んでも開始したServiceは影響を受けない。stopSelfでService自身に停止させるか、他のクラスからstopServiceで停止させる。bindServiceを用いた場合、Activity等とbindしている間だけServiceが動く。つまりServiceにbindされているインスタンスが無くなった（全てunbindされた）時点で終了する。

また、システムのメモリリソースが少なくなり、リソースを解放する必要が生じた時に、Serviceが強制終了される可能性がある。ただし、bind中であるとか、Foregroundで動作中のService（後述）が停止される可能性は極めて低い。Serviceが強制終了されると、リソースが回復次第Serviceが再起動される。

Service実行後の `onStartCommand` 中で `startForegorund` を実行することで、ServiceをForeground Serviceとして実行できる。これによりステータスバー（通知バー）にService起動の通知が常駐し、リソースが少なくなってもシステムによるService強制終了の候補とならない。また、Foreground Serviceとして実行することで、デバイスがSleepの状態に遷移しても処理が継続する。

Serviceが実行されると、Toastやステータスバー（通知バー）を用いて通知を送ることができる。

Serviceの注意点として、アプリケーションで宣言されたServiceはアプリケーションと同じプロセスで、そのアプリケーションのメインスレッドで動作することである。つまり、同じアプリケーションでServiceを動かしつつUI操作を行う場合、パフォーマンスが悪化する恐れがある。そのため、`HandlerThread / Handler`を上手く用いてService内で新しいスレッドを作成し、その中でServiceの処理を行わせるのがベターである。

Serviceを起動する`startService`は複数回呼び出しても問題ない。Serviceが起動していなければ起動する（`onCreate`から呼ばれる）が、既に起動している場合は`onStartCommand`から処理が流れる。このことから、Serviceに何かパラメータを渡す場合はIntentにパタメータをセットしてその都度`startService`を呼び出せばいい。

Serviceから処理結果を返したい場合はBroadcastReceiverを用いる。しかしこの方法ではActivityが破棄されるなどでReceiverの登録が解除されていた場合に処理結果を受け取ることができない。NotificationとTaskStackBuilderを用い、`stackBuilder.addNextIntent`にパラメータを入れたIntentをセットし、Activityの`onNewIntent`で受け取るのが良さそう。

```java
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {

   // some codes you want to process as service

   val builder = NotificationCompat.Builder(this)
                   .setSmallIton(R.mipmap.ic_launcher)
                   .setContentTitle("SampleService")
                   .setContentText("Completed")

   val resultIntent = Intent(this, MainActivity::class.java)
   resultIntent.putExtra("result", result)

   val stackBuilder = TaskStackBuilder.create(this)
   stackBuilder.addParentStack(MainActivity::class.java)
   stackBuilder.addNextIntent(resultIntent)
   val resultPendingIntent = stackBuilder.getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT)

   builder.setContentIntent(resultPendingIntent)

   val notificationManager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
   notificationManager.notify(startId, builder.build())
}

private var result: Long = -1

override fun onCreate(savedInstanceState: Bundle?) {
   super.onCreate(savedInstanceState)

   onNewIntent(intent)
}

override fun onNewIntent(intent: Intent?) {
   if (intent == null) return
   val extras = intent.extras
   if (extras.containsKey("result")) result = extras.getLong("result")
}
```

