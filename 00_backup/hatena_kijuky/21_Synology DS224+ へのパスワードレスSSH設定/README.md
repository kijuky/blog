https://kijuky.hatenablog.com/entry/2026/02/20/202656

---

# やろうとしたこと / やりたかったこと

- Mac から Synology DS224+ に SSH 接続する際、パスワードなし（公開鍵認証）でログインできるようにしたい
- `~/.ssh/config` に設定を追加して、簡単に接続できるようにしたい

# 発生した不具合・課題・問題

1. `ssh-copy-id` を使ったが、接続するとまだパスワードを要求される
2. `ssh -i ~/.ssh/id_synology -p 22` で接続しても鍵が拒否される（publickey rejected）
3. Synology の OpenSSH バージョン確認後も ED25519 鍵はサポートされているのに認証が通らない
4. `chown` で権限を変更しようとしたが、グループ名の指定でエラー発生

# 不具合の解消方法・解決策
1. Synology 側のホームディレクトリと `.ssh` / `authorized_keys` の権限を適切に設定

    ```console
    chmod 755 /volume1/homes/kizuki-yasue-admin
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```

2. 公開鍵の内容が1行で正しく `~/.ssh/authorized_keys` に登録されていることを確認
3. DSM GUI で SSH を有効化
4. Mac 側の `~/.ssh/config` に `IdentityFile [秘密鍵のパス]` と `IdentitiesOnly yes` を設定

# なぜそれで解決できたのか / 理論的な説明

- SSH の公開鍵認証は クライアントが秘密鍵で署名 → サーバーが `authorized_keys` の公開鍵で検証 という仕組み
- Synology DSM はセキュリティ上、ホームディレクトリのパーミッションが 755 以上でない場合、公開鍵認証を拒否
- `.ssh` と `authorized_keys` のパーミッションは Linux 標準通り厳しくする必要がある（700/600）
- 公開鍵が正しく1行で登録されていることで、サーバーが送られた署名を検証可能になり、鍵認証が通る
- GUIでの鍵登録は、admin/root でのログイン時に DSM が正しい場所に鍵を配置するため必須

# 参考

https://kb.synology.com/ja-jp/DSM/tutorial/How_to_log_in_to_DSM_with_key_pairs_as_admin_or_root_permission_via_SSH_on_computers#x_anchor_ida890

https://qiita.com/b-wind/items/fe22e34c7ea81cc84d6d

https://www.r2fish.tech/index/2022/03/13/publickey-auth/
