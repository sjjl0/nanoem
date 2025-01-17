==========================================
nanoem の技術的説明
==========================================

.. important::
   単に nanoem を使うだけであればこのドキュメントを読む必要は全くありません。ここは nanoem のソースコードを読もうとする技術者向けに書かれています。

.. warning::
   コード全体で少なくとも10万行以上あり、ゲームプログラムあるいは小規模なゲームエンジンを読み解く位の読解難易度があります。
   もちろん、 MMD/MME についても理解する必要があり、加えて依存ライブラリがそれなりにある上にそれらを最低限把握する必要があります。

ここでは nanoem のソースコードがどうなっているかを説明するドキュメントです。

おおまかに nanoem はふたつのコンポーネントに分かれています

  * nanoem
  * emapp

加えて OS 毎に固有の機能を実装した UI 層と依存ライブラリを集約した ``dependencies`` がありますが、
``nanoem`` と ``emapp`` を中心に説明します。

nanoem のソースコードは `GitHub 上で公開 <https://github.com/hkrn/nanoem>`_ されています。

nanoem
******************************************

C 言語で書かれた PMD/PMX/VMD のデータ操作に特化したライブラリです。厳密な意味での nanoem とはこちらのことを指します。
また、追加拡張として以下を付属しています。

* ミュータブルなデータ操作
* 文字変換 (CoreFoundation/ICU/Win32/Qt)
* PMM データ操作
* NMD データ操作
* 物理演算
* PMD -> PMX 変換
* JSON

nanoem の API は CoreFoundation の API を参考に設計していて原則イミュータブルな API を基本として提供しています。
また、全てのオブジェクト（構造体）は直接メンバーにアクセスすることのできない不透明型なものとして扱います。
これにより ABI 互換を確保しています。

オブジェクトは ``Model`` と ``Motion`` に大別しており、 ``Model`` は PMD/PMX 両方とも読み込める仕様ですが、
emapp 側の実装により PMD については追加拡張で提供されている変換機能を使って PMX に変換してから扱います。
``Motion`` は VMD のみですが、拡張関数により NMD も読み込むことが可能です。両方とも同じインターフェースを持ちます。

追加拡張として提供されているミュータブルなデータ操作を使うとイミュータブルなオブジェクトを直接書き込む形でデータの書き換えが可能です。
こちらはデータ書き換えだけでなく書込み可能なバッファオブジェクトに対して書き込むことが可能です。
ミュータブルなオブジェクトは必ずひとつのイミュータブルなオブジェクトに対してのみ利用する必要があります。

オブジェクト階層
==========================================

* ``Model``

  * ``Vertex``

    * 頂点

  * ``Material``

    * 材質またはマテリアル

  * ``Bone``

    * ボーン

  * ``Constraint``

    * IK
    * BulletPhysics の ``btConstraint`` とは違うので注意
    * この下に IK リンクに相当する ``ConstraintJoint`` がある

  * ``Morph``

    * モーフまたは表情

      * ``BoneMorph``

        * ボーンモーフ

      * ``GroupMorph``

        * グループモーフ

      * ``FlipMorph``

        * フリップモーフ

      * ``ImpulseMorph``

        * インパルスモーフ

      * ``MaterialMorph``

        * 材質モーフ

      * ``UVMorph``

        * UV モーフ

      * ``VertexMorph``

        * 頂点 モーフ

  * ``Label``

    * カテゴリ

  * ``RigidBody``

    * 剛体
    * BulletPhysics の ``btRigidBody`` に対応する

  * ``Joint``

    * ジョイント
    * BulletPhysics の ``btConstraint`` に対応する

  * ``SoftBody``

    * ソフトボディ
    * BulletPhysics の ``btSoftBody`` に対応する

