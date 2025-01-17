==========================================
よくある質問と回答 (エフェクト編)
==========================================

.. important::
   macOS での OpenGL 利用時に発生したエフェクトの不具合はサポート対象外です（macOS 10.13 未満か明示的にレンダラを OpenGL にしている場合に該当）。
   Metal に切り替えてから利用してください。

エフェクトの読み込みはできますか？
============================================================

できます。モデルまたはアクセサリと同時に読み込まれます。詳しくは :ref:`3F20FD13-9F9B-49FD-9072-0DE3FE50CE58` または
:ref:`14C11FDE-A0FC-4415-A408-383B0132F735` を参照してください。

ポストエフェクト系やあまり複雑でないエフェクトであればだいたい読み込めますが、ハードウェア（特に GPU）の環境差異があることから MME と同じ結果になることは保証していません。

.. _596AB6F3-51F6-4C4C-8A0A-5428B6381499:

マゼンタ色みたいな表示になった
============================================================

以下の条件をすべて満たすとビューポート画面がマゼンタ色になって正しく表示されない現象が発生します。ほかの GPU では同様の現象が発生しないため Apple Silicon GPU 特有の問題と考えられます。

* Apple Silicon を搭載している Mac を利用している
* 「:ref:`6D009308-F906-4BFB-B118-17DB0B526DA0`」を有効にしている

  * 動画書き出しのアンチエイリアス設定も同様
  * 無効にした場合は発生しない

* 透過度を重ね塗りする形のエフェクトを利用している

  * 具体例のひとつとして XDOF が該当するがそれ以外も当然存在する

上記に当てはまってマゼンタ色になってしまった場合は以下の対応を行ってください。

* 「:ref:`6D009308-F906-4BFB-B118-17DB0B526DA0`」を無効にする
* アンチエイリアス設定のかわりに以下に紹介するようなアンチエイリアスを行うポストエフェクトを利用する

  * `o_DLAA <https://okoneya.jp/mmd_files/#o_DLAA>`_
  * `FXAA <https://github.com/MikuMikuShaders/FXAA>`_
  * `SMAA <https://github.com/MikuMikuShaders/SMAA>`_

.. _986802EC-851B-46B8-A7D0-287AA1294F0E:

エフェクト詰め合わせが見当たらないのですが...
============================================================

.. note::
   エフェクト詰め合わせはエフェクトプラグインを実装する前に利用していた nanoem でのみ利用可能なエフェクトのパッケージです。
   OpenGL 上でしか利用できず、かつバイナリ形式での提供のため改変が出来ない問題を抱えていたため現在は提供を終了しています。

エフェクト詰め合わせは 2020/8/31 をもって配布を終了しました。保守も行っていないため、以下のオリジナルの方を利用してください。

- otamon さんの `おこねや <https://okoneya.jp/mmd_files/>`_
- sovoro さんの `エフェクト集置き場 <https://onedrive.live.com/?id=EF581C37A4524EDA%21108&cid=EF581C37A4524EDA>`_

ray-mmd は使えますか？
============================================================

できます。ただし条件付きです。

.. warning::
   画像または動画として書き出すときはアンチエイリアスを無効にしてください。

.. warning::
   高解像度で利用する場合は iMac Pro または `外付け GPU を利用した Mac <https://support.apple.com/ja-jp/HT208544>`_ が事実上必須です。
   （参考程度に `MBP 13 インチ 2018 年モデル <https://support.apple.com/kb/SP775>`_ 上で外付け GPU なしかつウィンドウサイズ変更なしの状態だと 20FPS 未満）

   また、低解像度に切り替えて実行する場合でも GPU のある iMac または MacBook Pro 15inch での利用が望ましいです。

.. note::
   技術的な仕様による問題のため OpenGL 上では正常に動作しません

1.22.0 から暫定的に利用できるようになっていますが、そのままでは読み込むことができないのでファイルの改変が必要です。
macOS の場合レンダラを Metal に切り替える必要があるので「設定」よりレンダラを Metal に切り替えてください。
（Metal 非対応の場合は切り替え不可）

注意書きにあるとおり ray-mmd は特性上、性能要件が非常に高いのでお使いのマシンを確認してください。
要件が外れてる場合 nanoem が非常に重くなりまともに操作できなくなって再起動せざるを得なくなるおそれがあります。

改変場所
-------------------------------------------------------

.. important::
   ray-mmd に限らない話ですがエフェクトを改変するときは元に戻せるようにバックアップを取ってください。
   また、macOS の場合テキストエディタで編集すると改行部分が正しく保存されず読み込み時にエラーになってしまいます。

以下のファイルを `Visual Studio Code <https://azure.microsoft.com/products/visual-studio-code/>`_ のようなプログラミング用のエディタなどで ``FOG_ENABLE`` の値を 1 から 0 に修正したあと ray-mmd の読み込みとエフェクトの割当調整を行ってください。

.. code-block:: none
   :caption: ray.conf

   #define FOG_ENABLE 0

