# Infrastructure層（インフラストラクチャ層）詳細ドキュメント

## 概要

Infrastructure層は、レイヤードアーキテクチャの最下位層で、外部システムとの連携や技術的な実装詳細を担当します。この層は、Domain層で定義されたインターフェースを具体的に実装し、データベース、ファイルシステム、外部APIなどの技術的な詳細を隠蔽します。

## レイヤードアーキテクチャにおけるInfrastructure層の重要な役割

### Infrastructure層の位置づけ

```
┌─────────────────────────────────────────────────────────────┐
│                   Presentation Layer                        │
│                           ↓                                 │
│                   Application Layer                         │
│                           ↓                                 │
│                     Domain Layer                            │
│                  (インターフェース定義)                           │
│                           ↑                                 │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                Infrastructure Layer                     ││
│  │                (技術的実装詳細)                            ││
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        ││
│  │  │  Mappers    │ │Query Builder│ │Repositories │        ││
│  │  │             │ │             │ │             │        ││
│  │  └─────────────┘ └─────────────┘ └─────────────┘        ││
│  └─────────────────────────────────────────────────────────┘│
│                           ↓                                 │
│              外部システム（DB、API、ファイル等）                     │
└─────────────────────────────────────────────────────────────┘
```

### 1. 依存性逆転の原則の実現
- **インターフェース実装**: Domain層で定義された契約の具体的実装
- **技術的詳細の隠蔽**: 上位層から技術的複雑性を隠す
- **変更の局所化**: 技術スタック変更の影響を限定

### 2. 外部システムとの統合
- **データベースアクセス**: Prismaを使用したデータ永続化
- **データ変換**: 外部形式とドメインオブジェクトの相互変換
- **クエリ最適化**: 効率的なデータ取得の実現

### 3. 技術的複雑性の管理
- **複雑なクエリの構築**: 動的で可読性の高いクエリ生成
- **データマッピング**: 異なるデータ形式間の変換
- **パフォーマンス最適化**: 効率的なデータアクセス

---

## Infrastructure層の詳細

### Mappers（マッパー）

#### 1. service.mapper.ts
**役割**: データベースモデルとドメインエンティティ間の変換

**主要な責務**:
- **データ変換**: PrismaモデルからServiceEntityへの変換
- **型安全性**: Value Objectの適切な生成
- **データ正規化**: 外部データの内部形式への変換

**レイヤードアーキテクチャにおける重要性**:
- **境界の明確化**: 外部データ形式とドメインモデルの分離
- **データ整合性**: 変換時のバリデーションと正規化
- **技術的詳細の隠蔽**: データベーススキーマの変更から上位層を保護

**主要メソッド**:
```typescript
toDomain()              // PrismaモデルをServiceEntityに変換
toDomainList()          // 複数データの一括変換
convertTagInfo()        // JSON形式のタグ情報を正規化
parseDateTime()         // 文字列日時をDateオブジェクトに変換
parseDate()             // 文字列日付をDateオブジェクトに変換
generateThumbnailUrl()  // S3パスから完全URLを生成
```

**なぜ必要か**:
- **データ形式の違いを吸収**: データベースとドメインモデルの形式差異を解決
- **型安全性の確保**: Value Objectの適切な生成でランタイムエラーを防止
- **データ品質の保証**: 変換時の検証とクリーニング
- **保守性の向上**: データ変換ロジックの一元化

**いつ使用するか**:
- データベースからデータを取得した時
- ドメインエンティティをデータベースに保存する時
- 外部APIからのデータを内部形式に変換する時

### Query Builders（クエリビルダー）

#### 1. service-query.builder.ts
**役割**: 動的で複雑なデータベースクエリの構築

**主要な責務**:
- **動的クエリ生成**: 条件に応じたWHERE句の構築
- **可読性の向上**: 複雑な条件を分かりやすく表現
- **再利用性**: 共通的な検索パターンの抽象化

