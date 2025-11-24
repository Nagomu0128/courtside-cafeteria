# 11. データベーススキーマ (Firestore)

## 原則：NoSQLドキュメント設計

Firestoreを使用するため、リレーショナルモデルではなく、読み取り頻度と書き込み頻度を考慮したドキュメント指向モデルを採用します。

### コレクション構造

```
/ (root)
├── users (collection)
│   └── {userId} (document)
│       ├── createdAt: Timestamp
│       ├── lastAccessedAt: Timestamp
│       ├── isAnonymous: boolean
│       ├── profile (map field)
│       │   ├── department: string
│       │   ├── name: string
│       │   ├── gender: string
│       │   └── ageGroup: string
│       │
│       └── orders (sub-collection) ← ユーザー注文一覧用（高速取得）
│           └── {orderId} (document)
│               ├── orderNumber: string
│               ├── menuId: string
│               ├── menuName: string (非正規化)
│               ├── status: string
│               └── orderedAt: Timestamp
│
├── menus (collection)
│   └── {menuId} (document)
│       ├── name: string
│       ├── description: string
│       ├── imageUrl: string (Firebase Storage URL)
│       ├── price: number
│       ├── availableDate: Timestamp
│       ├── orderDeadline: Timestamp
│       ├── maxQuantity: number
│       ├── status: string ('ACTIVE' | 'CLOSED' | 'DRAFT')
│       └── options (array of maps)
│           ├── name: string
│           ├── type: string
│           └── items: array
│
├── orders (collection) ← 管理者用・全注文管理（正規化データ）
│   └── {orderId} (document)
│       ├── orderNumber: string (ユーザー表示用)
│       ├── userId: string (Reference)
│       ├── menuId: string (Reference)
│       ├── status: string ('CONFIRMED' | 'CANCELLED')
│       ├── orderedAt: Timestamp
│       ├── userInfo: map
│       │   ├── department: string
│       │   ├── name: string
│       │   └── ...
│       └── selectedOptions: array of maps
│
├── counters (collection) - 注文番号採番用
│   └── orderSequence_{YYYYMMDD} (document)
│       └── current: number
│
└── system (collection)
    └── optionTemplates (document)
        └── templates: array
```

### インデックス設計

Firestoreの複合インデックス設定（`firestore.indexes.json`）

```json
{
  "indexes": [
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "menuId", "order": "ASCENDING" },
        { "fieldPath": "orderedAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "orderedAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "menus",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "availableDate", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" }
      ]
    }
  ]
}
```

### セキュリティルール (firestore.rules)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ユーザー関数
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return request.auth.uid == userId;
    }

    function isAdmin() {
      return request.auth.token.admin == true;
    }

    // Users: 本人のみ読み書き可、管理者は読み取り可
    match /users/{userId} {
      allow read, write: if isOwner(userId);
      allow read: if isAdmin();

      // Users > Orders サブコレクション: 本人のみアクセス可
      match /orders/{orderId} {
        allow read: if isOwner(userId);
        allow write: if false; // サーバーサイドからのみ書き込み
      }
    }

    // Menus: 誰でも読み取り可、管理者は書き込み可
    match /menus/{menuId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // Orders: 本人は作成・読み取り・更新可、管理者は全権限
    match /orders/{orderId} {
      allow read: if isOwner(resource.data.userId) || isAdmin();
      allow create: if isAuthenticated() && request.resource.data.userId == request.auth.uid;
      allow update: if (isOwner(resource.data.userId) && resource.data.status != 'CANCELLED') || isAdmin();
    }

    // Counters: 認証済みユーザーのみインクリメント可（トランザクション必須）
    match /counters/{counterId} {
      allow read, write: if isAuthenticated();
    }
  }
}
```

### データ同期方針

`/orders` と `/users/{userId}/orders` の二重書き込みについて：

| コレクション                       | 用途                         | データ内容                     |
| ---------------------------------- | ---------------------------- | ------------------------------ |
| `/orders/{orderId}`                | 管理者用（全注文管理）       | 完全な注文データ               |
| `/users/{userId}/orders/{orderId}` | ユーザー用（自分の注文一覧） | 表示用に非正規化した軽量データ |

**書き込みフロー：**

1. 注文作成時、サーバーサイドで両方に書き込む（バッチ書き込み）
2. `/users/{userId}/orders` は読み取り専用（クライアントからの書き込み禁止）
3. ステータス更新時も両方を更新

```typescript
// 例: バッチ書き込み
const batch = writeBatch(db);
batch.set(doc(db, "orders", orderId), fullOrderData);
batch.set(doc(db, "users", userId, "orders", orderId), summaryData);
await batch.commit();
```
