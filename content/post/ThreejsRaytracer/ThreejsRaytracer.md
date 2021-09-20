---
title: "ThreejsでPathtracerを実装してみた"
date: 2021-09-17T22:27:11+09:00
archives: "2021"
tags: [raytracing]
draft : false
---

# 目次
- Web上で動くパストレーサー
- ThreeJsでのダブルバッファリング
- OrbitCameraをshaderで使う
- 薄レンズモデル
- HDRIの実装

# Web上で動くパストレーサー
今まで私はShaderToy上でglslを書き、GPUパストレーサーを実装していたのですがやはり色々と追加したいことが増えてきたので他の方法でGPUパストレーサーを実装してみようかと考え、どんなものがあるかとGPUパストレーサーについて調べていました。

そんな時、かの有名な@gam0022さんが書いたWeb上で動くパストレーサーの記事を発見しました。

[WebGL+GLSLによる超高速なパストレーシング](https://qiita.com/gam0022/items/18bb3612d7bdb6f4360a)

この記事は素晴らしく、どんな感じにバッファリングをするのかなど貴重な情報が書かれており、パストレーサーそのものなら何とか行けそうと思い立つことができました。

というわけで実装を行ったのですが、Threejsの仕様をちゃんと理解していないため、そのままやったはずなのに動かないなど四苦八苦することとなりました。
苦節１か月、何とか見れる程度まで構築することが出来ました。以下のリンクから製作したWebパストレーサーを見ることができます。

※　現在まだ開発中のため内容が変わっている場合があります。　また、環境によってはなぜか４分割されるような見た目になっていることがあります。

https://kinakomoti-321.github.io/WebPathtracer/

{{< figure src="../pic1.png" class="center">}}

この記事では備忘録がてらWebパストレーサーをどのように実装したかを幾つか書いていこうと思います。この記事で書かれる実装は初心者が取り合えずできればいいだろという最適化のさも考えずに作った違法建築みたいなものなので参考程度にしていただけると幸いです。



# ThreeJsでのダブルバッファリング
パストレーサーは基本的に何度も輝度を計算し、輝度の平均値を色として出す形で行います。そのため、「前回までの輝度の累計」という情報を持つバッファが必要となってきます。

ThreeJsではこうした前回の描画情報を持たすことが可能なWebGLRenderTargetと呼ばれるものがあります。レンダリングした描画情報を入れるためのクラスであり、その情報をfloatTextureとして読みだすことができる。これを用いればパストレーサーの「前回までの輝度の累計」を保持することができます。

ただし、シェーダーにfloatTextureを読み込ませ、その結果を同じレンダーターゲットに書き込むということは出来ないらしく、読み込み用のレンダーターゲットと書き込み用のレンダーターゲットの２つを用意してお互いに交換し合ってバッファリングをするダブルバッファリングをする必要があります。

また、レンダーターゲットは先ほども言った通りレンダリングして画面に出てくる出力をテクスチャとして保存します。しかし、最終的な出力というのは輝度の平均値であり、これを保持しても意味がありません。そこで累計輝度を出力するシーンとその累計輝度の平均を出力するシーンで分け、前者をレンダーターゲットに保存する形で行います。

２つのシーンのレンダリングをしてもいいのかというのがあると思いますが、モニターに出さずに裏で計算だけ行うというオフスクリーンレンダリングというものがあるため累計輝度の計算シーンはこちらで行うことで処理ができます。実はレンダーターゲットをレンダリングに使うとそのままオフスクリーンレンダリングになります。

以上のような仕組みを使い、それぞれこのような処理を行います。

#### オフスクリーンレンダリング側
1. 板ポリにパストレーサーのシェーダーを付ける。
2. ReadBufferから前回の合計輝度を受け取り、今回の輝度を加算して出力する。
3. レンダリング結果をWriteBufferに書き込む。

#### スクリーンレンダリング側
1. WriteBufferから得たテクスチャをFlameで割り平均輝度を出すシェーダーを板ポリに付ける。
2. それを通常通りレンダリングする。（この出力がスクリーンで出てくる）
3. ReadBufferとWriteBufferの中身を交換して次のレンダリングに持ち越す。

まずはレンダーターゲットを作る処理についてはThreeJsからWebGLRenderTargetを呼び出して作ります。実装では@gam0022さんの記事の物をそのまま使わさせて頂きました。詳しい説明は@gam0022さんの記事をご参照ください。

```
	ReadBuffer = new THREE.WebGLRenderTarget(width,height,{
		wrapS: THREE.RepeatWrapping,
        	wrapT: THREE.RepeatWrapping,
        	minFilter: THREE.NearestFilter,
        	magFilter: THREE.NearestFilter,
        	format: THREE.RGBAFormat,
        	type: THREE.FloatType,
        	stencilBuffer: false,
        	depthBuffer: false
	});

	WriteBuffer = ReadBuffer.clone();
```
こうしてできたレンダーターゲットはシェーダーにfloatTextureとして情報を渡すことができます。シェーダーに渡す際は以下のコードで通常のTextureと同様に渡します。
```
//floatTextureを得る
	var tex = ReadBuffer.texture
```
輝度を加算していく処理はシェーダー側でこのテクスチャの値に今回の輝度の値を足して出力というような処理を書くことで作ることができます。

今回の場合、二つのシーンを用意するのでオフスクリーンレンダリングするシーン、表示するシーンそれぞれ作ります。そしてそれぞれにシェーダーを取り付けた板ポリを追加しておきます。

```
	//オフスクリーンレンダリングするシーン
	var scene_buffer = new THREE.Scene();
	//表示用のシーン
	var scene_render = new THREE.Scene();
	
	var geometry = new THREE.PlaneBUfferGeometry(10,10);

	//シェーダに渡す変数
	let uniforms_buffer = {
		...
		//バッファーのテクスチャを渡す
		buffer : {type:'f',value:ReadBuffer.texture},
		//現在フレーム数
		Flame : {value: flame},
		...
	}	
	//パストレーサーのシェーダーをマテリアルとして使う。
	const pathMaterial = new THREE.ShaderMaterial({
		uniform : uniforms_buffer,
		//シェーダーを取り付ける
                vertexShader: document.getElementById('vertexShader').textContent,
                fragmentShader: document.getElementById('PathTracer').textContent,
	})
	const plane_buffer = new THREE.Mesh(geometry,pathMaterial);
	scene_buffer.add(plane_buffer);

	//シェーダーに渡す変数
	let uniforms_render ={
		...
		//累計輝度を盛ってるバッファー
		buffer: {type: 'f',value : WriteBuffer.texture},
		Flame: {value: flame},
		...
	}
	const renderMaterial = new THREE.ShaderMaterial({
		uniform : uniforms_render,
		//平均輝度を求めるシェーダーを取り付ける。
                vertexShader: document.getElementById('vertexShader').textContent,
                fragmentShader: document.getElementById('Render').textContent,
	});

	const plane_render = new THREE.Mesh(geometry,renderMaterial);
	scene_render.add(plane_render);
```

このような形で２つのシーンを作り、毎フレーム呼び出す関数内で処理を書いていきます。オフスクリーンレンダリングで累計輝度を計算しWriteBufferにその描画情報を渡し、それを経由してスクリーンレンダリング側で平均輝度を出す形で表示します。また、毎フレーム更新が必要なUniform変数はレンダリング前に更新しておきます。最後に次のフレームのためReadBufferとWriteBufferの中身を入れ替えます。

```
	//フレームの更新
	flame += 1;

	//uniform変数の更新
	{
		...
		uniform_buffer.buffer.value = ReadBuffer.texture;
		uniform_buffer.Flame.value = flame;
		...
	}

	//オフスクリーンレンダリング
	//レンダーターゲットの設定
	renderer.setRenderTarget(writeBuffer);
	renderer.render(scene_buffer,camera);

	//uniform変数の更新
	{
		...
		uniform_render.buffer.value = WriteBuffer.texture;
		uniform_render.Flame.value = flame;
		...
	}
	//最終レンダリング
	//レンダーターゲットの初期化
	renderer.setRenderTarget(null);
	renderer.render(scene_render,camera);

	//バッファの交換
	var change = ReadBuffer;
	ReadBuffer = WriteBuffer;
	WriteBuffer = change;

```

このような形でパストレーサーのバッファリングをすることができ、パストレーサーを実装することができました。
{{<tweet 1421845387137077254>}}
# OrbitCameraをshaderで使う
以上の形でパストレーサーを実装しましたが、まだカメラの移動などが出来ない状態にあります。折角Threejsを使っているのでカメラの移動をOrbitCameraで制御できるようにしようと考えました。

レンダリング時は上のようにカメラを指定するため、カメラを複数用意することが可能です。なので直接レンダリングに使わないカメラとしてOrbitCameraを用意しておき、これの情報をShader側に渡すことでShader上のカメラとしてOrbitCameraを使うことができます。
今回の実装では薄レンズカメラを実装することも考えていたため、単純なカメラの位置とカメラの方向のみをOrbitCameraから得る形になりました。

通常のようにOrbitCameraを生成します
```
	//shaderに渡すカメラ
        shadercamera = new THREE.PerspectiveCamera(45, width / height, 1, 10000);
        shadercamera.position.set(0, 1, 8);

        shadercamera.lookAt(new THREE.Vector3(0.0, 0.0, 0.0))
        
	//OrbitCameraの生成
	var orbitControls = new OrbitControls(shadercamera, document.querySelector('#myCanvas'));
        orbitControls.enablePan = true;
        orbitControls.keyPanSpeed = 0.01;
        orbitControls.enableDamping = false;
        orbitControls.dampingFactor = 0.015;
        orbitControls.enableZoom = true;
        orbitControls.zoomSpeed = 1;
        orbitControls.rotateSpeed = 0.8;
        orbitControls.autoRotate = false;
        orbitControls.autoRotateSpeed = 0.0;
        orbitControls.target = new THREE.Vector3(0.0, 0.0, 0.0)
```

カメラのクラスには位置座標としてpositionを持っているのでこれを受け取り、カメラの位置座標としてShaderに渡します。カメラが向いている方向については直接得られないため、カメラのクオータニオンを受け取りそれを(0,0,-1)のベクトルに適応することでカメラの方向を得るようにしています。

```
	//カメラのクオータニオンと座標を得る
        var rotation = shadercamera.quaternion;
        var pos = shadercamera.position

	//カメラの方向ベクトルをクオータニオンから得る
        var cameraDir = new THREE.Vector3(0, 0, -1);
	cameraDir = cameraDir.applyQuaternion(shadercamera.quaternion);
```

これらは他のuniform変数と同様、毎フレーム更新しながらShader側に渡します。

ちなみにカメラが移動した際はレンダリングをやり直す処理が必要になってきます。この実装ではShader側でflameが1の時Bufferのを取らないことで処理していて、基本的に再レンダリングが必要な時はflameを1にする処理を書くことにしています。カメラが移動したことを感知する方法として前フレームにおけるカメラの行列との比較で行いました。

```
	//前回のカメラ行列との比較
	if(prev && !shadercamera.matrixWorld.equals(prev)){
		flame = 1;
	}
        prev = shadercamera.matrixWorld.clone();
```

以上のようにすることで、OrbitCameraをShader上で扱えるようになりました。やはりOrbitCameraが付くと非常に操作がやりやすく、ThreeJs上でやってよかったと思えました。

{{<tweet 1423271224080166912>}}

# 薄レンズモデル
カメラの実装ができ、様々なテストが非常に簡単にできるようになったので今までやったことがないことをしていこうということでカメラのレンズを実装することにしました。

簡単なカメラレンズのモデルとして薄レンズモデルというものがあります。これは厚さを考えない虫メガネのような凸レンズである。これの実装についてはこちらのサイトを参考にさせて頂きました。原理から実装まで事細かに書いてあるとてもわかりやすい記事であり、簡単にglslでも実装することが出来ました。

[レイトレース：薄いレンズのカメラ](https://t-pot.com/program/121_RayTraceThinLens/index.html)

大まかな処理としては
1. レンズ上の点をサンプリングする
2. サンプリングした点からピント方向へとカメラレイを作る
という形で行う。

コード例
```
Ray thinLensCamera(vec2 uv,vec3 atlook,vec3 camerapos){
    //レンズのパラメータ（shader外から持ってきた）
    float f = cameraLens.x;
    float F = cameraLens.y;
    float L = cameraLens.z;

    //各必要な値の計算
    float V = L * f / (L - f);
    
    vec3 up = vec3(0,1,0);
    vec3 cw = normalize(atlook);
    vec3 cu = normalize(cross(cw,up));
    vec3 cv = normalize(cross(cu,cw));
    
    vec3 X = camerapos + uv.x * cu + uv.y * cv;
    vec3 C = camerapos + cw * V;
    vec3 e = normalize(C - X);

    //レンズ上の位置をサンプリング
    //ランダム関数
    vec2 xi = hash23(vec3(iTime * uv.x, iTime * iTime * uv.y , iTime * iTime * iTime));
    xi = hash22(xi);
    float phi = 2.0 * M_PI * xi.x;
    float r = xi.y * f / (2.0 * F);
    vec3 S = C + r * cos(phi) * cu + r * sin(phi) * cv;
    
    //ピント位置の計算
    vec3 P = C + e * L / dot(e,cw);

    //レイ生成
    Ray camera;
    camera.origin = S;
    camera.direction = normalize(P - S);

    return camera;
}
```

ただし私が行った実装ではレンズのサンプリングが一様ではないことやパスの重みづけが正しくないなどの問題点が会ったりします。しかしながら、このような極簡単な薄レンズモデルでも被写界深度が再現でき、ピンホールカメラに比べると意外と写実的な絵を作ることが出来ます。

{{<tweet 1424411558092759040>}}

# HDRIの実装
ThreeJsでは簡単にHDRIをCubemapで使うことが出来るのですが、何故か私の環境では多くの記事があるCubeMapを使った方法がうまく行かなかったため、HDRIを普通のテクスチャのように読み込み、背景の実装を自ら行いました。

HDRIを読み込むライブラリとしてThreejsにはRGBELoader.jsという物があります。これは単純にHDRIを読み込み、ちゃんと輝度の値をfloatTextureとして変換してくれる機能を持ちます。特にテクスチャのローダーと変わりなく扱えます。

```
            var hdrloader = new RGBELoader;
            var tex5 = hdrloader.load("enviroment.hdr");

	    uniform_buffer = {
		    ...
		    HDRItex: {type:'f', value: tex5}
		    ...
	    }
```

背景はスカイスフィアで行いました。これの具体的な方法はこちらをご覧ください。

[レイトレで背景（スカイスフィア）を作る方法](https://kinakomoti-321.github.io/mochimochi3D/post/shereuv/test1/)

スカイスフィアに読み込んだHDRIを付ける形で、背景を作成しました。

# おわり
### 感想
今回の実装を通して、デバッグが難しいなどの点は見受けられましたが、Threejsを使うことでWebGLの環境整備が直ぐに整えられその他のライブラリを使って根本的な実装は比較的簡単に出来たと感じました。また、いくつかの機能はThreejsのライブラリに任せられることや修正がすぐに見えること、そして何より誰でも見れるように公開できることが特に良い点だと感じられます。datGUIも直ぐに導入できるため、BRDFや新しい機能の実装テストの場を作りたいとなった時に便利ではないかと思いました。

### 追加したい機能
まだまだこのパストレーサーに追加できる機能はあり
- ポリゴンを読み込んでレイトレする。(BVHなどの実装)
- Macrofacetの金属やDisneyBRDFなどの様々な質感の実装
- シーン情報を他のソフトで作って読み込ませる
といったことを追加していきたいです。