ミドルウェア
############

ミドルウェアオブジェクトは、再利用可能で構成可能なリクエスト処理、あるいは
レスポンス構築処理の層でアプリケーションを「ラップ」する機能を提供します。
視覚的には、アプリケーションは中央で終了し、ミドルウェアはタマネギのように
アプリの周囲を包み込みます。ここでは、Routes、Assets、例外処理、および
CORS ヘッダーミドルウェアでラップされたアプリケーションを確認できます。

.. image:: /_static/img/middleware-setup.png

アプリケーションによってリクエストが処理されると、最も外側のミドルウェアから
リクエストが入力されます。各ミドルウェアは、リクエスト/レスポンスを次のレイヤーに委譲するか、
またはレスポンスを返すことができます。レスポンスを返すことで、下位層がリクエストを見なくなります。
たとえば、開発中にプラグインの画像のリクエストを処理する AssetMiddleware がその例です。

.. image:: /_static/img/middleware-request.png

ミドルウェアがリクエストを処理するアクションを受け取らない場合、コントローラーが用意され、
そのアクションを呼び出すか、例外が発生してエラーページが生成されます。

ミドルウェアは CakePHP における新しい HTTP スタックの部分で、 PSR-7 のリクエスト
およびレスポンスのインターフェイスを活用しています。 PSR-7 標準の活用によって、
`Packagist <https://packagist.org>`__ で利用可能な、あらゆる PSR-7 互換の
ミドルウェアを使うことができます。

CakePHP のミドルウェア
=======================

CakePHP はウェブアプリケーションの一般的なタスクを処理するいくつかのミドルウェアを提供します。

* ``Cake\Error\Middleware\ErrorHandlerMiddleware`` はラップされたミドルウェアからくる
  例外を捕まえ、 :doc:`/development/errors` の例外ハンドラーを使ってエラーページを描画します。
* ``Cake\Routing\AssetMiddleware`` はリクエストが、プラグインの webroot フォルダー
  あるいはテーマのそれに格納された CSS 、 JavaScript または画像ファイルといった、
  テーマまたはプラグインのアセットファイルを参照するかどうかを確認します。
* ``Cake\Routing\Middleware\RoutingMiddleware`` は受け取った URL を解析して、
  リクエストにルーティングパラメーターを割り当てるために ``Router`` を使用します。
* ``Cake\I18n\Middleware\LocaleSelectorMiddleware`` はブラウザーによって送られる
  ``Accept-Language`` ヘッダーによって自動で言語を切り替えられるようにします。
* ``Cake\Http\Middleware\HttpsEnforcerMiddleware`` の利用はHTTPSを要求します。
* ``Cake\Http\Middleware\SecurityHeadersMiddleware`` は、 ``X-Frame-Options``
  のようなセキュリティに関連するヘッダーをレスポンスに簡単に追加することができます。
* ``Cake\Http\Middleware\EncryptedCookieMiddleware`` は、難読化されたデータで
  Cookie を操作する必要がある場合に備えて、暗号化された Cookie を操作する機能を提供します。
* ``Cake\Http\Middleware\CsrfProtectionMiddleware`` は、アプリケーションに CSRF
  保護を追加します。
* ``Cake\Http\Middleware\BodyParserMiddleware`` は ``Content-Type`` ヘッダーに基づいて
  JSON, XML, その他のエンコードされたリクエストボディーをデコードすることができます。
* ``Cake\Http\Middleware\CspMiddleware`` を使用すると、
  アプリケーションに Content-Security-Policy ヘッダを追加するのがより簡単になります。

.. _using-middleware:

ミドルウェアの使用
==================

ミドルウェアは、アプリケーションの全体、または個々のルーティングスコープに適用できます。

全てのリクエストにミドルウェアを適用するには、 ``App\Application`` クラスの
``middleware`` メソッドを使用してください。
アプリケーションの ``middleware`` フックメソッドはリクエスト処理の開始時に呼ばれて、
``MiddlewareQueue`` オブジェクトを加えることができます。 ::

    namespace App;

    use Cake\Http\BaseApplication;
    use Cake\Http\MiddlewareQueue;
    use Cake\Error\Middleware\ErrorHandlerMiddleware;

    class Application extends BaseApplication
    {
        public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
        {
            // ミドルウェアのキューにエラーハンドラーを結びつけます。
            $middlewareQueue->add(new ErrorHandlerMiddleware());
            return $middlewareQueue;
        }
    }

