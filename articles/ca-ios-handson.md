---
title: "サイバーエージェントiOS設計インターンに行った感想"
emoji: "💪"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [CyberAgent,intern,iOS,architecture, swift]
published: true
---

　こんにちは。Flutterで個人開発をしている谷口恭一です。
今回は2022年8月16日-18日で開催された「CyberAgent 無視できない！設計が学べる3days iOSハンズオン」に参加した報告をしたいと思います。
https://www.cyberagent.co.jp/careers/students/event/detail/id=27497

# インターンの内容
今回のインターンでは主にiOS開発における設計を学びました。フロントエンド開発における設計とは
- MVCモデル
- MVPモデル
- MVVMモデル
- Fluxアーキテクチャ
- Reduxアーキテクチャ
- DDD（ドメイン駆動設計）
- TDD（テスト駆動設計）
- Clean Architecture

などが挙がると思います。これらの設計手法はそれぞれ目的や長所があります。インターン開始までにこれらの設計手法を学ぶ事前課題がありました。インターンではswift, UIKitで書かれた音楽再生アプリをリアーキテクチャしていくというものであり、事前に使用したいアーキテクチャを決定する必要があります。
　また、いわゆる「設計」とは単に設計手法を適用することだけではありません。コードの可読性, 保守性, 拡張性を高めるための工夫を考えていく必要があります。自分の場合、具体的には以下のような内容となりました。
1. 既存の音楽再生アプリを「MVVMモデル」にリアーキテクチャ
 RxSwiftを用いてView層とViewModel層のデータバインディングを実装
2. DIの意義を学び、ViewModel層に実装
3. ユニットテストを実行してDIの意義を体験する
4. コードの可読性を考慮して全体的なリファクタリング



# 参加する前の技術レベル
現在大学３年生で研究室選びと個人開発に追われています。個人開発ではFlutter×Firebaseで開発をしています

https://apps.apple.com/jp/app/improve-%E8%A8%88%E7%94%BB-%E5%AE%9F%E8%A1%8C-%E5%85%B1%E6%9C%89-%E3%81%AE%E3%82%B5%E3%82%A4%E3%82%AF%E3%83%AB%E3%82%92%E5%9B%9E%E3%81%9D%E3%81%86/id1636323158

https://zenn.dev/kyoichi/articles/effort-app-idea

主要技術はFlutterであり、swiftの経験はありませんでした。個人開発アプリではDDDを採用し、riverpodを用いたProviderパターンでstateを管理しています。
設計に関しては普段から考えながら開発はしていましたが、ユニットテストなどのテストは実装したことがなかった状態でした。そこでインターンの目標を次のように設定することにしました。
- swiftを用いたiOS開発ができるようになる。（swiftやUIKitの仕様を理解できるレベル）
- MVVMを用いて「良い設計」理解し目指す。（本インターンの目標と一致）
- 得た知見を次の個人開発に応用する

swift言語やUIKit, RxSwiftの仕様を全く知らない状態であったことが不安でしたが無事参加することができました。

# 身についたスキル
インターン参加によって確実に身についたスキルを並べようと思います。
- swift, UIKit, RxSwiftの仕様の把握
- MVVMモデルの詳細な理解と実装
- RxSwiftを用いたView層をViewModel層のデータバインディングの実装

# 今後の展望

Flutterを用いた開発の際にも今回の設計に関する知見は役に立ちます。個人開発レベルでもコードの可読性や拡張性, 保守性を維持することの大切さを学びました。
今後の開発者人生の中でも思い出に残る楽しいインターンでした。

# 参考にした記事

## 設計
https://qiita.com/marty-suzuki/items/5a4f680b10bb82501aa3
https://qiita.com/baby-degu/items/d058a62f145235a0f007
## MVVM
https://speakerdeck.com/akifumifukaya/mvvm-overview?slide=16
https://medium.com/blablacar/rxswift-mvvm-66827b8b3f10
## swift関連
https://qiita.com/nsuhara/items/bc06c07ff30a5b78696d
https://qiita.com/Yaruki00/items/25c0ae6edb563458bf2d
https://qiita.com/furuyan/items/2e60b84835581ecf8ca0

## RxSwift
https://qiita.com/marty-suzuki/items/6a5915385692c713b264
https://qiita.com/usamik26/items/444d6dd7386b2949c06b
https://qiita.com/gyama_X/items/1c24bca68a14a92c5ce3
https://qiita.com/moaible/items/de94c574b25ea4f0ef17
https://qiita.com/k5n/items/17f845a75cce6b737d1e