**レイヤードアーキテクチャにおける重要性**:
- **複雑性の管理**: 複雑なクエリロジックの体系化
- **保守性の向上**: 条件変更時の影響局所化
- **パフォーマンス最適化**: 効率的なクエリの生成

**主要メソッド**:
```typescript
excludeDeleted()        // 削除フラグによる除外
filterByCompany()       // 企業IDフィルタ
filterByStatus()        // ステータスフィルタ
filterByCategory()      // カテゴリフィルタ
filterByKeyword()       // キーワード検索
filterByDateRange()     // 日付範囲フィルタ
applySearchCriteria()   // 検索条件の一括適用
build()                 // 最終的なWHERE句の構築
```

**ビルダーパターンの利点**:
```typescript
// 可読性の高いクエリ構築
new ServiceQueryBuilder()
  .filterByCompany('EC00000001')
  .filterByStatus('active')
  .filterByKeyword('サービス名')
  .build();
```

**なぜ必要か**:
- **動的クエリの管理**: 条件の組み合わせに応じた柔軟なクエリ生成
- **可読性の向上**: 複雑な条件を直感的に表現
- **メンテナンス性**: 条件追加・変更時の影響を限定
- **再利用性**: 共通的な検索パターンの抽象化

**いつ使用するか**:
- 複数の検索条件を組み合わせる時
- 条件によってクエリを動的に変更する時
- 複雑なWHERE句を構築する時

### Repositories（リポジトリ実装）

#### 1. prisma-service.repository.ts
**役割**: Domain層のRepositoryインターフェースの具体的実装

**主要な責務**:
- **データアクセス**: Prismaを使用したデータベース操作
- **クエリ実行**: 検索条件に基づくデータ取得
- **パフォーマンス最適化**: 効率的なクエリとページネーション

**レイヤードアーキテクチャにおける重要性**:
- **依存性逆転の実現**: Domain層のインターフェースを実装
- **技術的詳細の実装**: 具体的なデータアクセス方法の提供
- **変更の局所化**: データアクセス技術変更時の影響限定

**実装されるメソッド**:
```typescript
findByCriteria()        // 条件検索（ページネーション付き）
findById()              // ID検索
findByServiceId()       // サービスID検索
findByCompanyId()       // 企業別検索
findPublicServices()    // 公開サービス検索
countByCriteria()       // 件数取得
exists()                // 存在確認
```

**内部実装の詳細**:
- **WHERE句構築**: ServiceQueryBuilderを使用
- **ORDER BY句構築**: ソート条件の適切な変換
- **INCLUDE句構築**: 関連データの効率的な取得
- **ページネーション**: skip/takeによる効率的な分割取得

**なぜ必要か**:
- **インターフェース実装**: Domain層の契約を具体的に実現
- **データアクセスの最適化**: 効率的なクエリとキャッシュ戦略
- **技術的詳細の管理**: Prisma固有の実装詳細を隠蔽
- **エラーハンドリング**: データベースエラーの適切な処理

**いつ使用するか**:
- Domain層からデータアクセスが要求された時
- 検索、取得、更新、削除の各操作時
- ページネーションが必要な時

---

## Infrastructure層の設計パターン

### 1. Mapper Pattern
**目的**: 異なるデータ表現間の変換

**利点**:
- **境界の明確化**: 外部形式と内部形式の分離
- **変換ロジックの集約**: データ変換の一元管理
- **型安全性**: 適切な型変換の保証

**実装例**:
```typescript
// データベースモデル → ドメインエンティティ
static toDomain(prismaService: PrismaServiceWithRelations): ServiceEntity {
  return new ServiceEntity(
    prismaService.productId,
    prismaService.entryCompanyId,
    ServiceId.create(prismaService.serviceId), // Value Object生成
    ServiceName.create(prismaService.serviceName),
    // ... その他のプロパティ
  );
}
```

### 2. Builder Pattern
**目的**: 複雑なオブジェクトの段階的構築

**利点**:
- **可読性**: 直感的なメソッドチェーン
- **柔軟性**: 条件の動的な組み合わせ
- **再利用性**: 共通パターンの抽象化