``MiddlewareQueue`` の末尾に追加するだけではなくて、さまざまな操作をすることができます。 ::

        $layer = new \App\Middleware\CustomMiddleware;

        // 追加されたミドルウェアは行列の末尾になります。
        $middlewareQueue->add($layer);

        // 追加されたミドルウェアは行列の先頭になります。
        $middlewareQueue->prepend($layer);

        // 特定の位置に挿入します。もし位置が範囲外の場合、
        // 末尾に追加されます。
        $middlewareQueue->insertAt(2, $layer);

        // 別のミドルウェアの前に挿入します。
        // もしその名前のクラスが見つからない場合、
        // 例外が発生します。
        $middlewareQueue->insertBefore(
            'Cake\Error\Middleware\ErrorHandlerMiddleware',
            $layer
        );

        // 別のミドルウェアの後に挿入します。
        // もしその名前のクラスが見つからない場合、
        // ミドルウェアは末尾に追加されます。
        $middlewareQueue->insertAfter(
            'Cake\Error\Middleware\ErrorHandlerMiddleware',
            $layer
        );

アプリケーション全体にミドルウェアを適用するだけでなく、 :ref:`connecting-scoped-middleware`
を使用して、特定のルートセットにミドルウェアを適用することができます。

プラグインからのミドルウェア追加
--------------------------------

プラグインは ``middleware`` フックメソッドを使って、
ミドルウェアをアプリケーションのミドルウェアキューに適用することができます。 ::

    // ContactManager プラグインの bootstrap.php の中で
    namespace ContactManager;

    use Cake\Core\BasePlugin;
    use Cake\Http\MiddlewareQueue;
    use ContactManager\Middleware\ContactManagerContextMiddleware;

    class Plugin extends BasePlugin
    {
        public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
        {
            $middlewareQueue->add(new ContactManagerContextMiddleware());

            return $middlewareQueue;
        }
    }

ミドルウェアの作成
==================

ミドルウェアは無名関数（クロージャ）として、あるいは呼び出し可能なクラスとしても実装できます。
また、 ``Psr\Http\\ServerMiddlewareInterface`` を拡張するクラスとして実装することもできます。
クロージャは小さな課題に適している一方で、テストを行うのを難しくしますし、複雑な ``Application``
クラスを作ってしまいます。 CakePHP のミドルウェアクラスは、いくつかの規約を持っています。

* ミドルウェアクラスのファイルは **src/Middleware** に置かれるべきです。例えば
  **src/Middleware/CorsMiddleware.php** です。
* ミドルウェアクラスには ``Middleware`` と接尾語が付けられるべきです。例えば
  ``LinkMiddleware`` です。
