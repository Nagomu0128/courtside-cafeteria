# 6. インフラストラクチャ層の実装例 (Firebase)

## 原則：Firebase Client SDK を使用し、セキュリティルールで保護

### SDK使用方針
- **Repository実装**: Firebase Client SDK を使用（サーバーサイドでも動作可能）
- **Admin SDKは使用しない**: セキュリティルールで適切に保護すればClient SDKで十分
- **Admin SDKが必要な場合**: `specs/07-security.md` 参照（認証の特権操作のみ）

```typescript
import { ResultAsync, ok, err } from "neverthrow";
import { getFirestore, doc, setDoc, getDoc, collection, query, where, limit, getDocs, runTransaction, Timestamp } from "firebase/firestore";
import { DomainError } from "@/src/domain/errors/DomainError";

// infrastructure/repositories/FirestoreOrderRepository.ts
export class FirestoreOrderRepository implements IOrderRepository {
  private db = getFirestore();

  save(order: Order): ResultAsync<Order, DomainError> {
    const orderRef = doc(this.db, "orders", order.id);
    return ResultAsync.fromPromise(
      setDoc(orderRef, {
        id: order.id,
        orderNumber: order.orderNumber,
        userId: order.userId,
        menuId: order.menuId,
        userInfo: order.userInfo,
        selectedOptions: order.selectedOptions,
        status: order.status,
        orderedAt: Timestamp.fromDate(order.orderedAt),
        modifiedAt: order.modifiedAt ? Timestamp.fromDate(order.modifiedAt) : null,
        cancelledAt: order.cancelledAt ? Timestamp.fromDate(order.cancelledAt) : null,
      }),
      (error) => this.handleError(error)
    ).map(() => order);
  }

  findById(id: string): ResultAsync<Order | null, DomainError> {
    const orderRef = doc(this.db, "orders", id);
    return ResultAsync.fromPromise(
      getDoc(orderRef),
      (error) => this.handleError(error)
    ).map((docSnap) => {
      if (!docSnap.exists()) return null;
      return this.toDomainOrder(docSnap.data());
    });
  }

  findByUserIdAndMenuId(
    userId: string,
    menuId: string
  ): ResultAsync<Order | null, DomainError> {
    const ordersRef = collection(this.db, "orders");
    const q = query(
      ordersRef,
      where("userId", "==", userId),
      where("menuId", "==", menuId),
      where("status", "!=", "CANCELLED"),
      limit(1)
    );
    return ResultAsync.fromPromise(
      getDocs(q),
      (error) => this.handleError(error)
    ).map((snapshot) => {
      if (snapshot.empty) return null;
      return this.toDomainOrder(snapshot.docs[0].data());
    });
  }

  // トランザクションを使用したシーケンス採番
  getNextSequence(date: Date): ResultAsync<number, DomainError> {
    const dateStr = format(date, "yyyyMMdd");
    const counterRef = doc(this.db, "counters", `orderSequence_${dateStr}`);

    return ResultAsync.fromPromise(
      runTransaction(this.db, async (transaction) => {
        const docSnap = await transaction.get(counterRef);
        let newCount = 1;
        if (docSnap.exists()) {
          newCount = docSnap.data()!.current + 1;
        }
        transaction.set(counterRef, { current: newCount }, { merge: true });
        return newCount;
      }),
      (error) => this.handleError(error)
    );
  }

  private handleError(error: unknown): DomainError {
    console.error("Firestore Error:", error);
    return DomainError.internal("データベースエラーが発生しました");
  }

  private toDomainOrder(data: Record<string, unknown>): Order {
    return {
      id: data.id as string,
      userId: data.userId as string,
      menuId: data.menuId as string,
      orderNumber: data.orderNumber as string,
      userInfo: data.userInfo as OrderUserInfo,
      selectedOptions: (data.selectedOptions as SelectedOption[]) || [],
      status: data.status as OrderStatus,
      orderedAt: (data.orderedAt as Timestamp).toDate(),
      modifiedAt: (data.modifiedAt as Timestamp | null)?.toDate(),
      cancelledAt: (data.cancelledAt as Timestamp | null)?.toDate(),
    };
  }
}
```

### Firebase Storage (画像保存)

```typescript
import { getStorage, ref, uploadBytes, getDownloadURL } from "firebase/storage";

export class FirebaseStorageService {
  private storage = getStorage();

  async uploadMenuImage(file: Blob, fileName: string): Promise<string> {
    const storageRef = ref(this.storage, `menus/${fileName}`);

    await uploadBytes(storageRef, file, {
      contentType: 'image/jpeg'
    });

    // ダウンロードURLを取得（セキュリティルールで保護）
    const url = await getDownloadURL(storageRef);
    return url;
  }
}
```