**実装例**:
```typescript
// 段階的なクエリ構築
new ServiceQueryBuilder()
  .filterByCompany(companyId)
  .filterByStatus('active')
  .filterByKeyword(searchTerm)
  .build();
```

### 3. Repository Pattern
**目的**: データアクセスの抽象化

**利点**:
- **依存性逆転**: インターフェースベースの設計
- **テスタビリティ**: モックによる単体テスト
- **技術的柔軟性**: データストア変更への対応

**実装例**:
```typescript
@Injectable()
export class PrismaServiceRepository implements ServiceRepositoryInterface {
  // Domain層のインターフェースを実装
  async findByCriteria(criteria: ServiceSearchCriteria): Promise<PaginatedResult<ServiceEntity>> {
    // Prisma固有の実装詳細
  }
}
```

---

## Infrastructure層の技術的特徴

### 1. Prisma統合
- **型安全性**: TypeScriptとの完全な統合
- **クエリ最適化**: 自動的なクエリ最適化
- **マイグレーション**: スキーマ変更の管理

### 2. エラーハンドリング
- **データベースエラー**: 接続エラー、制約違反の処理
- **ログ出力**: 構造化されたエラーログ
- **リトライ機能**: 一時的な障害への対応

### 3. パフォーマンス最適化
- **効率的なクエリ**: 必要なデータのみの取得
- **ページネーション**: 大量データの効率的な分割
- **関連データ取得**: N+1問題の回避

---

## 使用場面とベストプラクティス

### Mappers
- **使用場面**: データ形式変換、外部システム連携
- **ベストプラクティス**:
  - 変換ロジックの一元化
  - エラーハンドリングの実装
  - Value Objectの適切な生成

### Query Builders
- **使用場面**: 動的クエリ生成、複雑な検索条件
- **ベストプラクティス**:
  - メソッドチェーンの活用
  - 条件の組み合わせ可能性
  - 可読性の重視

### Repository Implementations
- **使用場面**: データアクセス、CRUD操作
- **ベストプラクティス**:
  - インターフェース準拠
  - エラーハンドリング
  - パフォーマンス最適化

---

## ポイント

### 1. なぜInfrastructure層が必要なのか？
**問題**: ビジネスロジックとデータベースが密結合
```typescript
// 悪い例：ビジネスロジックにSQL文が混在
class ServiceService {
  async getServices() {
    const sql = "SELECT * FROM services WHERE status = 'active'";
    // ビジネスロジックとデータアクセスが混在
  }
}
```

**解決**: Infrastructure層による分離
```typescript
// 良い例：責務の分離
class ServiceApplicationService {
  constructor(private repository: ServiceRepositoryInterface) {}
  
  async getServices() {
    // ビジネスロジックに集中
    return this.repository.findPublicServices(criteria);
  }
}
```

### 2. Mapperの重要性
**問題**: データベース構造の変更がビジネスロジックに影響
```typescript
// データベースのカラム名変更時
// 悪い例：全てのビジネスロジックを修正が必要
service.service_name // カラム名変更でエラー
```

**解決**: Mapperによる変換
```typescript
// 良い例：Mapperで変換を一元化
ServiceMapper.toDomain(dbRecord) // 変更はMapperのみ
```

### 3. Query Builderの利点
**問題**: 複雑な条件分岐でコードが読みにくい
```typescript
// 悪い例：条件分岐が複雑
let where = {};
if (companyId) where.companyId = companyId;
if (status) where.status = status;
// 可読性が低い
```

**解決**: Builder Patternで可読性向上
```typescript
// 良い例：直感的で読みやすい
new ServiceQueryBuilder()
  .filterByCompany(companyId)
  .filterByStatus(status)
  .build();
```

Infrastructure層は、レイヤードアーキテクチャにおいて技術的な複雑性を管理し、上位層をデータアクセスの詳細から保護する重要な役割を果たします。適切な設計により、保守性、テスタビリティ、パフォーマンスを同時に実現できます。