* ミドルウェアは、 ``Psr\Http\ServerMiddlewareInterface``` を実装しなければなりません。

ミドルウェアは ``$handler->handle()`` を呼ぶか、独自のレスポンスを作成することによって、レスポンスを
返すことができます。我々の単純なミドルウェアで、両方のオプションを見ることができます。 ::

    // src/Middleware/TrackingCookieMiddleware.php の中で
    namespace App\Middleware;

    use Cake\Http\Cookie\Cookie;
    use Cake\I18n\Time;
    use Psr\Http\Message\ResponseInterface;
    use Psr\Http\Message\ServerRequestInterface;
    use Psr\Http\Server\RequestHandlerInterface;
    use Psr\Http\Server\MiddlewareInterface;

    class TrackingCookieMiddleware implements MiddlewareInterface
    {
        public function process(
            ServerRequestInterface $request,
            RequestHandlerInterface $handler
        ): ResponseInterface
        {
            // Calling $handler->handle()  を呼ぶことで、アプリケーションのキューの中で
            // *次の* ミドルウェアにコントロールを任せます。
            $response = $handler->handle($request);

            if (!$request->getCookie('landing_page')) {
                $expiry = new Time('+ 1 year');
                $response = $response->withCookie(new Cookie(
                    'landing_page',
                    $request->getRequestTarget(),
                    $expiry
                ));
            }

            return $response;
        }
    }

さて、我々はごく単純なミドルウェアを作成しましたので、それを我々のアプリケーションに
加えてみましょう。 ::

    // src/Application.php の中で
    namespace App;

    use App\Middleware\TrackingCookieMiddleware;
    use Cake\Http\MiddlewareQueue;

    class Application
    {
        public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
        {
            // 単純なミドルウェアをキューに追加します
            $middlewareQueue->add(new TrackingCookieMiddleware());

            // もう少しミドルウェアをキューに追加します

            return $middlewareQueue;
        }
    }

.. _routing-middleware:

ルーティングミドルウェア
========================

ルーティングミドルウェアは、アプリケーションのルートの適用や、リクエストが実行するプラグイン、
コントローラー、アクションを解決することができます。起動時間を向上させるために、
アプリケーションで使用されているルートコレクションをキャッシュすることができます。
キャッシュされたルートを有効にするために、目的の :ref:`キャッシュ設定 <cache-configuration>`
をパラメーターとして指定します。 ::

    // Application.php の中で
    public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
    {
        // ...
        $middlewareQueue->add(new RoutingMiddleware($this, 'routing'));
    }

上記は、生成されたルートコレクションを格納するために ``routing`` キャッシュエンジンを使用します。

.. _security-header-middleware:

セキュリティヘッダーの追加
==========================

``SecurityHeaderMiddleware`` レイヤーは、アプリケーションにセキュリティ関連の
ヘッダーを簡単に適用することができます。いったんミドルウェアをセットアップすると、
レスポンスに次のヘッダーを適用します。

* ``X-Content-Type-Options``
* ``X-Download-Options``
* ``X-Frame-Options``
* ``X-Permitted-Cross-Domain-Policies``
* ``Referrer-Policy``

このミドルウェアは、アプリケーションのミドルウェアスタックに適用される前に、
流れるようなインターフェースを使用して設定されます。 ::

    use Cake\Http\Middleware\SecurityHeadersMiddleware;

    $securityHeaders = new SecurityHeadersMiddleware();
    $securityHeaders
        ->setCrossDomainPolicy()
        ->setReferrerPolicy()
        ->setXFrameOptions()
        ->setXssProtection()
        ->noOpen()
        ->noSniff();

    $middlewareQueue->add($securityHeaders);

コンテンツセキュリティポリシーヘッダーミドルウェア
==================================================

``CspMiddleware``を使うと、アプリケーションに Content-Security-Policy ヘッダを追加するのがより簡単になります。
これを使う前に ``paragonie/csp-Builder``` をインストールする必要があります。

.. code-block::bash

    composer require paragonie/csp-builder

配列を使用するか、ビルドされた ``CSPBuilder`` オブジェクトを渡すことで、
ミドルウェアを設定することができます。 ::

    use Cake\Http\Middleware\CspMiddleware;

    $csp = new CspMiddleware([
        'script-src' => [
            'allow' => [
                'https://www.google-analytics.com',
            ],
            'self' => true,
            'unsafe-inline' => false,
            'unsafe-eval' => false,
        ],
    ]);

    $middlewareQueue->add($csp);

.. _encrypted-cookie-middleware:

クッキー暗号化ミドルウェア
==========================

アプリケーションが難読化してユーザーの改ざんから保護したいデータを含むクッキーがある場合、
CakePHP のクッキー暗号化ミドルウェアを使用して、ミドルウェア経由でクッキーデータを透過的に
暗号化や復号化することができます。 クッキーデータは、OpenSSL 経由で AES を使用して
暗号化されます。 ::

    use Cake\Http\Middleware\EncryptedCookieMiddleware;

    $cookies = new EncryptedCookieMiddleware(
        // 保護するクッキーの名前
        ['secrets', 'protected'],
        Configure::read('Security.cookieKey')
    );

    $middlewareQueue->add($cookies);

.. note::
    クッキーデータで使用する暗号化キーは、クッキーデータ *のみ* に使用することを
    お勧めします。

