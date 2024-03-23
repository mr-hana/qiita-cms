---
title: Fluid Frameworkを使って共同編集可能なお絵描きアプリに挑戦する
tags:
  - TypeScript
  - React
  - FluidFramework
private: false
updated_at: '2021-12-01T07:00:39+09:00'
id: 7de0826cffabe9ad2e96
organization_url_name: systemi
slide: false
ignorePublish: false
---
最近Teamsで共有されたファイルを直接編集することが多く、その際の共同編集体験が良かったので調べてみました。
どうやらFluid Frameworkというマイクロソフト謹製のライブラリーが用いられているとのことです。こちら[オープンソース化](https://github.com/microsoft/fluidframework)もされています。
[公式サイト](https://fluidframework.com/)では付箋アプリなどおもしろいサンプルも提供しています。

難しいこともいろいろと書いてあるので手を動かして体験してみようと思います。
テキストの共有はサンプルもあったので画像にしてみようかと共同編集可能なお絵描きアプリに挑戦してみます。

## Fluid Framework

その前に簡単にFluid Frameworkとは。

> **What is Fluid Framework?**
> Fluid Framework is a collection of client libraries for distributing and synchronizing shared state. These libraries allow multiple clients to simultaneously create and operate on shared data structures using coding patterns similar to those used to work with local data

> 引用（https://fluidframework.com/docs/#what-is-fluid-framework）

一言でいうと各クライアントの状態を共有、同期するためのクライアントライブラリーです。
特徴的な点はクライアントの状態を共有してマージするのは各クライアントで行う点です。

想定されている流れは以下の通りです。

> **The following is a typical flow.**
> 
> - Client code changes data locally.
> - Fluid runtime sends that change to the Fluid service.
> - Fluid service sequences that operation and broadcasts it to all clients.
> - Fluid runtime incorporates that operation into local data and raises a “valueChanged” event.
> - Client code handles that event (updates view, runs business logic).

> 引用（https://fluidframework.com/docs/#how-fluid-works）

大きく分けてFluid ContainerとFluid Serviceのふたつの要素で構成されています。
Fluid Containerはローカルの編集状態をFluid Serviceに送り、Fluid Serviceはシーケンス化した操作を各クライアントへ配布します。
Fluid Frameworkはサーバーでは操作のシーケンスを配布するだけで、状態のマージは各クライアントが行います。その結果遅延時間を劇的に削減しているとのことです。

他にも興味深い内容が書かれていますので公式サイトをぜひご覧ください。

https://fluidframework.com/docs/concepts/architecture/

https://fluidframework.com/docs/data-structures/overview/

## お絵描きアプリ開発

今回は下記を参考にReactで作成します。

https://fluidframework.com/docs/recipes/react/

### 環境

```console
node -v
> v16.7.0

yarn -v
> 1.22.11

npx create-react-app --version
> 4.0.3
```

### Reactの構築

```console
npx create-react-app fluid-canvas --template typescript
cd fluid-canvas
```

typescriptのバージョンは4.1.2です。

### Fluid Framework インストール

```console
yarn add @fluidframework/tinylicious-client fluid-framework
```
バージョンは0.52.1です。

#### fluid-framework
Fluid Framework本体。クライアント間のデータを同期するためのライブラリーです。

#### @fluidframework/tinylicious-client
Tinyliciousは開発目的のローカルのインメモリーを用いたFluidサービスです。
Tinylicious ClientはTinyliciousへの接続やFluid Containerのスキーマを定義します。
本来はFluid Loaderや接続手順、データ取得の手続きなどを構築しないといけなさそうなんですがカプセル化してくれています。

https://fluidframework.com/docs/testing/tinylicious/

Fluidサービスは他にもAzure Fluid Relay用や独自構築できるRouterliciousが提供されています。

### react-signature-pad-wrapper インストール

```console
yarn add react-signature-pad-wrapper
```

描画には[Signature Pad](https://github.com/szimek/signature_pad)をReact用にラッパーした[react-signature-pad-wrapper](https://github.com/michaeldzjap/react-signature-pad-wrapper)(v2.0.2)を使用します。
Signature Padはスムーズな署名を描くためのライブラリーと謳っており、シンプルでアウトプットもPNG, JPEG, SVGに変換できて使い勝手が良いです。

### お絵描き部分作成

```diff_tsx:App.tsx
import React from 'react';
- import logo from './logo.svg';
- import './App.css';
+ import SignaturePad from "react-signature-pad-wrapper"

function App() {
+  const signaturePadRef = React.useRef<SignaturePad>(null);

+  return (
+    <SignaturePad ref={signaturePadRef} />
+  );

-  return (
-    <div className="App">
-      <header className="App-header">
-        <img src={logo} className="App-logo" alt="logo" />
-        <p>
-          Edit <code>src/App.tsx</code> and save to reload.
-        </p>
-        <a
-          className="App-link"
-          href="https://reactjs.org"
-          target="_blank"
-          rel="noopener noreferrer"
-        >
-         Learn React
-       </a>
-     </header>
-   </div>
- );
}

export default App;
```

signaturePadRef は描画した絵を取得、更新する際に使用します。ついでにlogo.svgなどは削除しました。
起動して動作確認します。

```
yarn start
```

![animation.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/114149/4e789c2a-eb87-573e-62bd-aa4215019d4f.gif)

マウスドラッグで絵が描けるようになりました。
本家ではクリアや色変更などのデモもあります。もっとリッチにできますが今回はこのまま進めます。

### Fluid Framework導入
```diff_tsx:App.tsx
import React from "react";
import SignaturePad from "react-signature-pad-wrapper"
+ import { TinyliciousClient } from "@fluidframework/tinylicious-client";
+ import { SharedMap } from "fluid-framework";
```
TinyliciousClient と SharedMap をimportします。SharedMap はFluid Frameworkが提供するDDS（distributed data structures）の一種で、Key-Valueデータを提供します。
DDSはローカルデータを扱うように操作して、各クライアント間で状態を共有することができます。

### コンテナーの生成または取得

```diff_tsx:App.tsx
import React from "react";
import SignaturePad from "react-signature-pad-wrapper"
import { TinyliciousClient } from "@fluidframework/tinylicious-client";
import { SharedMap } from "fluid-framework";

+ const dataKey = "drawing";
+ const containerSchema = {
+   initialObjects: { view: SharedMap }
+ };

+ const client = new TinyliciousClient();
+ const getViewData = async (): Promise<SharedMap> => {
+   let container;
+   const containerId = window.location.hash.substring(1);
+   if (!containerId) {
+     ({ container } = await client.createContainer(containerSchema));
+     const id = await container.attach();
+     window.location.hash = id;
+   } else {
+     ({ container } = await client.getContainer(containerId, containerSchema));
+   }

+   return container.initialObjects.view as SharedMap;
+ }

function App() {
  ...
}
```

dataKey はShardMapに設定するKeyです。

containerSchema はFluid Containerの定義です。initialObjectsはコンテナーの作成時に作成され、コンテナーが有効な間存在します。各クライアントはinitialObjects を介してアクセスし、分散された状態を共有します。
他にDynamic objects(dynamicObjectTypes)があります。Dynamic objectsはアプリで扱うデータサイズが大きいため遅延ロードしたり、必要なデータがユーザーの操作に依存する場合にUXを守るために利用するようです。

getViewData はコンテナーからShardMapを取得します。コンテナーが未作成の場合は新しくコンテナーを作成し、コンテナーIDをURLハッシュに設定します。URLハッシュにコンテナーIDが設定されている場合はコンテナーを取得します。
サンプルではURLハッシュへのアクセスが```location.hash```になっていますが、eslintのno-restricted-globalsに引っかかるので```window.location.hash```に変更しています。

### Fluidデータの取得

```diff_tsx:App.tsx
function App() {
  const signaturePadRef = React.useRef<SignaturePad>(null);

+  const [fluidData, setFluidData] = React.useState<SharedMap>();
+  React.useEffect(() => {
+    getViewData().then(view => setFluidData(view));
+  }, []);

  ...
}
```
useEffectでgetViewDataをコンポーネント生成時に１度だけ呼び出します（第２引数に空配列を設定）。
fluidDataにcontainerSchema で定義したview（ShardMap）を設定しています。

### Fluidデータの同期

```diff_tsx:App.tsx
function App() {
  ...

+  React.useEffect(() => {
+    if (!fluidData) {
+      return;
+    }
+
+    const syncView = () => {
+      if (signaturePadRef.current) {
+        signaturePadRef.current.fromDataURL(fluidData.get(dataKey) as string);
+      }
+    }

+    syncView();
+    fluidData.on("valueChanged", syncView);
+    return () => { fluidData.off("valueChanged", syncView) }
+  }, [fluidData]);

  ...
}
```

useEffectをもうひとつ追加してFluidデータとUIを同期します。UIの更新処理はsyncViewにてfluidDataの値を取得して、Signature PadのfromDataURLに設定します。
また、Fluidデータは別クライアントが変更する可能性があります。Fluidデータに変更があるとvalueChanged イベントが発生するので、syncViewでUIの更新処理を行います。

### Fluidデータの更新

```diff_tsx:App.tsx
function App() {
  ...

+  const [imageData, setImageData] = React.useState<string>();
+  React.useEffect(() => {
+    if (imageData) {
+      fluidData?.set(dataKey, imageData);
+    }
+  }, [imageData, fluidData]);

+  const onEnd = React.useCallback(() => {
+    const signaturePad = signaturePadRef.current;
+    const dataUrl = signaturePad?.toDataURL("image/svg+xml");
+    setImageData(dataUrl);
+  }, [setImageData]);

  return (
-    <SignaturePad ref={signaturePadRef} />
+    <SignaturePad ref={signaturePadRef} options={{onEnd: onEnd}} />
  );
}
```

ローカル用にimageDataを用意します。imageDataに変更がある場合はfluidDataにも設定し、別クライアントへ変更を共有します。
onEnd はSignature Pad の描画終了時にコールされます[^1]。Signature PadのtoDataURLの値をimageDataに設定します。

## お絵描きアプリ実行

Fluidサービス（サーバー）としてtinylicious を起動します。

```console
npx tinylicious@latest
```

アプリを実行します。

```console
yarn start
```

![animation.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/114149/4d6b866c-11f3-0a6e-5af5-de2395f80b9e.gif)

２つのブラウザから同じURLにアクセスして描画が同期しています。

### 複数端末からアクセスしてみる

ただし、tinylicious はローカルでしかアクセスできないので別端末からもアクセスできるようにしてみます。公式にてngrok を使ってみよ、とあるのでやってみます。
アカウント登録や認証用設定が必要なので公式の手順を参照ください。

https://fluidframework.com/docs/testing/tinylicious/#testing-with-tinylicious-and-multiple-clients

```ngrok http```実行後にForwardingに表示される転送用のドメインをTinyliciousClientProps のconnection のdomain に設定します。設定したTinyliciousClientProps をTinyliciousClient の引数にします。

```diff_tsx:App.tsx
+ const clientProps: TinyliciousClientProps = {
+   connection: { port: 443, domain: "https://forwarding-domain.ngrok.io" }
+ }

- const client = new TinyliciousClient();
+ const client = new TinyliciousClient(clientProps);
```

httpで設定する場合はportを80にしてください。

```console
npx tinylicious@latest
ngrok http 7070
yarn start
```
tinyliciousのPortのデフォルトは7070です。

![animation.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/114149/cbbdf5d6-da11-03c9-ad91-a920f4e63818.gif)

わかりづらいですが２つの端末から描画しています。

## まとめ
クライアントロジックだけで共同編集機能を実現しているFluid Frameworkはとても強力です。
今はバージョン1ではなく、「まだ製品品質のソリューションを提供できる状態ではありません」とのことなので今後が楽しみです。

今回作成したソースは[こちら](https://github.com/mr-hana/fluid-canvas)です。

[^1]: 本記事を書いている最中にSignature Padが３年ぶりのメジャーアップデートをしたようでonEndがなくなっていました。react-signature-pad-wrapperの方もそのうち対応するかもしれません。
