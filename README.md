# Apexクラス実装方針解説

ヘキサゴナルアーキテクチャで実装するケースを実装を検証する。  
ヘキサゴナルアーキテクチャについて、このREADMEでは詳しく解説できないが、実装は以下の書籍に準じている。
[手を動かしてわかるクリーンアーキテクチャ　ヘキサゴナルアーキテクチャによるクリーンなアプリケーション開発](https://amzn.asia/d/dugG7Cw)

以下では、パッケージごとに実装する内容と約束事を記載する。

## classes直下のディレクトリ

- 機能単位のディレクトリを、classesディレクトリに作成(例: 取引先窓口登録 -> register-contact)して、その配下に、規則に基づいた資材を開発する。
- この機能単位とは、ドメイン単位で設計されることとし、決してオブジェクト単位で設計するものではない(取引先オブジェクトにまつわる機能だからAccount、という安易な作成はしないこと。取引先の登録という仕事であるならば、register-accountとする)
- パッケージの概念がないため、ディレクトリを作成して表現することとする
- 別機能の資材へアクセスすることは禁止する。(アクセス修飾子のほとんどはpublicで宣言せざるを得ないため、運用ベースで禁止とする)

## Adapter

解説 
- portで定義されたインターフェースを実装しており、DBやUIとのやりとりを行う。アダプタを変更することで、異なるDBやUIに対応することが容易にすることを狙う。

### in

- Controllerサフィックスを付与したクラスを実装し、このクラスはフロントからのパラメータ受け取りから、返却値の返却の責務を追う
- 複数のLWCからの呼び出しはOKだが、単一責務の原則とテスト容易性担保のため、`@AuraEnable`アノテーションを付与するpublicメソッドは1つのみとする
- フロントエンドからの入力値のフォーマットは、adapter.inパッケージにValidatorを実装して行う。そのクラス名にはValidatorサフィックスを付与する
- Validatorは、Apexクラスが正しく処理できるようにするためのもので、Salesforce標準機能の入力規則とは別物と考える

### out

- この層は、永続化層へのアクセスと外部APIへのアクセスを責務とする
- 実装クラス命名は、データ操作ロジック以外の意味合いを持たない場合、Implサフィックスを付与する
- mockの場合、Mockサフィックスを付与し、クラス名は~Mockとなる
- mockには`@IsTest`アノテーションをクラスに付与し、mockに対するテストクラス実装は不要とする

## Application
解説
- port: ビジネスロジックが外部システムと通信するためのインターフェース。具体的な処理をビジネスロジックから切り離し、どのように外部とやり取りするかを定義し、ビジネスロジックがどのような技術に依存するかを知らずに済むようになる。
  - in: ビジネスロジックとのやりとりを担うクラスのInterfaceを定義
  - out: SOQLや外部システムとの通信を担うクラスのInterfaceを定義
-

### port.in

- UseCaseインターフェースを提供する
- 上記以外は何も提供しないこと

### port.out

- 永続化層インターフェースを提供する
- 上記以外は何も提供しないこと

### domain.model

- ドメインオブジェクト、値オブジェクトの実装をする。
- クラスの責務をクラス名から読み取れるよう、以下の規則で実装する
  - 値オブジェクトはVoサフィックスを付与する
  - Entityは、Entityサフィックスを付与する
  - UseCaseへの入力モデルは、inputサフィックスを付与する

### service

- この層は、UseCaseInterfaceを実装したクラスを実装する。この層の責務は以下とする
  - 永続化層の呼び出し
  - ドメインオブジェクトのビジネスロジックの呼び出し
  - ビジネスロジック実行で得た生成値を、呼び出し元に返却
- UseCaseインターフェースは、application.port.inパッケージに宣言
- 実装クラスは、application.domain.serviceパッケージ実装
- フィールドには永続化層のInterfaceを持たせる。DIは、injectorパッケージに実装したクラスで依存性注入を行う。それ以外は禁止する
- 原則再利用を禁止する

## injector

- Java(SpringBoot)のように、DIコンテナでインスタンス生成ができないため、独自にDIする仕組みが必要なため実装する
- UseCaseインターフェースを実装するクラスのインスタンスをビルドすることのみを責務とする
- コンストラクタはstaticメソッドにすることを許容するが、他の機能や、`Controller`以外からアクセスすることを禁止する

## その他

- ディレクトリはケバブケースとする

## 感じた課題点

- 書籍では、UseCaseの入力モデルでは入力値チェックを行わなくて良いとする（UseCaseInterface実装クラスが汚れるため）が、これが正しく機能しないと、INSERT・UPDATE時に入力規則に引っかかるため、入力規則に基づいた実装は必要になる。
- このアーキテクチャの良さは、実際に開発をやってもらわないと伝わりづらい。
  - 参考コードベースだと、普通に開発すると10行程度1クラスで済む資材を合計10クラス作ることになるため、学習コストは高くなってしまう
  - 初学者やスピード重視開発者がこれだけを見るとめんどくさいと思われても仕方がない
- Idからオブジェクトをクエリするだけなら、このアーキテクチャはSFのメモリやCPUリソース的に勿体無い実装と言える。その場合、LWCのLightningデータサービスを使用した方がいい。その他、ビジネスロジックをもたないクエリとの棲み分けを考える必要はありそう。
- Salesforceそのものが、データ駆動な製品といえることを再確認。sObjectを大切にしており、カスタムApexクラスをLWCに返却する実装にすると、オブジェクト権限が適用できず、相性が悪いので適材適所とする必要あり。
- パッケージプライベートにできないため、public修飾子を持つメソッドやクラスに、他の機能単位のクラスが実質アクセス可能。運用ベースで防ぐしかない（これはApexEnterprisePatternsも同様）。

```cls
  // register-accountを例に、アーキテクチャなし版の開発と比較する。
  // 今回実装分。レイヤーに基づいて必要なインスタンスをビルドして処理させる
  @AuraEnabled
  public static void register(Id accountId, String contactName) {
    RegisterContactUseCase useCase = RegisterContactInjector.RegisterContactService();
    try {
      useCase.exec(new ContactVo(accountId, contactName));
    } catch (DmlException e) {
      throw e;
    }
  }
  
  ↓
  
  // SOQLで実装すると、これで済む。SELECTだけならなおさら短くすむ。LWCならLightningDataServiceでよい。
  @AuraEnabled
  public static void register(Id accountId, String contactName) {
    try {
      Account act = [SELECT Id FROM Account WHERE Id = :accountId()];
      if (act == null) throw new HandledException('No Account Is Exist.');

      insert new Contact(AccountId = :accountId, LastName = contactName);
    } catch (DmlException e) {
      throw e;
    }
  }
```