* ``Motion``

  * ``AccessoryKeyframe``

    * アクセサリのキーフレーム
    * NMD のみで VMD は対応しない

  * ``BoneKeyframe``

    * ボーンのキーフレーム
    * NMD ではさらに追加で物理演算の切替情報を持つ

  * ``CameraKeyframe``

    * カメラのキーフレーム

  * ``LightKeyframe``

    * 照明のキーフレーム

  * ``ModelKeyframe``

    * モデルのキーフレーム
    * NMD ではさらに追加で外部親などの情報を持つ

  * ``MorphKeyframe``

    * モーフのキーフレーム

  * ``SelfShadowKeyframe``

    * セルフシャドウのキーフレーム

NMD について
==========================================

NMD は VMD の上位互換として VMD を拡張した protobuf ベースのバイナリデータです。

* ボーン及びモーフ名に対する制約がない

  * VMD の場合は 15bytes 以内におさめる必要がある

* VMD と比較して概ね 10% 以上の削減が可能

  * 名前を数値の ID として管理しているため

VMD のデータを NMD としてそのまま保存できます。NMD を VMD に保存することもできますが、
NMD にしか保存できない拡張情報は失われます。

データ仕様は ``nanoem/proto/motion.proto`` で定義されています。

エラーについて
==========================================

ステータスを示す列挙型を使います。エラーが発生する可能性のある関数は必ずその列挙型を関数の最後の引数にとります。

カスタムアロケータについて
==========================================

nanoem は組み込みでカスタムアロケータに差し替える機能を提供しており、nanoem 内では必ずカスタムアロケータを経由してメモリ確保及び解放を行います。
emapp ではそのカスタムアロケータを利用してメモリリークをチェックします。

文字列について
==========================================

文字列は ShiftJIS/UTF-8/UTF-16 を扱う必要があるため、文字列オブジェクトとして独立した存在で扱います。
追加機能として文字列変換を提供していますが、こちらは事実上必須になっています。

文字列のファクトリーオブジェクトを経由する形で専用の関数を使って文字列オブジェクトと文字列をやり取りします。

ユーザオブジェクトについて
==========================================

各オブジェクトには任意のオブジェクトに紐付けることができるユーザオブジェクトがあります。
通常ユーザオブジェクトによる任意のオブジェクトには関与しませんが、オブジェクトのデストラクタが呼ばれると
そのタイミングで任意のオブジェクトに対する破棄を行います。

emapp において ``Model`` のオブジェクトはこれを利用して拡張データをもたせています。
その一方で ``Motion`` のオブジェクトは単純に使う理由がないことから利用していません。

nanoem の実装ポリシー
==========================================

* スペースのみかつインデントは 4

  * ``.editorconfig`` で定義

* C89 ベース

  * 変数の定義は関数の先頭で行う

* オブジェクトに相当する構造体は全て opaque とする

  * 構造体のメンバーアクセスは必ず関数を通じて行う
  * メンバーを直接公開することを禁止

* 名前付けは OpenCL をベースにしたカスタム

  * 構造体の名前は ``lower_snake_case``
  * 関数名は nanoem を先頭につけて ``UpperCamelCase``
  * 定数は ``UPPER_SNAKE_CASE``

emapp
******************************************

C++ で書かれたアプリケーションのコアとなるライブラリです。nanoem の大半の処理はここに集中しています。
nanoem と emapp を土台に、プラットフォーム毎の UI は UI 層に分離させるように設計しています。

* emapp は歴史的経緯から C++ の例外、RTTI 及び C++11 の一部機能 (nullptr) を除いてつかっていません

  * :ref:`1BF7070C-25E0-4E04-B314-6C67FE55E6AB` を参照
  * 共有ポインタや自動ポインタも使っていないため、ポインタ管理は厳格に行う必要があります

* 依存ライブラリの関係から C++14 対応のコンパイラが必要です

ライフサイクル
==========================================

emapp のライフサイクルは比較的ゲームあるいはゲームエンジンに近いものになっています。

* アプリケーションの初期化

  * 各種ライブラリの初期化
  * プロジェクトの作成
  * 前回クラッシュが発生した場合はリカバリ処理を走らせるかを確認

    * ユーザが受け付けた場合はリカバリ処理を実行

