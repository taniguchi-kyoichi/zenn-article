---
title: "IsarとRiverpodでローカルDB管理"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Isar", "Riverpod", "Flutter", "Dart"]
published: true
---

こんにちは、Flutterで個人アプリ開発をしている大学生です。数週間ほど前に新しいアプリをリリースしました。どんなアプリかというと、
#### 自己管理ができなくて、だらけてしまう。。。
という人のためのゲーミフィケーション習慣化アプリです！
- ##### 「勉強30分50円」のようにやるべきことに報酬を設定して、自己管理をする
- ##### 「旅行40,000円」のように、"ごほうび"を設定してお金が溜まったら換金できる
というコア機能を持っています。つまり・・・
### RPGゲームのレベル上げをするような習慣化
を実現できます！！ぜひダウンロードしてみてください！
https://apps.apple.com/jp/app/%E3%81%94%E3%81%BB%E3%81%86%E3%81%B3%E7%BF%92%E6%85%A3-%E3%82%B2%E3%83%BC%E3%83%A0%E6%84%9F%E8%A6%9A%E3%81%A7%E8%87%AA%E5%B7%B1%E7%AE%A1%E7%90%86/id1671700938

このアプリでは、ローカルDBに次々値を保存・編集していくような機能が必要でした。
そこでIsarというライブラリが見つかり、使ってみると結構便利で高速でした。
またIsarは開発されてからそこまで時間が経っていないので、Web上にあまり記事がありませんでした。
よって、この記事ではIsarの基本と実際の開発でありそうなシチュエーションである、Riverpodとの連携を解説していきたいと思います。


## [Isar Database](https://pub.dev/packages/isar "Isar")
[Isar](https://pub.dev/packages/isar "Isar")はローカルDBを管理するパッケージです。[Hive](https://pub.dev/packages/hive "Hive")というパッケージが先にあり、その後継となっています。
Isarはリレーショナルデータベースの[sqflite](https://pub.dev/packages/sqflite "sqflite")と異なり、キーバリュー形式でデータを保存するNoSQLです。Firebase Firestoreと同じような形式と言えます。
自分はまだ使ったことはありませんが、全文検索機能があるらしい！
また、Isarは優秀な日本語マニュアルがあるので、相当学習コストが低いと思います。
https://isar.dev/ja/tutorials/quickstart.html

このドキュメントを読めば、一通りの実装方法はわかりますが、簡単に要約します。
Isarは以下の手順で実装できます。
### ライブラリのインポート
```shell
$ flutter pub add isar isar_flutter_libs
$ flutter pub add -d isar_generator build_runner
```

### データの定義
```dart
part 'email.g.dart';

@collection
class User {
  Id id = Isar.autoIncrement; // id = nullでも自動インクリメントされます。

  String? name;

  int? age;
}
```
IDは自動インクリメントではなく、自身で指定することも可能。
Isarのクラスはコレクションと呼ばれる。（Firestoreと同じ）
またフィールドには以下のデータ構造が対応している。

- bool
- byte
- short
- int
- float
- double
- DateTime
- String
- List<bool>
- List<byte>
- List<short>
- List<int>
- List<float>
- List<double>
- List<DateTime>
- List<String>

また、この他にEnumも対応しています。これは使い勝手がいい！



### コード生成
```shell
$ flutter pub run build_runner build
```
freezedのように `--delete-conflicting-outputs`オプションを付けたほうが良いかもしれません。


### Isarインスタンスの取得, CRUD
```dart
// 保存するパスの指定（パスを指定しなくても良い）
var path = '';
if (!kIsWeb) {
    final dir = await getApplicationSupportDirectory();
    path = dir.path;
}
isar = await Isar.open(
    [EmailSchema,],
    directory: path,
);

final newUser = User()..name = 'Jane Doe'..age = 36;

await isar.writeTxn(() async {
  await isar.users.put(newUser); // 挿入と更新
});

final existingUser = await isar.users.get(newUser.id); // 取得

await isar.writeTxn(() async {
  await isar.users.delete(existingUser.id!); // 削除
});
```

このようにIsarでは相当少ないコードで実装できていることがわかります。ここまでがIsarの基本的な使い方です。

## Riverpodと一緒に使う
Flutterアプリの設計としてよく用いられているのがriverpodだと思います。そこで、IsarオブジェクトをStateNotifierでラップしてProviderで監視するような設計を考えてみたいと思います。ただし、Isarオブジェクトは公式ドキュメントと同じEmailオブジェクトとします。

### Isarインスタンスをシングルトンにする
IsarインスタンスはSharedPreferencesのように１つのインスタンスを共有しておけば良いと考えて、シングルトンで実装します。

```dart
class MyIsar {
  late Isar isar;

  static final MyIsar _instance = MyIsar._internal();

  factory MyIsar() {
    return _instance;
  }

  MyIsar._internal();

  Future<void> init() async {
    MyIsar._internal();

    var path = '';
    if (!kIsWeb) {
      final dir = await getApplicationSupportDirectory();
      path = dir.path;
    }
    isar = await Isar.open(
      [EmailSchema],
      directory: path,
    );
  }
}
```

### IsarオブジェクトのリストをStateNotifierでラップしてメソッドを定義
```dart
class EmailListState extends StateNotifier<List<Email>> {
  EmailListState(List<Email> emailList) : super([]);

  final Isar isar = MyIsar().instance.isar;

  Future<void> getAll() async {
    final list = isar.emails;
    final getList = await list.where().findAll();
    state = getList;
  }

  Future<void> add(Email item) async {
    final list = isar.emails;
    await isar.writeTxn(() async {
      await list.put(item);
    });
    await getAll();
  }

  Future<void> deleteAt(int id) async {
    final list = isar.emails;
    await isar.writeTxn(() async {
      await list.delete(id);
    });
    await getAll();
  }

  Future<void> deleteAll() async {
    final list = isar.emails;
    await isar.writeTxn(() async {
      await list.where().deleteAll();
    });
    await getAll();
  }
}

```

### Providerを定義
```dart
final emailListStateNotifierProvider =
StateNotifierProvider<EmailListState, List<Email>>(
      (ref) => EmailListState([]),
);
```

### main関数内でIsarを初期化しておく
```dart
void main() {
  WidgetsBinding widgetsBinding = WidgetsFlutterBinding.ensureInitialized();
  await MyIsar().init();

  runApp(const ProviderScope(
    child: MyApp(),
  ));
}
```

以上の方法でIsarオブジェクトを監視することができました。addメソッドなどを実行するとローカルDBが変更され、変更されたローカルDBを再取得し、自動的にProviderが更新されるという設計になっています。よってUI - state - ローカルDB 間に単方向のデータフローが保たれるというメリットがあります。
