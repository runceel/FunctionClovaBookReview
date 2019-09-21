# 指摘内容箇条書き

## Newtonsoft.Json のバージョン問題

近いうちに解消されそうです。

https://github.com/Azure/azure-functions-host/issues/4049


## HttpTrigger の HTTP メソッドの指定

今回のようなスマートスピーカーのエンドポイントとしては POST メソッドにのみ対応していればいいので以下のように "post" のみで問題ない。

```cs
[FunctionName(nameof(AlexaEndpoint))]
public static async Task<IActionResult> Run(
  // post だけでいい。 Route も指定しないなら不要
  [HttpTrigger(AuthorizationLevel.Function, "post")] 
  HttpRequest req,
  ILogger log)
{
  // 省略
}
```

## ディクショナリーの場合は `[]` のコレクション初期化子を使おう

List などと同じような書き方もありますがディクショナリーと同じインターフェースを持つものであれば `[]` を使った方が可読性が高い（好み）と思います。

#### オリジナルコード
```cs
webhookResponse.Payload = new Struct
{
    Fields =
    {
        {
            "google", Value.ForStruct(new Struct
            {
                Fields = { { "expectUserResponse", Value.ForBool(false) } }
            })
        }
    }
};
```

#### 改善？版
```cs
webhookResponse.Payload = new Struct
{
    Fields =
    {
        ["google"] = Value.ForStruct(new Struct
        {
            Fields =
            {
                ["expectUserResponse"] = Value.ForBool(false),
            },
        }),
    },
};
```

参考ドキュメント：
https://docs.microsoft.com/ja-jp/dotnet/csharp/programming-guide/classes-and-structs/how-to-initialize-a-dictionary-with-a-collection-initializer

## スレッドセーフじゃないオブジェクトの使いまわし問題

`LoggableClova` クラスが以下のように定義され

```cs
using LineDC.CEK;
using Microsoft.Extensions.Logging;
namespace DiceRoller
{
    // IClovaを拡張
    public interface ILoggableClova : IClova
    {
        // ロガーを受け取るプロパティ
        ILogger Logger { get; set; }
    }
}
```

以下のように `AddClova` で DI コンテナに登録されて

```cs
using LineDC.CEK;
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
[assembly: FunctionsStartup(typeof(DiceRoller.Startup))]
namespace DiceRoller
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            builder.Services.AddClova<ILoggableClova, DiceClova>();
        }
    }
}
```

ロガーが設定されるようになっています。

```cs
[FunctionName(nameof(ClovaEndpoint))]
public async Task<IActionResult> Run(
[HttpTrigger(AuthorizationLevel.Function, "get", "post",
Route = null)] HttpRequest req,
ILogger log)
{
    // ロガーをセット
    Clova.Logger = log;
    var response = await Clova.RespondAsync(
        req.Headers["SignatureCEK"], req.Body);
    return new OkObjectResult(response);
}
```

CEK.CSharp 内の `AddClova` メソッドが

```cs
using LineDC.CEK.Models;
using Microsoft.Extensions.DependencyInjection;

namespace LineDC.CEK
{
    public static class ClovaServiceCollectionExtensions
    {
        public static IServiceCollection AddClova<T1, T2>(this IServiceCollection services, Lang defaultLang = Lang.Ja)
            where T1 : class, IClova
            where T2 : ClovaBase, T1, new()
        {
            var clova = new T2();
            clova.SetDefaultLang(defaultLang);

            return services.AddScoped<T1, T2>(_ => clova);
        }
    }
}
```

のようになってるので `AddClova` メソッドを使うと実質シングルトンになります。そのため、この実装だとスキルが同時に使用されるとロガーが他人用のもので上書きされてしまいます。

さらに `ClovaBase` の実装を見ると `Request`, `Response` もインスタンス単位で持っているので同時にスキルが実行されると他人への回答が混ざってしまう危険もありました。

DI コンテナーで管理するクラスのスコープには気を付けよう。

この問題はプルリクエストが作られてるので、CEK.CSharp 0.2.2 以降で修正されます。

https://github.com/line-developer-community/clova-extensions-kit-csharp/pull/6

## Clova を継承したクラスは DI 出来ない問題

`AddClova` の中でデフォルトコンストラクターで `new` されてるので自作の `ClovaBase` を継承したクラスを `AddClova` で DI コンテナーに追加した場合は何も DI されません。何か DI をしたい場合には以下のように自分で DI コンテナーに登録する or プルリクエストを投げるのがいいと思います。

```cs
// ClovaBase を継承したクラスに渡す設定が詰まったクラス
public class CEKCSharpConfiguration
{
    public Lang Lang { get; set; }
}

// ClovaBase を継承したクラス（ここに色々書く）
public class MyClova : ClovaBase
{
    public MyClova(CKECSharpConfiguration c)
    {
        SetDefaultLang(c.Lang);
    }

    // 色々書く
}

// Startup.cs で以下のように登録する
builder.Services.AddSingleton(new CEKCSharpConfiguration { Lang = Lang.Ja });
builder.Services.AddScoped<IClova, MyClova>();
```

## まとめ

各種プラットフォームに対応したスキルを、作るためのノウハウの詰まった本なのでスマートスピーカーに興味のある人は是非読んでみてください！！
Azure Functions を使わない場合でも、このように共通化を考えることが出来るという参考になると思います。