* アプリケーションの終了が呼ばれるまでフレーム処理

  * 描画処理

    * シャドウマップを描画
    * エフェクトのオフスクリーンレンダーターゲットを描画
    * ビューポートを描画

      * ScriptExternal のためのモデルあるいはアクセサリの描画
      * 背景動画を描画
      * グリッドを描画
      * プリプロセスのエフェクトを描画
      * すべてのモデルのエッジを描画
      * すべてのモデル及びアクセサリの描画
      * すべてのモデル及びアクセサリの地面影を描画
      * ポストプロセスのエフェクトを描画
      * screen.bmp 専用のレンダーターゲットを転写

    * 描画コマンドの一括処理

      * 31.0 から導入
      * 詳細は「描画コマンドの一括処理」の項目にて

    * UI のアイコンなどを描画
    * UI (ImGui) を描画
    * ウィンドウに描画結果を表示
    * リセット処理が要求された場合はリセット処理を実行

      * 各種エフェクトのすべてのレンダーターゲットを再生成及び再設定
      * ビューポートのレンダーターゲットを再生成及び再設定

    * 2フレームに1回 UI スレッドにダミーのイベントを通知
    * 動画エンコード処理

      * 動画エンコード処理が実行中の場合のみ

    * プロジェクトの更新処理

      * 音源の位置を更新

        * 音源再生中の場合のみ

      * シーク処理

        * 物理演算前のモーションの適用処理

          * すべてのモデルに対して以下の順番で実行

            * モデル（表示や IK 有効無効の方）のキーフレームを適用
            * モデルの材質のリセット
            * モデルのボーン変形をリセット
            * モデルのモーフをリセット
            * モーフのキーフレームを適用
            * 物理演算適用前のボーンのキーフレームを適用
            * ボーンのキーフレーム単位の物理演算の有効無効の切り替え処理
            * 物理演算に適用するためのボーンのパラメータを設定

        * 物理演算の実行
        * 物理演算後のモーションの適用処理

          * すべてのモデルに対して物理演算適用後のボーンのキーフレームを適用
          * すべてのアクセサリに対してキーフレームを適用
          * カメラのキーフレームを適用
          * 照明のキーフレームを適用
          * セルフシャドウのキーフレームを適用

        * カメラの更新
        * 照明の更新

      * モデルの変形処理

  * 各種イベント処理

    * イベント処理中にエラーが発生したらエラーダイアログを表示

  * イベント処理中にリセット処理が要求された場合は再度リセット処理を実行

* アプリケーションの終了

  * プロジェクトの破棄
  * 各種ライブラリの終了処理

主な要素
==========================================

アプリケーション
------------------------------------------

プロジェクト、描画、UI (ImGui)、入力のやり取りを一括管理するオブジェクト。
``emapp::BaseApplicationClient`` を通じて処理する。

スレッドに対応して UI 層とのやり取りの分離をはかる ``emapp::ThreadedApplicationService`` があり、
Windows 版及び macOS 版ではこちらを利用する。 ``emapp::ThreadedApplicationClient``　を通じて処理する。

``emapp::BaseApplicationService`` が対応する

プロジェクト
------------------------------------------

すべてのモデル、モーション、アクセサリ、エフェクトを包括管理するオブジェクト。
オフスクリーンを含めた全てのレンダーターゲットの描画及び破棄もここで行っている。

``emapp::Project`` が対応する

モデル
------------------------------------------

PMD/PMX に対応する描画対象オブジェクト。

``emapp::Model`` が対応する

アクセサリ
------------------------------------------

X に対応する描画対象オブジェクト。

``emapp::Accessory`` が対応する

モーション
------------------------------------------

動きを定義するオブジェクト。以下の種類があり、この内ひとつのみに所属する。

* モデル
* アクセサリ
* カメラ
* 照明
* セルフシャドウ

``emapp::Motion`` が対応する

カメラ
------------------------------------------

``emapp::ICamera`` が対応する

照明
------------------------------------------

唯一の大域光源。 MMD の仕様にあわせてディレクショナルライトのみ。

``emapp::ILight`` が対応する

エフェクト
------------------------------------------

MME 互換の複数のテクニック及びパスから構成されるオブジェクト。
MME の技術仕様は MME に同梱している ``REFERENCE.txt`` を参照。