1.22.3 未満の場合は不具合によりさらに以下の ``Sky*box*`` を ``sky*box*`` に小文字に変更する3箇所の追加の改修が必要です
（1.22.3 以降の場合は不要です）。

.. code-block:: none
   :caption: Shader/textures.fxsub

   "sky*box*.* =./Skybox/skylighting_none.fx;"

ikPolishShader は使えますか？
============================================================

.. warning::
   重さは ray-mmd と同等あるいはそれ以上です

28.0 から利用可能です。ただしいくつか注意があります

* ikPolishShader v0.26 では高品質 (2) まで対応

  * デフォルトのカスタム設定 3 はコンパイルが通らない

* ikPolishShader v0.27 では低品質 (1) まで対応

  * 普通 (2) は表示上の問題があり
  * 高品質 (3) またはデフォルトのカスタム設定 (0) は落ちる

0.26 と 0.27 とでは表示上互換性のない変更があるため両方確認しています

MotionBlur 系が動かないのですが...
============================================================

一部の改変が必要です。これは nanoem が利用する描画バックエンドと MMD の描画バックエンド (DirectX9) のラスタライズの仕様の違いのためです。 [#f1]_

.. caution::
   28.1 以前では下記にある改変を行ってもふたつ以上のモデルがある状態で MotionBlur を利用すると正しく機能しない不具合があります。
   この問題については 28.2 で修正されています。

一例としてそぼろさんの MotionBlur2 の場合は ``VelocityMap.fx`` を編集し、「ここから追加」の行から「ここまで」の部分を追加すると正しく機能するようになります。

.. code-block:: none
   :caption: VelocityMap.fx:343

   Out.Pos.xy = (tpos * 2 - 1) * float2(1,-1);
   Out.Pos.zw = float2(0, 1);

   // ここから追加
   #if defined(NANOEM)
   Out.Pos.x += VPBufOffset;
   Out.Pos.y -= VPBufOffset;
   #endif
   // ここまで

また、同じくモーションブラーを利用する TrueCamera/TrueCameraLX についてもファイルが
``TrueCameraObject.fx`` または ``TCLX_Object.fxsub`` で改変する行の位置に違いがありますが、改変内容は同じです。

``'#' : invalid directive`` が出る
============================================================

これは以下のようなコードを利用していると未実装のために発生します。

.. code-block:: none

  #define some_macro(n) replaced_result_##n

文字列結合と呼ばれる処理のため、上記の ``define`` の行を削除し、例えば以下のように使われている場合は

.. code-block:: none

  some_macro(test)

文字列を置き換えた結果を使用箇所全てに適用してください。

.. code-block:: none

  replaced_result_test

詳しくは `トークン連結演算子 <https://docs.microsoft.com/ja-jp/cpp/preprocessor/token-pasting-operator-hash-hash>`_ を参照してください。

画面が固まったかのような表示になる
============================================================

一部エフェクトでビューポート切替時にクリア処理がないためビューポートが固まったかのような表示になることがあります。回避策としてクリア処理の追加が必要です。

当該問題を確認している `DropShadow <http://www.nicovideo.jp/watch/sm19160219>`_ の場合は以下の改変が必要です。

.. code-block:: none
   :caption: DropShadow.fx:212

   "RenderColorTarget0=;"
       "RenderDepthStencilTarget=;"
       // ここから追加
       #if defined(NANOEM)
       "ClearSetColor=ClearColor;"
       "ClearSetDepth=ClearDepth;"
       "Clear=Color;"
       "Clear=Depth;"
       #endif
       // ここまで
       "Pass=Gaussian_Y;"

nanoem 上で動いてると MME からどう判断すればよいですか？
============================================================

``NANOEM`` マクロが予め定義されているので、それの有無で判断できます。

また、nanoem では動いているレンダラにあわせて MME から変換する都合上、変換先のシェーダ形式を示すマクロが定義されています。

これらはいずれも定義された上で変換先の場合は1を、変換先ではない場合は0を示す値が入ってるため、
定義の有無だけではなく数値の値も判断する必要があります。

.. csv-table::

  マクロ名,対応するレンダラ,変換先のシェーダ形式（言語）
  ``NANOEM_OUTPUT_SHADER_LANGUAGE_GLSL``,OpenGL,GLSL
  ``NANOEM_OUTPUT_SHADER_LANGUAGE_ESSL``,OpenGL (ES),GLSL
  ``NANOEM_OUTPUT_SHADER_LANGUAGE_HLSL``,DirectX,HLSL
  ``NANOEM_OUTPUT_SHADER_LANGUAGE_MSL``,Metal,MSL
  ``NANOEM_OUTPUT_SHADER_LANGUAGE_SPIRV``,(将来予約用),SPIR-V

.. [#f1] 技術的な話として nanoem では ``Draw=Buffer`` の場合頂点シェーダにわたす前に予めサブテクセルのズレを意図的に起こして頂点シェーダによる処理によりゼロサムにして差異を吸収する仕組みを持ってますが、モーションブラーなどで使われる頂点テクスチャフェッチのようなケースの場合は例外のため改変が必要です。
