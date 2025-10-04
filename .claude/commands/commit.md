# COMMIT コマンド

## 目的
現在の変更を適切な粒度に切り分けて git commit する。

## 注意事項
- `git diff -a` を実行した上で適切な commit 粒度、commit メッセージを考えた上で commit すること。
  - `git add`、`git commit` を使って自分で判断した粒度でお願いします。
- commit メッセージは1行目に英語で prefix 付きのコメント、3行目以降は日本語で変更の背景を記述する
- `🤖 Generated with [Claude Code](https://claude.ai/code)` や `Co-Authored-By: Claude <noreply@anthropic.com>` といった余計な署名はしないこと。

## commit メッセージの prefix について
- `feat:` 新機能
- `fix:` バグ修正
- `docs:` ドキュメントの変更
- `style:` フォーマットの変更
- `refactor:` 仕様に影響がないコード改善
- `perf:` パフォーマンス向上関連
- `test:` テスト関連
- `chore:` ビルド、補助ツール、ライブラリ関連
