### インストール

Fabricationは以下のRubyで動作検証を行っています：1.9.3、2.0.0、Rubinius

（Ruby 1.8.x系との互換性があるのはバージョン1.2.0までです）

Bundler経由で使用する場合は、Gemfileに以下のように記述するだけです。

    gem 'fabrication'

以下のような所定のパスに置かれたFabricatorは自動で読み込まれるため、require文を新たに追加する必要はありません。

    spec/fabricators/**/*fabricator.rb
    test/fabricators/**/*fabricator.rb

### 設定

設定をオーバーライドするには、以下のようなconfigureブロックを記述したfabrication.rbをsupportフォルダに設置してください。

    Fabrication.configure do |config|
      config.fabricator_path = 'data/fabricators'
      config.path_prefix = Rails.root
      config.sequence_start = 10000
    end

#### 設定可能な項目

##### fabricator_path

プロジェクトフォルダ内での、Fabricator定義が置かれるパスを設定します。

デフォルト: ['test/fabricators', 'spec/fabricators']

##### path_prefix

アプリケーションのルートパスを指定できます。Railsで使用する時、特に便利でしょう。

デフォルト: （定義されている場合）Rails.root、（それ以外の場合） '.'

##### sequence_start

シーケンスで使用される連番の最初の番号を指定できます。ここで設定を行っても、個々のシーケンス定義によりオーバーライドできる点は変わりありません。

デフォルト: 0

### Fabricatorを定義する

#### 引数

Fabricatorに対して最初に与える引数は、生成するオブジェクトまたは定義するアソシエーションの名前です。この名前は、クラス名をシンボル化した形式で付けます。

    class Person; end

    Fabricator(:person)

クラス名と異なる名前を付ける場合、第2引数に `from: :symbolized_class_name` のような形式で、シンボル化した名前を与える必要があります。

    Fabricator(:adult, from: :person)

ここで `:from` に指定する値は、元のクラス名だけでなく他のfabricatorの名前を使うこともできます。

#### 属性

Fabricatorブロックは必須ではありませんが、ブロック内で生成する属性を列挙すると、その宣言順にそれらが生成されます。

    Fabricator(:person) do
      name 'Greg Graffin'
      profession 'Professor/Musician'
    end

動的な値を生成する場合、ブロックで属性を与えることができます。

    Fabricator(:person) do
      name { Faker::Name.name }
      profession { %w(Butcher Baker Candlestick\ Maker).sample }
    end

属性は宣言順に処理され、ある属性より上で定義されたフィールドはブロック変数として使うことができます。

    Fabricator(:person) do
      name { Faker::Name.name }
      email { |attrs| "#{attrs[:name].parameterize}@example.com" }
    end

#### キーワード ####
** フィールド名にキーワードを使うのはベストプラクティスとは言えません! **

ブロック変数を使うことで予約されたキーワード（alias, class, def, if, while, …）の名前が付いたフィールドを参照できるからです。

    class Person
      attr_accessor :alias, :codename
      alias aka codename
    end

    Fabricator(:person) do |f|
      f.alias 'James Bond'
      codename '007'
    end

    Fabricate(:person).aka #=> '007'

#### アソシエーション

属性名を書くだけで、その名前を持つ他のFabricatorとの関連付けを行うことができます。
Fabricationは名前からFabricatorを見つけ出し、オブジェクトを生成し、現在のオブジェクトとの関連付けを行います。これは `belongs_to` アソシエーションを使う際に便利です。

    Fabricator(:person) do
      vehicle
    end

…は以下と同じ動作をします…

    Fabricator(:person) do
      vehicle { Fabricate(:vehicle) }
    end

個々の定義によってどのFabricatorを使うか明示的に指定することもできます。

    Fabricator(:person) do
      ride(fabricator: :vehicle)
    end

…は以下と同じ動作をします…

    Fabricator(:person) do
      ride { Fabricate(:vehicle) }
    end

countパラメータを使うと、オブジェクトを要素に持つ配列を生成することもできます。属性ブロックは、インクリメントされる値とともに生成されるオブジェクトを含みます。ブロックを省略したいかもしれませんが、その場合も問題なく動作します。

    Fabricator(:person) do
      open_source_projects(count: 5)
      children(count: 3) { |attrs, i| Fabricate(:person, name: "Kid #{i}") }
    end

#### 継承

`:from` 属性を使うと、他のFabricatorから属性を継承することができます。

    Fabricator(:llc, from: :company) do
      type "LLC"
    end