このミドルウェアが使用する暗号化アルゴリズムとパディングスタイルは、
CakePHP の以前のバージョンの ``CookieComponent`` と後方互換性があります。

.. _csrf-middleware:

クロスサイトリクエストフォージェリー (CSRF) ミドルウェア
========================================================

CSRF 保護は、アプリケーション全体または特定のルーティングスコープに適用することができます。

.. note::

    次のアプローチの両方を一緒に使用することはできません。1つだけを選択する必要があります。
    両方のアプローチを併用すると、すべての `PUT` および `POST` リクエストで CSRF トークンの不一致エラーが発生します。

``CsrfProtectionMiddleware`` をアプリケーションミドルウェアスタックに適用することにより、
アプリケーションのすべてのアクションを保護します。 ::

    // Application.php の中で
    use Cake\Http\Middleware\CsrfProtectionMiddleware;

    public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
    {
        $options = [
            // ...
        ];
        $csrf = new CsrfProtectionMiddleware($options);

        $middlewareQueue->add($csrf);
        return $middlewareQueue;
    }

``CsrfProtectionMiddleware`` をルーティングスコープに適用することにより、
特定のルートグループを含めたり除外したりできます。 ::

    // Application.php の中で
    use Cake\Http\Middleware\CsrfProtectionMiddleware;

    public function routes(RouteBuilder $routes) : void
    {
        $options = [
            // ...
        ];
        $routes->registerMiddleware('csrf', new CsrfProtectionMiddleware($options));
        parent::routes($routes);
    }

    // config/routes.php の中で
    Router::scope('/', function (RouteBuilder $routes) {
        $routes->applyMiddleware('csrf');
    });

オプションは、ミドルウェアのコンストラクタに渡すことができます。
利用可能な設定オプションは次の通りです。

- ``cookieName`` 送信するクッキー名。デフォルトは ``csrfToken`` 。
- ``expiry`` CSRF トークンの有効期限。デフォルトは、ブラウザーセッション。
- ``secure`` クッキーにセキュアフラグをセットするかどうか。
  これは、HTTPS 接続でのみクッキーが設定され、通常の HTTP 経由での試みは失敗します。
  デフォルトは ``false`` 。
- ``httpOnly`` クッキーに HttpOnly フラグをセットするかどうか。デフォルトは ``false`` 。
- ``field`` 確認するフォームフィールド。デフォルトは ``_csrfToken`` 。
  これを変更するには、FormHelper の設定も必要です。

有効にすると、リクエストオブジェクトの現在の CSRF トークンにアクセスできます。 ::

    $token = $this->request->getAttribute('csrfToken');

ホワイトリストコールバック機能を使用して、
CSRF トークンチェックを実行する URL をより詳細に制御できます。 ::

    // config/routes.php の中で
    use Cake\Http\Middleware\CsrfProtectionMiddleware;

    public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
    {
        $csrf = new CsrfProtectionMiddleware();

        // コールバックが `true` を返す場合、トークンのチェックはスキップされます。
        $csrf->whitelistCallback(function ($request) {
            // Skip token check for API URLs.
            if ($request->getParam('prefix') === 'Api') {
                return true;
            }
        });

         // ルーティングミドルウェアが CSRF 保護ミドルウェアより先にキューに追加されていることを確認してください。
        $middlewareQueue->add($csrf);

        return $middlewareQueue;
    }

.. note::

    Cookie/セッションを使用してステートフルリクエストを処理する URL にのみ CSRF 保護ミドルウェアを適用する必要があります。
    ステートレスリクエスト(例えば API 開発時)は CSRF の影響を受けないため、これらの URL にミドルウェアを適用する必要はありません。

FormHelper との統合
-------------------

``CsrfProtectionMiddleware`` は、シームレスに ``FormHelper`` と統合されます。
``FormHelper`` でフォームを作成するたびに、CSRF トークンを含む隠しフィールドを
挿入します。

.. note::

    CSRF 保護を使用する場合は、常に ``FormHelper`` でフォームを開始する必要があります。
    そうしないと、各フォームに hidden 入力を手動で作成する必要があります。

CSRF 保護と AJAX リクエスト
---------------------------