エフェクトの仕様が複雑でかつ他のオブジェクトにもかなり食い込んでるため、読解難易度は最も高いとみています。

``emapp::IEffect`` が対応する

テクニック
------------------------------------------

描画するための条件定義。複数のパスから構成される。

``emapp::effect::Technique`` が対応する

パス
------------------------------------------

頂点シェーダとピクセルシェーダをセットにした描画単位。

``emapp::effect::Pass`` が対応する

.. _1BF7070C-25E0-4E04-B314-6C67FE55E6AB:

コマンド
------------------------------------------

巻き戻しが可能な操作単位。undo.c のユーザデータとして持っており、undo.c のコールバックを通じて巻き戻しあるいはやり直しが実行される。
また、undo.c の永続化の仕組みを利用してアプリケーションがクラッシュしたときに直前の操作まで巻き戻す仕組みもコマンドが持っている。

キーフレーム登録あるいは削除、ボーンの移動やモーフの変更などの主要な操作はコマンドを通じて実行される。

複数のコマンドをひとつにまとめることが可能なバッチコマンドもあり、大規模なキーフレーム変更が発生するコマンドで利用している。

描画コマンドの一括処理
==========================================

30.0 までは材質またはレンダーターゲット単位にレンダーパスを発行する処理になっていましたが Apple Silicon 対応において描画が崩れる問題があったため、
レンダーターゲット単位にレンダーパスをまとめるように描画処理の見直しを実施しました。これにより無駄なレンダーパスを作らせないようにしたためパフォーマンス改善を可能になりました。

* ``Project::SerialDrawQueue``

  * 原則として毎回レンダーパスを発行する
  * 例外として前回が SerialDrawQueue でかつ同じレンダーパスの場合マージ可能ならマージする
  * 主にポストエフェクトで利用

* ``Project::BatchDrawQueue``

  * レンダーパス単位にまとめる
  * 従来の描画は基本的にこちらを利用

描画コマンドの一括処理オブジェクトの管理は Project にあるものの、インターフェースである ``emapp::sg::PassBlock::IDrawQueue`` を経由するため中身は直接公開していません。

音源の同期補正処理
==========================================

音源の位置とクロックオフセットを比較し、レイテンシの小さいほうを優先する同期補正処理が Windows 版と macOS 版に実装されています。具体的な流れは以下の通りです。

* 音源の位置をサンプルオフセットとして計算して取得
* クロックを秒に変換して音源の周波数と乗算しサンプルオフセットとして比較
* 上記二つの差分をレイテンシとして取得し、しきい値 (音源の周波数を 60FPS 基底に Windows 版では 15FPS macOS 版では 10FPS 相当で計算) で比較

  * レイテンシがしきい値よりも小さい場合はクロックを採用
  * レイテンシがしきい値よりも大きい場合は音源の位置を採用

通常はクロックを採用するものの、フレーム落ちにより処理が追いつかなくなった場合は強制的に音源の位置を採用する仕組みとなっています。

emapp の実装ポリシー
==========================================

* スペースのみ、インデントは 4

  * ``.editorconfig`` で定義

* C++ の例外は使用禁止
* C++ の実行時型情報 (RTTI) は使用禁止
* C++11 の ``nullptr`` 以外は使用禁止

  * ただし UI 層は例外的に C++11 の使用が認められる
  * UI 層でも C++14 以降の機能利用は認められない

* STL は原則として利用しない

  * かわりに同梱の `TinySTL <https://github.com/mendsley/tinystl>`_ を利用する
  * ただし UI 層では一部利用 (``std::atomic``) している

* 名前付けは Qt/WebKit をベースにしたカスタム

  * クラス名は ``UpperCamelCase``
  * メソッド名は ``lowerCamelCase``
  * 定数は k を頭につけて ``UpperCamelCase``
  * メンバー変数は原則として ``m_`` 接頭詞がつく
  * protobuf のような自動生成によるものは適用対象外

単体テスト
==========================================