`:from` オプションを指定することで、その名前のFabricatorからクラスと全ての属性を継承します。

`:class_name` パラメータを指定すると、生成されるクラスを厳密に指定することもできます。

    Fabricator(:llc, class_name: :company) do
      type "LLC"
    end

#### 初期化のカスタマイズ

もし通常の初期化手順でオブジェクトを作りたくない場合、 `initialize_with` オプションでこれをオーバーライドできます。

    Fabricator(:car) do
      initialize_with { Manufacturer.produce(:new_car) }
      color 'red'
    end

initialize_withブロックでインスタンス化されるオブジェクトは、定義された属性が全て適用され、Fabricatorメソッド呼び出しの際に返されます。

#### コールバック

オブジェクトのコールバックとは別に、Fabricatorでコールバックを指定することができます。

Fabricationのオブジェクト生成サイクルの中に何らかの処理を挟みたい場合、以下のコールバックを使えます。

    after_build
    before_validation
    after_validation
    before_save
    before_create
    after_create
    after_save

これらは名前の通りのタイミングで実行されます。Fabricator内でこれらをブロック付きで定義すると、生成されるオブジェクトと定義された一時的な属性のハッシュを受け取ることができます。

Fabricatorが何らかの動作をする時、これらを定義しておくことで期待通りの動作をさせることができるでしょう。また、コールバックは複数定義できます。つまり、一つのFabricatorの中で同じタイプのコールバックを複数宣言してもよく、他のFabricatorを継承している時も、それらが競合して混乱することはありません。

    Fabricator(:place) do
      before_validation { |place, transients| place.geolocate! }
      after_create { |place, transients| Fabricate(:restaurant, place: place) }
    end

コンストラクタに引数が必要なオブジェクトを使う時は `on_init` コールバックを使いましょう。

    Fabricator(:location) do
      on_init { init_with(30.284167, -81.396111) }
    end

#### エイリアス

Fabricator呼び出しに `:aliases` オプションを付けることで、そのFabricatorにエイリアス（別名）を設定できます。

    Fabricator(:thingy, aliases: [:widget, :wocket])

このように書くことで、:thingy、:widget、:wocketのどの名前でもFabricatorで生成されたオブジェクトを受け取ることができます。

#### 一時的属性（Transient Attributes）

一時的属性を用いて、生成されたクラスには含まれない変数を使うことができます。一時的属性は他の属性を生成する時に通常の属性のように振る舞いますが、オブジェクトに属性が関連づけられる際に消去されます。

    Fabricator(:city) do
      transient :asian
      name { |attrs| attrs[:asian] ? "Tokyo" : "Stockholm" }
    end

    Fabricate(:city, asian: true)
    # => <City name: 'Tokyo'>

複数の一時的属性を `transient` で設定できます。

    Fabricator(:the_count) do
      transient :one, :two, :three
    end

また、デフォルト値をハッシュ形式で末尾に追加できます。

    Fabricator(:fruit) do
      transient :color, delicious: true
    end


#### リロード

Fabricationを初期状態に戻したい場合は、以下のメソッドを呼び出してください。

    Fabrication.clear_definitions

これはSporkのような環境を使っていて、テスト環境全部をリロードしたくない時などに便利です。

### オブジェクトの生成

#### 基本

オブジェクトを生成する最もシンプルな方法は、FabricateにFabricatorの名前を渡すことです。

    Fabricate(:person)

こうすることで、Fabricatorで定義した属性を持つPersonクラスのインスタンスが得られます。

追加で属性を設定したり、Fabricatorで定義した属性をオーバーライドしたい場合、Fabricateにハッシュを渡すとその通りに値がセットされます。

    Fabricate(:person, first_name: "Corbin", last_name: "Dallas")

Fabricateに渡された引数は、Fabricatorでの定義より常に優先されます。

#### ブロックを付けて生成

ハッシュを渡すだけではなく、Fabricateにブロックを渡してもFabricatorでの定義と全く同じ機能を使ってオブジェクトを生成できます。

    Fabricate(:person, name: "Franky Four Fingers") do
      addiction "Gambling"
      fingers(count: 9)
    end

ハッシュで指定した内容は、ブロックで指定した内容より優先されます。

#### 永続化しないオブジェクト生成

データベースにオブジェクトを永続化させたくない場合、 `Fabricate.build` を使うことでsaveステップを省略することができます。他の便利な機能はこの場合でも問題なく使えます。

    Fabricate.build(:person)

