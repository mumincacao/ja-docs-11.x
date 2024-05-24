# ハッシュ

- [イントロダクション](#introduction)
- [設定](#configuration)
- [基本的な使用法](#basic-usage)
    - [パスワードのハッシュ](#hashing-passwords)
    - [パスワードがハッシュと一致するかの確認](#verifying-that-a-password-matches-a-hash)
    - [パスワードを再ハッシュする必要があるかの判断](#determining-if-a-password-needs-to-be-rehashed)
- [ハッシュアルゴリズムの確認](#hash-algorithm-verification)

<a name="introduction"></a>
## イントロダクション

Laravelの`Hash`[ファサード](/docs/{{version}}/facades)は、ユーザーパスワードを保存するための安全なBcryptおよびArgon2ハッシュを提供します。[Laravelアプリケーションスターターキット](/docs/{{version}}/starter-kits)のいずれかを使用している場合、デフォルトで登録と認証にBcryptが使用されます。

Bcryptは、その「作業係数」が調整可能であるため、パスワードのハッシュに最適です。つまり、ハードウェアの能力が上がると、ハッシュの生成にかかる時間が長くなる可能性があります。パスワードをハッシュする場合は、遅いのが利点です。アルゴリズムがパスワードをハッシュするのに時間がかかるほど、悪意のあるユーザーがアプリケーションに対するブルートフォース攻撃で使用される可能性のあるすべての文字列ハッシュ値の「レインボーテーブル」を生成するのにかかる時間が長くなります。

<a name="configuration"></a>
## 設定

Laravelはデフォルトで、データのハッシュ化に`bcrypt`ハッシュドライバを使用します。しかし、[`argon`](https://en.wikipedia.org/wiki/Argon2)や[`argon2id`](https://en.wikipedia.org/wiki/Argon2)など、他のハッシュドライバもサポートしています。

`HASH_DRIVER`環境変数を浸かり、アプリケーションのハッシュドライバを指定できます。しかし、Laravelのハッシュドライバオプションをすべてカスタマイズしたい場合は、`config:publish` Artisanコマンドを使用して、完全な`hashing`設定ファイルをリソース公開する必要があります。

```bash
php artisan config:publish hashing
```

<a name="basic-usage"></a>
## 基本的な使用法

<a name="hashing-passwords"></a>
### パスワードのハッシュ

`Hash`ファサードで`make`メソッドを呼び出すことにより、パスワードをハッシュすることができます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Hash;

    class PasswordController extends Controller
    {
        /**
         * ユーザーのパスワードを更新
         */
        public function update(Request $request): RedirectResponse
        {
            // 新しいパスワードの長さをバリデート…

            $request->user()->fill([
                'password' => Hash::make($request->newPassword)
            ])->save();

            return redirect('/profile');
        }
    }

<a name="adjusting-the-bcrypt-work-factor"></a>
#### Bcryptの作業係数の調整

Bcryptアルゴリズムを使用している場合、`make`メソッドを使用すると、`rounds`オプションを使用してアルゴリズムの作業係数を管理できます。ただし、Laravelが管理するデフォルトの作業係数は、ほとんどのアプリケーションで適切でしょう。

    $hashed = Hash::make('password', [
        'rounds' => 12,
    ]);

<a name="adjusting-the-argon2-work-factor"></a>
#### Argon2作業係数の調整

Argon2アルゴリズムを使用している場合、`make`メソッドを使用すると、`memory`、`time`、`threads`オプションを使用してアルゴリズムの作業要素を管理できます。ただし、Laravelが管理するデフォルト値は、ほとんどのアプリケーションで適切でしょう。

    $hashed = Hash::make('password', [
        'memory' => 1024,
        'time' => 2,
        'threads' => 2,
    ]);

> [!NOTE]
> これらのオプションの詳細には、[Argonハッシュに関するPHP公式ドキュメント](https://secure.php.net/manual/en/function.password-hash.php)を参照してください。

<a name="verifying-that-a-password-matches-a-hash"></a>
### パスワードがハッシュと一致するかの確認

`Hash`ファサードが提供する`check`メソッドを使用すると、指定するプレーンテキスト文字列が指定するハッシュに対応することを確認できます。

    if (Hash::check('plain-text', $hashedPassword)) {
        // パスワードが一致
    }

<a name="determining-if-a-password-needs-to-be-rehashed"></a>
### パスワードを再ハッシュする必要があるかの判断

`Hash`ファサードが提供する`needsRehash`メソッドを使用すると、パスワードがハッシュされてから、ハッシャーによって使用される作業要素が変更されたかどうかを判別できます。一部のアプリケーションは、アプリケーションの認証プロセス中にこのチェックを実行することを選択しています。

    if (Hash::needsRehash($hashed)) {
        $hashed = Hash::make('plain-text');
    }

<a name="hash-algorithm-verification"></a>
## ハッシュアルゴリズムの確認

ハッシュアルゴリズムの操作を防ぐために、Laravelの`Hash::check`メソッドは最初に、指定したハッシュがアプリケーションで選択しているハッシュアルゴリズムを使って生成されているかを確認します。アルゴリズムが異なる場合、`RuntimeException`例外を投げます

これはほとんどのアプリケーションで期待される動作であり、ハッシュアルゴリズムが変更されることは期待されておらず、異なるアルゴリズムは悪意のある攻撃の兆候となり得ます。しかし、あるアルゴリズムから別のアルゴリズムに移行する場合など、アプリケーション内で複数のハッシュアルゴリズムをサポートする必要がある場合は、`HASH_VERIFY`環境変数を`false`に設定することで、ハッシュアルゴリズムの検証を無効にできます。

```ini
HASH_VERIFY=false
```
