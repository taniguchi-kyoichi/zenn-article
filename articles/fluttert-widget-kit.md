---
title: "FlutterからiOSのWidgetKitを使う"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Flutter, swiftUI, WidgetKit, AppGroup, UserDefault]
published: true
---
こんにちは。Flutterで個人開発をしているKyoichiです。
私は１ヶ月に使用できる目標金額から逆算してお金を管理する家計簿アプリ「Monthsave」を2022年2月にリリースしました。
https://apps.apple.com/jp/app/%EF%BC%91%E3%83%B6%E6%9C%88%E3%81%AE%E4%BA%88%E7%AE%97%E3%82%92%E6%B1%BA%E3%82%81%E3%81%A6%E7%AE%A1%E7%90%86-monthsave/id1609449862?ign-itscg=30200&ign-itsct=apps_box_link
このFlutterアプリでiOSのウィジェット機能を使用することになりました。しかしFlutterにはウィジェット機能を提供するパッケージがありません。よってswiftのコードを呼び出してこの機能を実装する必要があり、少々詰まりました。そこで、コピペ改変でこの機能を実装できるように記事を書きました。

※ この記事は2022/2/19にQiitaで書いた記事を修正したものである。

# 実装方法
1. [Flutterのshared_preferencesで表示したい値を保存](#flutterのsharedpreferencesで表示したい値を保存)
2. [FlutterのMethodCannelでswiftのコードを呼び出す](#flutterのmethodcannelでswiftのコードを呼び出す)
3. [swiftで呼び出されたメソッドをcallする](#swiftで呼び出されたメソッドをcallする)
4. [AppGroupを用いてWidgetKitを使用する](#appgroupを用いてwidgetkitを使用する)
5. [UserDefaultsをiOSの形式で保存し直し、Widgetを更新する](#userdefaultsをiosの形式で保存し直しwidgetを更新する)
6. [WidgetKit側でUserDefaultsを参照し表示](#widgetkit側でuserdefaultを参照し表示)

# Flutterのshared_preferencesで表示したい値を保存
iOSのウィジェットで表示したいデータは一旦shared_preferencesに保存する必要があります。Flutterのテンプレートであるカウンターアプリのカウントをウィジェットで表示することを考えると、まず以下のように保存します。
```dart: main.dart
int _counter = 0;

final SharedPreferences _prefs = await SharedPreferences.getInstance();
    _prefs.setInt('count', _counter);
```

# FlutterのMethodCannelでswiftのコードを呼び出す
次に、ウィジェットを初期化するために、swiftのコードをMethodChannelを用いてFlutterから呼び出します。アプリ始動時やデータ更新時にこちらのメソッドを呼ぶようにします。
```dart
static const methodChannel = MethodChannel('[your app name]/sample');
try {
      final int result = await methodChannel.invokeMethod('setDataForWidgetKit');
      print('SET setUserDefaultsForAppGroup: $result');
    } on PlatformException catch (e) {
      print('ERROR setUserDefaultsData: ${e.message}');
    }
```
# swiftで呼び出されたメソッドをcallする
Xcodeを開いて、AppDelegate内のApplication関数内に次のコードを追加します。
```swift
let controller : FlutterViewController = window?.rootViewController as! FlutterViewController
        let channel = FlutterMethodChannel(name: "[your app name]/sample",
                                    binaryMessenger: controller.binaryMessenger)
        channel.setMethodCallHandler({
            (call: FlutterMethodCall, result: @escaping FlutterResult) -> Void in

            if call.method == "setDataForWidgetKit"  {
                self.setUserDefaultsForAppGroup(result: result)

            }

            result(FlutterMethodNotImplemented)
            return
        })
```
先ほど作成した、[your app name]/sampleという名前のMethodCannelの、setDataForWidgetKit関数がFlutterからinvokeされたら、swiftのsetUserDefaultsForAppGroup関数を呼び出すという意味です。

# AppGroupを用いてWidgetKitを使用する
swiftでも同じであるが、WidgetKitなどのモジュールはAppExtensionsから選択して使用する。これらの機能は別のアプリと連携するときと同様に、同じAppGroupに所属させる必要がある。
Xcode上のTARGETのRunner->Signing & Capabilities -> + CapabilityからApp Groupを追加する。
グループの名前は名前は慣例的にgroup.[bundle id]とする。
次に、File -> New -> TargetからWidget Extensionを追加する。

# UserDefaultsをiOSの形式で保存し直し、Widgetを更新する
先ほど呼び出したsetUserDefaultsForAppGroup関数を実装します。
Flutterではshared_preferencesに"count"というkeyで値を保存しました。swiftコード内でこの値を参照するときはkeyが"flutter.count"となります。まずこの値をswiftで取得します。
AppGroup内で共有するデータはUserDefault形式で保存する必要があるため、この値を再度保存します。最後に以下のコードを実行してウィジェットを更新します。
```swift
if #available(iOS 14.0, *) {
            WidgetCenter.shared.reloadAllTimelines()
        }
```
なお、iOS14.0以上でしかウィジェットは使用できないことを考慮しています。
一連のコードは以下のようになります。
```swift
private func setUserDefaultsForAppGroup(result: FlutterResult) {

        guard let userDefaults = UserDefaults(suiteName: "group.[your bundle id]") else {
            return result(FlutterError(code: "UNAVAILABLE",
                                       message: "setUserDefaultsForAppGroup Failed",
                                       details: nil))
        }

        let defaults = UserDefaults.standard
        let count = defaults.value(forKey: "flutter.count") as? Int

        userDefaults.setValue(count, forKey: "count")

        if #available(iOS 14.0, *) {
            WidgetCenter.shared.reloadAllTimelines()
        }
        result(true)

    }
```
# WidgetKit側でUserDefaultを参照し表示
最後にWidgetKitのテンプレートを修正して、値を表示させれば完成です。なお、WidgetKitの仕様などは今回の記事の主題ではないため省略します。
まずSimpleEntry内にプロパティを追加します。
```swift
let count: Int
```
次に、getSnapshot関数内に以下のコードを追加してUserDefaultからデータを参照します。
```swift
let userDefaults = UserDefaults(suiteName: "group.[your bundle id]")
let countData = userDefaults?.value(forKey: "count") as? Int
```
最後にmy_widgetEntryViewのbodyの値を次のように変更します。
```swift
Text(String(entry.count))
```
# つまずいたところ
- swiftの機能であるAppGroupやUserDefaultsの仕組みをある程度理解しなければならない
- MethodChannelで呼び出したswiftのコードの実行時にエラーやログがまともに出ないため、手探りでミスを見つけなければならない。（そんなことないかも）
- WidgetKitのUIを作るとき、結局ある程度はswiftUIを理解しなければならない
- MethodChannelでは、必ずresultを返さないと例外が発生する（この仕様に気づくまでに時間がかかった）
# 参考資料
https://zenn.dev/mcz9mm/articles/ca58e5b7f9bc96ad552f

https://hondakenya.work/flutter-widgetkit/