リクエストデータパラメータに加えて、特別な ``X-CSRF-Token`` ヘッダーを通じて
CSRF トークンを送信することができます。ヘッダーを使用すると、重厚な JavaScript
アプリケーションや XML/JSON ベースの API エンドポイントに CSRF トークンを簡単に
統合することができます。

CSRF トークンは、JavaScript で Cookie の ``csrfToken`` を介して取得するか、
PHPで ``csrfToken`` という名前のリクエストオブジェクト属性を介して取得できます。
JavaScript コードが CakePHP ビューテンプレートとは別のファイルにある場合、
および JavaScript を介して Cookie を解析する機能を既に持っている場合は、
Cookie を使用する方が簡単です。

JavaScript ファイルが別のファイルにあるものの、Cookie の処理を扱いたくない場合、
たとえば、次のようなスクリプトブロックを定義することにより、
レイアウトのグローバル JavaScript 変数にトークンを設定できます。 ::

    echo $this->Html->scriptBlock(sprintf(
        'var csrfToken = %s;',
        json_encode($this->request->getAttribute('csrfToken'))
    ));

このスクリプトブロックの後に読み込まれる任意のスクリプトファイル内で、
``csrfToken`` または ``window.csrfToken`` としてトークンにアクセスできます。

別の代替方法は、次のようなカスタムメタタグにトークンを配置することです。 ::

    echo $this->Html->meta('csrfToken', $this->request->getAttribute('csrfToken'));

次に、 ``csrfToken`` という名前の ``meta`` 要素を探すことで、
jQuery を使用した場合と同じくらい簡単にスクリプト内でアクセスできます。 ::

    var csrfToken = $('meta[name="csrfToken"]').attr('content');

.. _body-parser-middleware:

ボディパーサミドルウェア
======================

アプリケーションが JSON、XML、またはその他のエンコードされたリクエストボディを受け入れる場合、
``BodyParserMiddleware`` を使用すると、それらのリクエストを配列にデコードして、
``$request->getParsedData()`` および ``$request->getData()`` で利用可能です。
デフォルトでは ``json`` ボディのみがパースされますが、オプションでXMLパースを有効にすることができます。
独自のパーサーを定義することもできます。 ::

    use Cake\Http\Middleware\BodyParserMiddleware;

    // JSONのみがパースされます。
    $bodies = new BodyParserMiddleware();

    // XMLパースを有効にする
    $bodies = new BodyParserMiddleware(['xml' => true]);

    // JSONパースを無効にする
    $bodies = new BodyParserMiddleware(['json' => false]);

    // content-type ヘッダーの値にマッチする独自のパーサを
    // それらをパース可能な callable に追加します。
    $bodies = new BodyParserMiddleware();
    $bodies->addParser(['text/csv'], function ($body, $request) {
        // Use a CSV parsing library.
        return Csv::parse($body);
    });

.. _https-enforcer-middleware:

HTTPS 実施ミドルウェア
=========================

HTTPS接続経由の場合にのみアプリケーションを使用できるようにしたい場合、
``HttpsEnforcerMiddleware`` を利用することができます。 ::

    use Cake\Http\Middleware\HttpsEnforcerMiddleware;

    // 常に例外を発生させ、リダイレクトしません。
    $https = new HttpsEnforcerMiddleware([
        'redirect' => false,
    ]);

    // リダイレクト時にステータスコード302を送信します。
    $https = new HttpsEnforcerMiddleware([
        'redirect' => true,
        'statusCode' => 302,
    ]);

    // リダイレクトレスポンスで追加のヘッダーを送信します。
    $https = new HttpsEnforcerMiddleware([
        'headers' => ['X-Https-Upgrade', => true],
    ]);

    // ``debug`` がオンの場合、HTTPSの強制を無効にします。
    $https = new HttpsEnforcerMiddleware([
        'disableOnDebug' => true,
    ]);

GETを使用しない非HTTPSリクエストを受信した場合、``BadRequestException`` が発生します。

.. meta::
    :title lang=ja: Http ミドルウェア
    :keywords lang=ja: http, ミドルウェア, psr-7, リクエスト, レスポンス, wsgi, アプリケーション, baseapplication, https
