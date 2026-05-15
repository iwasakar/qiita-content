---
title: 【日本語】What is Pest 3.0(PHP Testing Flamework)
tags:
  - TDD
  - PHPUnit
  - Laravel
  - pest
private: false
updated_at: '2024-09-04T11:16:02+09:00'
id: 6bb033ae4f630ecf4ade
organization_url_name: dialog-inc
slide: false
ignorePublish: false
---
Laravel11からデフォルトのテストフレームワークとなったPestですが、近日バージョンアップが行われます。
まだまだPHPUnitの歴史が深く、浸透していないですが、日本最速(？)キャッチアップを目指して紐解いていきます。

## Pest 2.0との互換性
完全下位互換性があるため、簡単にバージョンアップ可能です。
この辺は昨今当たり前になってきていますね。

## 主要なアップデート
タスク管理、アーキテクチャプリセット、ミューテーションテストの3点です。
以下にまとめます。

## タスク管理
タスク管理を使用すると、担当者(assignee)、課題番号(issue)、プル リクエスト番号、メモなどをテスト上で直接追跡できます。タスクを追跡すると、テストで常に利用できる作業履歴を保持できます。
TDDが捗りそうです。
```php
test('some test 1', function () {
    // ...
})->todo(assignee: 'paulredmond', issue: 13);
 
it('returns a successful response', function () {
    $response = $this->get('/');
 
    $response->assertStatus(200);
})->todo(
    assignee:'nunomaduro',
    issue:11,
    pr:1,
    note:'Need to assert the response body.'
);
 
// Eric is a closer ☕️
test('drives always go in the fairway', function () {
    // ...
})->done(assignee: 'nunomaduro', issue: 11);
```
`#11`や`@nunomaduro`をクリックすればGithubにアクセスできます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3798970/ce700745-2321-b759-8aad-ac63e10b3ba3.png)

コマンドラインから、担当者または問題別にテストをフィルタリングできます。
```sh
$ pest --issue=11
$ pest --assignee=paulredmond --pr=1
$ pest --assignee=ericlbarnes
$ pest --issue=13
$ pest --notes # returns tests with notes
```

## アーキテクチャプリセット

アーキテクチャ プリセットは、複数のプロジェクトで同じアーキテクチャ ルールをすばやく使用し、Laravel、PHP、Strict/Relaxed PHP プリセットなどの一般的なプリセットを活用する方法です。これにより、デバッグ関数などを残すなどの操作が防止されるため、コードの一貫性が向上します。
社内の開発環境の均一化の手段の1つとして有効です。またLaravel初心者がいるPJで、PRで指摘しあう必要もなくなるので、より高度なコミュニケーションに集中できます。

これらのプリセットは次の方法で使用できます
`arch()->preset()`
```php
arch()->expect('dd')->not->toBeUsed();
```
ddを使っていないことだけをテストすると下記のように検知されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3798970/cb5e7296-dab1-5cc2-68be-0377b300bd26.png)

上記のようなテストを1つずつ書くこともできます。（既存PJや大規模PJ向け）
新規PJや中規模、小規模なPJ向けには、下記のように呼び出すだけで一括でテストできます。
```php
// die(), phpinfo()といった不要な関数の呼び出しがないことをテストします。
arch()->preset()->php();
 
// eval(), md5()といったセキュリティに問題ある関数の呼び出しがないことをテストします。
arch()->preset()->security();
 
// コントローラのメソッドを検証するためのプリセットです、
// コントローラファイルの接尾辞が `Controller` である、など。
arch()->preset()->laravel();
 
// クラスがfinalなものであることを保証するプリセットです。
arch()->preset()->strict();
 
// 厳密な型が使用されないようにプリセットされ、finalもprivateメソッドもないことをテストします。
arch()->preset()->relaxed();
```
## ミューテーションテスト
使い方は、`--mutate`オプションを付けるだけです
コードをわずかに自動改変し、テストを実施します。
この時テストが失敗すれば、テストは正しく書かれています。
もし、テストが成功してしまうと改変したコードに対してテストが行われていないことがわかります。

## 参考
動画を見てわかるように、ここまでサクサクと試せるのはPestだけではないのでしょうか？
まさにPestの素晴らしさを体現している動画だと思い、私は非常に感動しました。

https://www.youtube.com/watch?v=BNhbgcNJyAk

https://laravel-news.com/everything-we-know-about-pest-3
