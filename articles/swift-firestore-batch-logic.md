---
title: "[Swift] Firestoreのバッチ処理でドキュメントを一気に更新"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Swift, Firestore, Firebase]
published: true
---
こんにちは！本日は、私が先日開発したアプリ「ごほうび習慣プラス」の開発時に必要になったFirestoreのバッチ処理による一斉更新について解説します。
このアプリは、日々の習慣を報酬と結びつけることで、ユーザーがモチベーションを持続できるように設計されています。アプリはこちらのリンクからご覧いただけます

https://apps.apple.com/jp/app/id6474091359

「ごほうび習慣プラス」では習慣化をゲーム化するというコンセプトを持っています。このアプローチにより、習慣を守ることが、単なる義務ではなく、楽しい挑戦となります。

このアプリ内のデータはFirebase Firestoreを使用しています。アプリのアップデートで、ドキュメントに新たなフィールドが必要になりました。よってデータベース上に存在するすべてのドキュメントに対して一斉にフォールドを追加する必要が出てきました。
大量のドキュメントを効率的かつ安全に更新するには、適切なアプローチが必要です。本記事では、Swiftを使用してFirestoreのバッチ処理機能を活用し、複数のドキュメントを一括で更新する方法を解説します。

## トランザクションとバッチ処理の違い

Firestoreには、データを更新する二つの主な方法があります: トランザクションとバッチ処理です。トランザクションは、複数の操作を一連の処理として扱い、途中でエラーが発生した場合には全ての操作をロールバックします。
一方、バッチ処理では複数の更新操作を一つの原子的な処理として実行し、一度にすべての更新を行います。大量のデータを効率的に更新する場合、バッチ処理が最適な選択肢となります。

Firestoreのバッチ処理は、複数の書き込み操作（追加、更新、削除）を一つの原子的な操作として扱います。これにより、部分的な更新を防ぎ、データの整合性を保ちながら複数のドキュメントを効率的に更新することができます。

## Swiftによるバッチ処理の実装

以下のSwiftコードは、Firestoreの特定のコレクション内のすべてのドキュメントに新しいフィールドを追加する方法を示しています。

```swift
import FirebaseFirestore

struct FirestoreBatchUpdater {
    let db = Firestore.firestore()

    func updateDocuments() {
        let collectionRef = db.collection("yourCollectionName")
        let batch = db.batch()

        collectionRef.getDocuments { (querySnapshot, _) in
            guard let documents = querySnapshot?.documents else { return }

            for document in documents {
                let docRef = collectionRef.document(document.documentID)
                batch.updateData(["newField": "newValue"], forDocument: docRef)
            }

            batch.commit { err in
                if let err = err {
                    print("Error committing batch: \(err)")
                } else {
                    print("Batch committed successfully.")
                }
            }
        }
    }
}
```

## コード例の解説

バッチを初期化して、一度に行いたい更新内容をすべてバッチに紐づけます。今回はupdateDateを使ってすべてのドキュメントに値を追加しています。

その後バッチをコミットすることにより、すべての更新を行います。もし更新中に何処かでエラーが発生したらドキュメントは安全にロールバックされ、Swift上にエラーが返ってくるという挙動になります。

ここまでお読みいただいてありがとうございました。最後にごほうび習慣プラスをぜひ使ってみてください！

https://apps.apple.com/jp/app/id6474091359