`Catch2 <https://github.com/catchorg/Catch2>`_ を利用した単体テストが ``nanoem/test`` 及び ``emapp/test`` にあります。
リリース前は必ず全てのテストをパスする必要があります。

fx9
==========================================

エフェクトをコンパイルするために作られた内製ライブラリです。fx9 はエフェクトプラグインを通じて利用されます。

* 出力するシェーダ言語を設定する。以下から設定可能

  * GLSL
  * MSL
  * HLSL
  * SPIR-V

* エフェクトのソースを入力
* AST に変換して SPIR-V 形式にコンパイル

  * fx9 がやることは文法をパースして AST に変換すること
  * 字句解析及び AST は `glslang <https://github.com/KhronosGroup/glslang>`_ が提供するものを利用する

    * fx9 が実装しているのは DirectX のエフェクトの文法を解析処理と AST への変換処理である
    * 文法解析のバックエンドは `Lemon <https://www.sqlite.org/lemon.html>`_ を利用している

* 最適化が有効の場合は `SPIRV-Tools <https://github.com/KhronosGroup/SPIRV-Tools>`_ でシェーダを最適化する
* 出力するシェーダ言語に応じて `SPIRV-Cross <https://github.com/KhronosGroup/SPIRV-Cross>`_ で変換
* fx9 独自の protobuf 形式のバイナリデータで出力

  * データ仕様は ``emapp/resources/protobuf/effect.proto`` で定義
  * protobuf が emapp 上にあるため、独立したライブラリとしてまだ完全に分離できてない状態

nanodxm
==========================================

DirectX の .x 形式のテキストデータをパースするために作られた内製ライブラリです。バイナリは未対応です。

独立したライブラリとして一応使うことが可能です。

undo.c
==========================================

emapp で使われている undo/redo の操作に特化した内製ライブラリです。
クラッシュ後の起動時に行われるリカバリ処理を実現するためにコールバックを通じた永続化にも対応しています。

独立したライブラリとして一応使うことが可能です。

sokol
==========================================

https://github.com/floooh/sokol (実際にはフォーク版 https://github.com/hkrn/sokol を利用)

nanoem の描画バックエンドとして利用しているライブラリです。

基本的にはオリジナルの実装をそのまま利用しますが、デバッグやバッファの読み取りのために内部構造に直接アクセスして拡張しています。
また、複数のバックエンドを切り替えられるようにするため、共有ライブラリとして組み込んでいます。

最初期は `bgfx <https://github.com/bkaradzic/bgfx/>`_ を利用していましたが、以下の理由から切り替えを行っています。ただし、関連ライブラリである bx および bimg は引き続き利用しています。

* バッファの明示的な上限設定が必要
* 内部構造上デバッグが困難
* バックエンドのオブジェクトへの直接アクセスができなかった

ImGui
==========================================

https://github.com/ocornut/imgui

nanoem の GUI バックエンドとして利用しているライブラリです。

主にゲーム開発における GUI ライブラリとして利用されますが、nanoem では直接ユーザが利用する GUI ライブラリとして利用しています。
初期から利用しておりその時はデフォルトのルックフィールを利用していましたが、 `nuklear <https://github.com/vurtun/nuklear>`_ を一時期に採用してた関係から nuklear の見た目と合わせる形でルックフィールを変更しています。

デバッグ表示の可視化
==========================================

Visual Studio を利用している場合は以下に natvis のファイルがあるので、それらを ``%UserProfile%Visual Studio XXXX\Visualizers`` に配置することでデバッグ時に表示を可視化できます。

* dependencies/bx/scripts/bx.natvis
* dependencies/bx/scripts/tinystl.natvis
* dependencies/glm/util/glm.natvis
* dependencies/imgui/misc/debuggers/imgui.natvis
* scripts/nanoem.nativs

また QtCreator を利用している場合は ``scripts/qtcreator/helper.py`` があるので https://doc.qt.io/qtcreator/creator-debugging-helpers.html の ``Extra Debugging Helpers`` に該当パスを指定することによりデバッグ時に表示を可視化できます。

.. _2712B38B-9A84-43A2-B903-FB390383054C:

