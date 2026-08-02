---
title: Railsのtransaction内でメール送信を行ったらループ送信が発生した
pubDate: 2025-08-10
author: nax2
tags: ['Rails']

---
# Railsのtransaction内でメール送信を行ったらループ送信が発生した

障害対応の中でtransactionの使い方に関する学びがあったので記録します。


## transaction処理でDB操作以外は行わない

Railsのtransaction内でメール送信を行ったところ、エラー時にメールが繰り返し送信される問題が発生しました。  
ActiveRecordのtransactionではメール送信はrollbackしないため、DBの送信履歴と不整合が起きていました。

## rollbackできる処理とできない処理

### rollback可能な処理
- ActiveRecordによるDB更新(Model.create や Model.updateなど)
- ValidationErrorによるrollback
- 例外を投げるメソッドで失敗した場合  
  (save! や update!のActiveRecord::RecordInvalidなどの例外発生)

### rollback不可能な処理
- メール送信（ActionMailer）
- 外部APIコール
- DB以外のファイルの書き込み

## DB操作のみtransaction内に残し、それ以外は別処理へ移動

transactionでrollbackしない処理は、別処理として切り出しました。  
変更後の例は以下です。
```ruby
obj.each { |arg|
  begin
    DBの更新1
    DBの更新2
    DBの更新3
  rescue => e
    error出力
    next
  end

  begin
    メール送信
  rescue => e
    error出力
  end
end
```