`build` を呼び出した場合、その `build` が終了するまでは、他の全ての `Fabricate` 呼び出しは `build` として処理されます。オブジェクトを `build` している時に他のオブジェクトが生成された場合でも、それらは両方ともデータベースに永続化されません。

以下の例では、 `person` を `build` しているブロックの中で `Fabricate(:car)` を呼び出していますが、これらはどちらも永続化されません。

    Fabricate.build(:person) do
      cars { 2.times { Fabricate(:car) } }
    end

#### 属性のハッシュ

どのオブジェクトでも、それをハッシュ形式で受け取ることができます。この形式には定義されたフィールドが全て含まれますが、実際のオブジェクトは生成されませんし永続化もされません。もし `ActiveSupport` を使っている場合はこのハッシュは `HashWithIndifferentAccess` クラスになり、そうでない場合は通常のRubyの `Hash` クラスになります。

    Fabricate.attributes_for(:company)

### シーケンス

シーケンスを使うと、その処理に特有の連番を得ることができます。Fabricationは簡単かつ柔軟にシーケンスを使う方法を提供します。

アプリケーションの中のどこでも、0から始まる連番をシンプルなコマンドで生成できます。

    Fabricate.sequence
    # => 0
    # => 1
    # => 2

引数を付けると、シーケンスに名前を付けることができます。

    Fabricate.sequence(:name)
    # => 0
    # => 1
    # => 2

第2引数に数字を与えると連番の開始番号を指定できます。最初の呼び出し時には開始番号が常に返されますが、2度目以降の呼び出しでは無視されます。

    Fabricate.sequence(:number, 99)
    # => 99
    # => 100
    # => 101

また、全てのシーケンスに共通の開始番号を指定することもできます（「設定」の項目を参照）。

eメールアドレスを生成する時などには、ブロックを渡すとそのレスポンスが戻ります。

    Fabricate.sequence(:name) { |i| "Name #{i}" }
    # => "Name 0"
    # => "Name 1"
    # => "Name 2"

Fabricatorの中でシーケンスを使う場合、以下のように短く書くことができます。

    Fabricate(:person) do
      ssn { sequence(:ssn, 111111111) }
      email { sequence(:email) { |i| "user#{i}@example.com" } }
    end
    # => <Person ssn: 111111111, email: "user0@example.com">
    # => <Person ssn: 111111112, email: "user1@example.com">
    # => <Person ssn: 111111113, email: "user2@example.com">

### Rails 3

Rails 3でモデル生成をする際に、Fabricatorも自動生成するには `config/application.rb` で設定します。RSpecを使う場合は…

    config.generators do |g|
      g.test_framework      :rspec, fixture: true
      g.fixture_replacement :fabrication
    end

test/unitを使う場合は…

    config.generators do |g|
      g.test_framework      :test_unit, fixture_replacement: :fabrication
      g.fixture_replacement :fabrication, dir: "test/fabricators"
    end

minitestを使う場合は…

    config.generators do |g|
      g.test_framework      :mini_test, fixture_replacement: :fabrication
      g.fixture_replacement :fabrication, dir: "test/fabricators"
    end

以上の設定をすれば、Fabricatorはモデル生成時に同時生成されます。

    rails generate model widget

を実行すると…

    spec/fabricators/widget_fabricator.rb

    Fabricator(:widget) do
    end

も生成されます。

### Cucumberステップ

#### インストール

Gemパッケージにはいくつかの便利なCucumberステップをstep_definitionsフォルダにロードするジェネレータが含まれています。

    rails generate fabrication:cucumber_steps

#### ステップ定義

WidgetのFabricatorが定義済みなら、「widget」と書くだけでwidgetが一つ生成可能です。

    Given 1 widget

特定の属性を持った一つのwidgetを生成する場合は…

    Given the following widget:
      | name      | widget_1 |
      | color     | red      |
      | adjective | awesome  |

複数の「widgets」を生成する場合は…

    Given 10 widgets

特定の属性を持った複数の「widgets」を生成する場合は…

    Given the following widgets:
      | name     | color | adjective |
      | widget_1 | red   | awesome   |
      | widget_2 | blue  | fantastic |
      ...

既に生成済みのwidgetに関連づけられた「wockets」を生成する場合は…

    And that widget has 10 wockets

widgetに関連づけられた特定の属性を持つ「wockets」を生成する場合は…

    And that widget has the following wocket:
      | title    | Amazing |
      | category | fancy   |

これは最も直近に生成された「widget」を返し、wocketのFabricatorに渡します。これには「wocket」が「widget」のセッターを持っている必要があります。

