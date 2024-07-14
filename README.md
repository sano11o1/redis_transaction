# redis_transaction

RDBMSのように行ロック、テーブルロック機能はRedisは提供しない。
代わりにキーをwatchし、watch開始~EXEC実行の間にキーが更新された場合はエラーとする。

楽観的ロックの動作確認
```
# セッションAで実行
watch foo

# セッションBで実行
set foo 789

# セッションAで実行
MULTI
  set foo 123
  set bar 456
EXEC

GET foo -> 789 #Bの変更
GET bar -> nil #456は適応されない
```

トランザクション内のfooとbarの順番を入れ替えても、トランザクションのfoo, barの両方が適応されないことを確認
```
# セッションAで実行
watch foo

# セッションBで実行
set foo 789

# セッションAで実行
MULTI
  set bar 456
  set foo 123
EXEC

GET foo -> 789 #Bの変更
GET bar -> nil #456は適応されない
```

トランザクション実行前に文法エラーが発生した場合は、MULTI内のコマンドが実行されない(ロールバックっぽい動きになる)
```
set foo 98765

MULTI
  SET foo 123456
  INCRRRRR count #文法エラー
EXEC

GET foo # 98765のまま
```

トランザクション実行時(EXEC)にエラーが発生した場合、ロールバックされない
```
set count count #インクリメントできない文字列を入れる

MULTI
  SET foo 123456
  INCR count
EXEC

GET foo # 123456が反映されている
GET count # count
```

トランザクション実行時のエラーは開発中に気づけるので、大きな問題にはならないということらしい、、
