---
title: "[Swift]毎週指定の時間に通知する"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [swift, ios]
published: true
---

この記事では、Swiftで開発されたアプリ「ごほうび習慣プラス」の核となるコンポーネントの一つである通知サービスについて解説します。このアプリは、ゲーミフィケーションを活用した習慣化アプリで、日々の習慣を楽しく維持するための支援をしています。

https://apps.apple.com/jp/app/%E3%81%94%E3%81%BB%E3%81%86%E3%81%B3%E7%BF%92%E6%85%A3%E3%83%97%E3%83%A9%E3%82%B9-%E3%82%B2%E3%83%BC%E3%83%9F%E3%83%95%E3%82%A3%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E7%BF%92%E6%85%A3%E5%8C%96%E3%82%A2%E3%83%97%E3%83%AA/id6474091359

「ごほうび習慣プラス」における通知機能では、ユーザーが習慣化タスクの実施を支援するために毎週特定の時間に通知を受け取る機能を提供しています。この機能は、習慣化を促進する上で極めて重要な役割を果たしており、ユーザーが定期的にタスクを行うためのリマインダーとして機能します。

## 習慣化支援の重要性

習慣化は、新しい行動を定着させるために効果的な方法です。しかし、日々の忙しさの中で新しい習慣を思い出して実行することは難しいことが多いです。ここで「ごほうび習慣プラス」の通知機能が重要な役割を果たします。

## 通知機能の役割

- リマインダーとしての役割: ユーザーが設定した特定の曜日と時間に通知が送られることで、習慣化タスクの実施を忘れずにすむようになります。

- 継続的なエンゲージメント: 定期的な通知により、ユーザーはタスクを実行するためのモチベーションを維持しやすくなります。

# 実装方法

全体のコードは以下のようになります。通知機能全体を一つのアプリケーションサービスとしてまとめました。

``` swift
import Foundation
import UserNotifications

struct NotificationService {
    static func setRepeatCalenderNotification(id: String, weekday: Weekday, title: String, body: String, hour: Int, minute: Int, completion: @escaping (_ error: Error?) -> Void) {
        let content = UNMutableNotificationContent()
        content.title = title
        content.body = body
        content.sound = UNNotificationSound.default

        var dateComponents = DateComponents()
        dateComponents.weekday = weekday.rawValue
        dateComponents.hour = hour
        dateComponents.minute = minute

        let trigger = UNCalendarNotificationTrigger(dateMatching: dateComponents, repeats: true)

        let request = UNNotificationRequest(identifier: "\(id)/\(weekday.name)", content: content, trigger: trigger)

        UNUserNotificationCenter.current().add(request) { error in
            completion(error)
        }
    }
    
    static func cancelRepeatNotification(id: String, weekday: Weekday) {
        UNUserNotificationCenter.current().removePendingNotificationRequests(withIdentifiers: ["\(id)/\(weekday.name)"])
    }
    
    static func checkNotificationAuthorization(completion: @escaping (_ isAuthorized: Bool) -> Void) {
        UNUserNotificationCenter.current().getNotificationSettings { settings in
            if settings.authorizationStatus == .authorized {
                completion(true)
            } else {
                completion(false)
            }
        }
    }
}

enum Weekday: Int {
    case sunday = 1, monday, tuesday, wednesday, thursday, friday, saturday
    
    var name: String {
        switch self {
        case .sunday:
            return "Sunday"
        case .monday:
            return "Monday"
        case .tuesday:
            return "Tuesday"
        case .wednesday:
            return "Wednesday"
        case .thursday:
            return "Thursday"
        case .friday:
            return "Friday"
        case .saturday:
            return "Saturday"
        }
    }

    var title: String {
        switch self {
        case .sunday:
            return "日曜日"
        case .monday:
            return "月曜日"
        case .tuesday:
            return "火曜日"
        case .wednesday:
            return "水曜日"
        case .thursday:
            return "木曜日"
        case .friday:
            return "金曜日"
        case .saturday:
            return "土曜日"
        }
    }
}
```

## setRepeatCalenderNotification メソッド

- 通知内容の設定:

UNMutableNotificationContent オブジェクトを使用して、通知のタイトル、本文、サウンドを設定します。

- 通知トリガーの設定:

DateComponents を使用して、通知をトリガーする曜日、時間（時と分）を設定します。

UNCalendarNotificationTrigger を使って、これらの DateComponents に基づいて通知トリガーを作成します。repeats パラメータは true に設定され、これにより通知は毎週繰り返されます。

## cancelRepeatNotification メソッド

特定のIDと曜日に関連付けられた通知をキャンセルします。

## checkNotificationAuthorization メソッド

このメソッドは、アプリが通知を送信するための権限を持っているかどうかをチェックします。ユーザーが通知を許可していない場合、アプリは通知を送信できません。

## Weekdayの実装

Weekday 列挙型は、曜日を表します。各曜日は整数と文字列の両方の値を持たせます。この整数はDateComponentsの曜日を表すidと対応させると便利です。

ここまでお読みいただいてありがとうございました。このメソッドを順番に実行することで毎日指定時間に通知するといった実装も可能になります。最後にごほうび習慣プラスをぜひ使ってみてください！

https://apps.apple.com/jp/app/%E3%81%94%E3%81%BB%E3%81%86%E3%81%B3%E7%BF%92%E6%85%A3%E3%83%97%E3%83%A9%E3%82%B9-%E3%82%B2%E3%83%BC%E3%83%9F%E3%83%95%E3%82%A3%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E7%BF%92%E6%85%A3%E5%8C%96%E3%82%A2%E3%83%97%E3%83%AA/id6474091359