もっと複雑なケース、「widgets」と「wockets」が既に生成されていて、それらの関連付けを設定したい場合には…

    And that wocket belongs to that widget

一つまたは複数のオブジェクトがデータベースに永続化されているか、確認することができます…

    Then I should see 1 widget in the database

または、特定の属性を持ったオブジェクトが永続化されているか確認できます…

    Then I should see the following widget in the database:
      | name  | Sprocket |
      | gears | 4        |
      | color | green    |

これは「widget」という名前を持つFabricatorに定義づけられたクラスを探し、テーブルに列挙されたパラメータを使ってwhere(…)で絞り込みを行います。この方法ではデータベースにある一つのオブジェクトに対して確認を行うので、できるだけ詳しく記述しましょう！

#### トランスフォーム

Cucumberステップのテーブルに適用するトランスフォームを定義することができます。それらは縦横どちらのテーブルでも使うことができ、カラムの値を書き換えることができます。替わりにセットされるオブジェクトを生成するためのロジックを文字列で与えます。これらは `spec/fabricators` フォルダに置くこともできます。

例えば、「company」という名前のフィールド全てに対するトランスフォームを定義するとします。以下のコードはセルからlambdaに文字列を渡してその戻り値を属性にセットすることで、与えられたcompany nameを生成されたインスタンスのそれに置き換えます。

    Fabrication::Transform.define(:company, lambda{ |company_name| Company.where(name: company_name).first })

期待する文字列をセルに入れ、カラム名をシンボルと同じにすることでこれを呼び出すことができます。

    Scenario: a single object with transform to apply
      Given the following company:
        | name | Widgets Inc |
      Given the following division:
        | name    | Southwest   |
        | company | Widgets Inc |
      Then that division should reference that company

    Scenario: multiple objects with transform to apply
      Given the following company:
        | name | Widgets Inc |
      Given the following divisions:
        | name      | company     |
        | Southwest | Widgets Inc |
        | North     | Widgets Inc |
      Then they should reference that company

divisionsが生成された時、それはラムダ式によってcompanyオブジェクトを受け取ります。

特定のモデルにのみ限定してこの動作を行いたい場合、 `only_for` を指定します。

    Fabrication::Transform.only_for(:division, :company, lambda { |company_name| Company.where(name: company_name).first })

### その他

#### 助けが必要？

立ち入った質問があったり、助言が必要であればFabrication [メーリングリスト](https://groups.google.com/group/fabricationgem) にメールしてください。
[ドキュメントの生のファイル](https://github.com/paulelliott/fabrication-site/blob/master/views/_content.markdown) もあります。

何らかの不具合を見つけたら、 [GitHubでissue報告](https://github.com/paulelliott/fabrication/issues) をお願いします。

#### Vim

[rails.vim](https://github.com/tpope/vim-rails) でFabricationがサポートされました！これをインストールすると、Fabricatorファイルを以下のようなコマンドで開けます。

    :Rfabricator your_fabricator

#### Make構文

MachinistからFabricationに移行しているのであれば、make構文をincludeすることでその作業を楽にできます。 `fabrication/syntax/make` をrequireすると、 `make` と `make!` をクラス内で使えるようになります。

また、クラス名と同じ名前から始まるFabricatorで後置構文を使えます。

    Fabricator(:author_with_books, from: :author) do
      books(count: 2)
    end
    Author.make(:with_books)

### プロジェクトに貢献する

私 ([paulelliott](http://github.com/paulelliott)) は積極的にこのプロジェクトをメンテナンスしています。何か貢献していただけるのであれば、プロジェクトをfolkし、変更とテストをfeatureブランチで行い、pull requestしてください。

もちろん、Fabricationのソースコードは [Githubにあります](https://github.com/paulelliott/fabrication) し、 [Fabricationのサイト](https://github.com/paulelliott/fabrication-site) のソースもあります。

Rakeを通すには…

1. プロジェクトをcloneします
2. mongodbとsqlite3をインストールします (brew install …)
3. bundlerをインストールします (gem install bundler)
4. プロジェクトのルートに移動して `bundle` を実行します
5. `rake` でテストが実行されます。全部のテストが緑色になるはず！

### 日本語訳について（About Japanese translation）

この日本語訳は試訳です。細かい部分で技術的または翻訳上の誤りがあるかもしれません。何か間違いがあったり、より良い訳があれば教えてください。

This translation trial may contain technical or English-reading mistakes in details. If you figure out them or have better translations, please tell me that.
