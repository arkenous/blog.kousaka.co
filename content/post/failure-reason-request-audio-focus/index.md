---
title: "requestAudioFocusが失敗するケースとは"
date: 2017-07-22T22:00:00+09:00
description: "Androidでムービー・オーディオ周りを扱うことがあり，フォアグラウンドで再生時にバックグラウンドで再生されている音楽等を自動的に停止させたくてrequestAudioFocus周りを調べていた．だが，APIドキュメント等を調べても，このリクエストに失敗するのがどのようなケースなのかについて具体的に書かれたものがなかった．そのためAndroidのソースコードを読んで分かったことを記す．"
slug: failure-reason-request-audio-focus
tags:
  - Android
---

Androidでムービー・オーディオ周りを扱うことがあり，フォアグラウンドで再生時にバックグラウンドで再生されている音楽等を自動的に停止させたくて`requestAudioFocus`周りを調べていた．

だが，APIドキュメント等を調べても，このリクエストに失敗するのがどのようなケースなのかについて具体的に書かれたものがなかった．そのためAndroidのソースコードを読んで分かったことを記す．

- `requestAudioFocus`でAudioFocusを取得することで，フォアグラウンドで音を再生する際にバックグラウンドで再生中の音をどう扱うかを制御できる
- ただし，あくまでrequestのため，失敗するケースがある
- Android NougatのAPIソースコードより，**着信中・通話中のケースでAudioFocusのリクエストが失敗する**

## requestAudioFocusとは

Androidでは，音の再生や音量の調整などの制御をどのアプリに握らせるのかという部分を[android.media.AudioManager](https://developer.android.com/reference/android/media/AudioManager.html)クラスで管理している．これの[requestAudioFocus](https://developer.android.com/reference/android/media/AudioManager.html#requestAudioFocus(android.media.AudioManager.OnAudioFocusChangeListener,%20int,%20int))を用いることでAudioFocusをリクエストでき，リクエストに成功した（`AUDIOFOCUS_REQUEST_GRANTED`が返ってきた）時点でバックグラウンド再生されていた音楽等が停止（`durationHint`に`AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`を用いた場合は，バックグラウンド再生されていた音楽等の音量が一時的に下がる）される．

しかし，AudioFocus取得のメソッド名が`requestAudioFocus`であること，またこの返り値として`AUDIOFOCUS_REQUEST_GRANTED`が用意されていることからわかるように，`AUDIOFOCUS_REQUEST_FAILED`が返ってくる，つまりAudioFocus取得に失敗するケースが存在する．具体的にどのようなケースにおいてリクエストに失敗するのかについて，[requestAudioFocusのAPI Docs](https://developer.android.com/reference/android/media/AudioManager.html#requestAudioFocus(android.media.AudioManager.OnAudioFocusChangeListener,%20int,%20int))や[Audio Handling周りの公式ドキュメント](https://developer.android.com/guide/topics/media-apps/volume-and-earphones.html)等を調べてみたが，どこにもそれらしきことをはっきりと明記したものは見当たらなかった．

## リクエストが失敗するケースとは

Android Nougat 7.0.0 のソースコードを辿る[^codesearch]と，以下のような流れになっていた．

1. `android.media.AudioManager#requestAudioFocus`
2. `com.android.server.audio.AudioService#requestAudioFocus`
3. `com.android.server.audio.MediaFocusControl#requestAudioFocus`

そして，`com.android.server.audio.MediaFocusControl#requestAudioFocus`内で呼ばれている`com.android.server.audio.MediaFocusControl#canReassignAudioFocus`の結果返されるboolean値を判断材料として用いていることが分かった．

`com.android.server.audio.MediaFocusControl#canReassignAudioFocus`は以下のように実装されている．

```java
/**
 * Helper function:
 * Returns true if the system is in a state where the focus can be reevaluated, false otherwise.
 * The implementation guarantees that a state where focus cannot be immediately reassigned
 * implies that an "locked" focus owner is at the top of the focus stack.
 * Modifications to the implementation that break this assumption will cause focus requests to
 * misbehave when honoring the AudioManager.AUDIOFOCUS_FLAG_DELAY_OK flag.
 */
private boolean canReassignAudioFocus() {
// focus requests are rejected during a phone call or when the phone is ringing
// this is equivalent to IN_VOICE_COMM_FOCUS_ID having the focus
if (!mFocusStack.isEmpty() && isLockedFocusOwner(mFocusStack.peek())) {
    return false;
}
	  return true;
}
```

また，`com.android.server.audio.MediaFocusControl#isLockedFocusOwner`は以下のように実装されている．

```java
private boolean isLockedFocusOwner(FocusRequester fr) {
    return (fr.hasSameClient(AudioSystem.IN_VOICE_COMM_FOCUS_ID) || fr.isLockedFocusOwner());
}
```

`android.media.AudioSystem#IN_VOICE_COMM_FOCUS_ID`は以下のように定義されている．

```java
public final static String IN_VOICE_COMM_FOCUS_ID = "AudioFocus_For_Phone_Ring_And_Calls";
```

`com.android.server.audio.MediaFocusControl#canReassignAudioFocus`のコメント等から，**電話の着信中もしくは通話中にAudioFocusをリクエストすると，`AUDIOFOCUS_REQUEST_FAILED`が返されて失敗する**ようだ．

ついつい忘れそうになるが，Androidスマートフォンも携帯電話の一つなので，電話機能を優先するのは然もありなんといったところだろう．

---

### 参考・関連

- [Handling Changes in Audio Output | Android Developers](https://developer.android.com/guide/topics/media-apps/volume-and-earphones.html)
- [AudioManager | Android Developers](https://developer.android.com/reference/android/media/AudioManager.html)
- [音を制御する - AudioManager - Qiita](http://qiita.com/KeithYokoma/items/3896f5934478fa560a50)

[^codesearch]: Androidのソースコードに潜る際は，[Android Code Search](https://cs.android.com/)を利用した．