よろずのおはなし
******************************************

MMD は DirectX11 に移行できるのか？
==========================================

まずこの質問の背景として2021年1月末にセキュリティ上の理由で SHA1 署名のエンドユーザ向け DirectX9 のインストーラ及び「古い」DirectX9 SDK が Microsoft 公式ページから削除されました（`英語での解説記事 <https://walbourn.github.io/where-is-the-directx-sdk-2021-edition/>`_。もともと計画されてたもので時勢により延期されたため復活はないと見られましたが、少なくとも `英語版 <https://www.microsoft.com/en-us/download/details.aspx?id=35>`_ は SHA2 に署名し直して再開したようです）。

一時期上記の混乱が発生したため MMD の公式ページから再配布可能なエンドユーザ向け DirectX9 インストーラを直接ダウンロードできる仕組みを取って対応しました。そのことからおそらくこの疑問が出るだろうと見て記しておきます。

.. note::
   意外に思われるかもしれませんが DirectX9 ランタイムそのものは Windows10 に含まれており、保守も続けられています。
   
   一方で DirectX9 を利用する際のデファクトスタンダードであり MMD も利用しているライブラリである D3DX が Windows 10 に入ってないことから DirectX9 インストーラによる導入が必要となります [#f1]_ [#f2]_。

   D3DX は歴史的経緯によりバージョン違いが多数あり適切に導入するのが非常に難しいことからインストーラ経由による導入が必須であり、D3DX 単体の再配布禁止の根拠となっています [#f2]_。

以下の理由から少なくとも本体自身からはおそらく不可能と見ています。また、仮に運良く移行できたとしても DirectX9 のそれとは別物になる可能性が高いため、利用者が使ってくれるかどうかが未知数です。

* DirectX9 と DirectX11 とでは設計レベルで異なるため DirectX9 から DirectX11 に移行するには設計変更が必要

  * さらに MMD では D3DX を利用しているが DirectX11 では直接同等の機能を提供しておらず [#f3]_、独自のものに置き換える必要がある

* MME で利用しているエフェクト文法が DirectX11 では互換性がないため利用できない

  * DirectX11 に対応したエフェクトを改めて書き起こす必要がある
  * DirectX のエフェクト形式そのものが非推奨のため今後利用できるかどうかが不透明

* そもそも MMD/MME 自体の開発が事実上停滞している状況にある

ただし MMD 本体からではなく MME の仕組みを利用して DirectX11 に対応させる取り組みはいくつかあります。

* MME と同じ仕組みで DirectX11 に対応させる MME の開発者自身による取り組み

  * 動画は `sm21860058 <https://www.nicovideo.jp/watch/sm21860058>`_
  * あくまで DirectX11 の新機能を試すための試験的な取り組みからか、2021年2月時点でそれ以上の動きはないようです

* 別の開発者による MME と同じ仕組みで DirectX11 に対応させる取り組み

  * 動画は `sm35941062 <https://www.nicovideo.jp/watch/sm35941062>`_
  * 同開発者による MMDPlugin の仕組みを利用する形
  * エフェクトにも対応しているように見えるが HLSL Shader Model 5.0 ベースに MME の構文を適用できるようにした独自形式の様子

MMD は x86 以外に移行できるのか？
==========================================

先に書いたとおり MMD は D3DX に依存しており、D3DX がクローズドソースであるため D3DX 利用を脱却するか D3DX が x86 以外にも対応しない限りは不可能とみられます。
DirectX9 版の D3DX はすでに保守対象から外れており移行推奨の記事 [#f3]_ があるくらいなので x86 以外の対応は絶望的と考えられます。

少なくとも ARM 版は x86 エミュレーションで対応できるとみられますが、性質上速度低下は避けられず特に物理演算がボトルネックになる可能性があると考えられます。

.. [#f1] https://walbourn.github.io/where-is-the-directx-sdk-2013-edition/
.. [#f2] https://support.steampowered.com/kb_article.php?ref=9974-PAXN-6252
.. [#f3] https://walbourn.github.io/living-without-d3dx/
