+++
title = "パスファインダで文字をスライス GUI付き+BugFix"
date = "2010-01-13T00:00:00+09:00"
tags = ["indesign", "scriptui"]
+++

InDesignにてテキストフレームの文字をグラフィック化して、パスファインダ（交差）を利用して分割して、お好みで変形マトリクスを利用して散らしたりしてみます。

### 上のパネル

- ［slice］で分割数を設定。縦横それぞれ1以上の整数値を入れてください。細分割しすぎるとエラーになります。
- 「分割サイズが小さい方で処理」チェックすると分割サイズが小さい正方形で分割します。
- 「元のテキストを削除」チェックすると処理後に元のテキストフレームを削除します。


### 下のパネル

- ［set random transform range］で分割後の変形・移動値を設定。
- 設定した数値のプラスマイナス値の範囲内（Scale以外）でランダムに処理します。
- デフォルトの「0, 0, 100, 0」で変形・移動は「無し」になりますのでお好みで。

![/images/2010/09/701-slice_gui.jpg](/images/2010/09/701-slice_gui.jpg)

以前、
[kamiseto](http://d.hatena.ne.jp/kamiseto/) さんに教えてもらったイベントのカリー化をスライダー回りの処理に使用してみます。

（とりあえず教えてもらったまま、動きがわかってきたので次回別件にいかせるようにがんばる。）

```js

/**
テキストを切り刻む with GUI
"sliced or chopped text objects "
使い方：
テキストフレームを選択して実行。
ダイアログに数値をいれて続行。
実行するとバラバラに分割したグループオブジェクトになります。
注意；
空のテキストフレームだとエラーになります。
スペースだけなどグラフィック化してもパスにならない文字もエラーになります。
あまり細かくすると変形時に境界線エラーになる場合があります。
動作確認：OS10.4.11 InDesign CS3
milligramme
www.milligramme.cc
*/
if(app.documents.length == 0){
  exit();
}
if(app.selection.length == 1 && app.selection[0].constructor.name == "TextFrame"){
  var docObj = app.documents[0];

  //============================================================
  //ScriptUI 設定パート
  var dlg = new Window ('dialog', "slice",[0,0,250,400]);
  with (dlg){
    center ();
    var pnl1 = add('panel', [10,10,240,150],"[devide]");
    with (pnl1){
      var wdv = add('edittext', [90,10,120,25],"2"); //よこわり
      var hdv = add('edittext', [20,40,50,55],"3"); //たてわり
      var dsm = add('checkbox', [20,80,230,100],"分割サイズが小さい方で処理");
      var lvo = add('checkbox', [20,100,230,130],"元のテキストを削除");
      dsm.value = true;
      lvo.value = false;
      add('panel', [60,30,150,70]);
    }
    var pnl2 = add('panel', [10,160,240,330],"[set random transform range]");
    with (pnl2){
      add('statictext',[20,20,60,50],"Shear");
      var slider1 = add('slider',[20,40,50,120],0,0,45);// シアー -45°〜45°
      add('statictext',[10,130,30,150],"±");
      var edit1 = add('edittext',[20,130,50,150], "0");
      var offSet = 50;
      add('statictext',[20+offSet,20,60+offSet,50],"Rotate");
      var slider2 = add('slider',[20+offSet,40,50+offSet,120],0,0,180);// 回転 -180°〜180°
      add('statictext',[10+offSet,130,30+offSet,150],"±");
      var edit2 = add('edittext',[20+offSet,130,50+offSet,150], "0");
      offSet = 100;
      add('statictext',[20+offSet,20,60+offSet,50],"Scale");
      var slider3 = add('slider',[20+offSet,40,50+offSet,120],100,100,400); // 拡大 100〜400%
      var edit3 = add('edittext',[20+offSet,130,50+offSet,150], "100");
      offSet = 150;
      var trasferRange = Math.round( 
        Math.max (
          docObj.documentPreferences.pageWidth/2, 
          docObj.documentPreferences.pageHeight/2 
          ));
      add('statictext',[20+offSet,20,60+offSet,50],"Move");
      var slider4 = add('slider',[20+offSet,40,50+offSet,120],0,0,trasferRange); // 移動 ドキュメントサイズの大きい方の半分
      add('statictext',[10+offSet,130,30+offSet,150],"±");
      var edit4 = add('edittext',[20+offSet,130,50+offSet,150], "0");
    }
    var cancelButton = add('button',[30,360,120,350],'cancel');
    var okButton = add('button',[130,360,210,350],'ok');
  }
  okButton.onClick = function(){
    dlg.close();
    okClick=true;
  };
  cancelButton.onClick=function(){
    dlg.close();
    okClick=false;
  };
  //スライダー部分のカリー化 (thanx kamiseto)
  Function.prototype.curry = function(){
    var slice = Array.prototype.slice;
    var args = slice.apply(arguments);
    var that = this;
    return function (){
      return that.apply(this, args.concat (slice.apply(arguments)));
    };
  };
  //イベントをまとめておく
  var Events = {
    //スライダーをドラッグした時の動作、連動してテキストボックス内も変化
    SliderOnChange : function(target){
      target.text = Math.round(this.value);
    },
    //テキストボックスに数値を入れた場合、スライダーに反映させる。
    //スライダーで決めた範囲外の数値をいれても、スライダーは範囲内におさまる。
    EditOnChange : function(target){
      target.value = this.text;
    }
  }
  //スライダーとエディットボックスのペア
  var P = [
    [slider1 , edit1],
    [slider2 , edit2],
    [slider3 , edit3],
    [slider4 , edit4]
    ];
  //イベントをカリー化しながら追加する
  while(X = P.shift()){
    //スライダーに対するイベント
    X[0].onChanging= Events.SliderOnChange.curry(X[1]);
    //エデットボックスに対するイベント
    X[1].onChange = Events.EditOnChange.curry(X[0]);
  }
  dlg.show ();

  if(okClick == true){
    var devW= parseInt (wdv.text, 10); //分割数の設定たて
    var devH= parseInt (hdv.text, 10); //分割数の設定よこ
    var fitToSmaller= dsm.value; //分割して大きさのちいさい方で処理する。
    var removeOrg= lvo.value; //元のテキストフレームを削除する。
    var mtxSh= slider1.value;
    var mtxRo= slider2.value;
    var mtxSc= slider3.value/100;
    var mtxTr= slider4.value;
  }
  else{
    exit();
  }

  //============================================================
  //ルーラーを一時的にスプレッドにする
  var rulerBk=docObj.viewPreferences.rulerOrigin;
  var rulerTemp=RulerOrigin.SPREAD_ORIGIN;
  docObj.viewPreferences.rulerOrigin=rulerTemp;
  //メイン処理へ
  main(docObj, devW, devH, fitToSmaller, removeOrg);
  //ルーラーを元に戻す
  docObj.viewPreferences.rulerOrigin=rulerBk;
}
else{
  alert("テキストフレームを１つだけ選択して実行してください。");
}

//メイン処理
function main(docObj, devW, devH, fitToSmaller, removeOrg){
  var pageObj=docObj.pages.itemByName(docObj.selection[0].parent.name)
  //テキストをグラフィック化（元テキストフレームは残す）
  var outTex=docObj.selection[0].createOutlines(false); //文字がないとここでエラー
  var outBon=outTex[0].visibleBounds;
  //グラフィック化後の［黒］対応でオーバープリントを解除しておく
  try{
    outTex[0].overprintFill=false;
  }catch(e){}
  try{
    outTex[0].overprintStroke=false
  }catch(e){}

  //ばらばらにしたテキストを入れておく配列
  var choppingArray=new Array();
  var tempDup, cuttingRectangle, intersecObj;
  if(outTex[0].pageItems.length > 1){ //multi lines text
    var multiLineOutTex=outTex[0].allPageItems;
    for(var ml=0; ml < multiLineOutTex.length; ml++){
      cuttingRectangle= sliceBounds (multiLineOutTex[ml], devW, devH, fitToSmaller); //Fn-1へ
      for(var ln=0; ln < cuttingRectangle.length; ln++){
        tempDup=multiLineOutTex[ml].duplicate ();
        try{
          intersecObj=cuttingRectangle[ln].intersectPath (tempDup);
          //intersecObj.fillColor="Cyan" //色塗り
          choppingArray.push(intersecObj);
        }catch(e){
          cuttingRectangle[ln].remove();
          tempDup.remove();
        }
      } //for ln
    } // for ml
  }
  else{ //one line text
    cuttingRectangle=sliceBounds (outTex[0], devW, devH, fitToSmaller); //Fn-1へ
    for(var ln=0; ln < cuttingRectangle.length; ln++){
      tempDup=outTex[0].duplicate ();
      try{
        intersecObj=cuttingRectangle[ln].intersectPath (tempDup);
        //intersecObj.fillColor="Magenta" //色塗り
        choppingArray.push(intersecObj);
      }catch(e){
        cuttingRectangle[ln].remove();
        tempDup.remove();
      }
    } //for ln
  }
  if(removeOrg == true){
    docObj.selection[0].remove();
  } //元のテキストフレームを削除する
  outTex[0].remove(); //バラバラ前のグラフィック化オブジェクトを削除する
  transformEachObj(choppingArray); //Fn-2へ。 バラしたオブジェクトを移動・変形しないなら、この行をコメントアウト
  pageObj.groups.add(choppingArray); //バラしたオブジェクトをグループ化する
}

//オブジェクトのサイズの四角形を指定数で分割する。Fn-1
function sliceBounds (sel, devW, devH, fitToSmaller){
  var svBon=sel.visibleBounds;
  var selW=svBon[3]-svBon[1];
  var selH=svBon[2]-svBon[0];
  var tempRectangle= app.documents[0].pages.itemByName (app.selection[0].parent.name).rectangles.add();
  tempRectangle.fillColor="Yellow";
  tempRectangle.strokeWeight=0;
  tempRectangle.strokeColor="None";
  var currX=svBon[1];
  var currY=svBon[0];
  var i=j=0;
  cuttingRectangle=new Array();
  if(fitToSmaller == true){
    var diceSiz=Math.min(selH/devH, selW/devW);
    while(currY <= svBon[2]){
      var tempDup=tempRectangle.duplicate ();
      tempDup.visibleBounds=[
        svBon[0]+j*diceSiz, svBon[1]+i*diceSiz,
        svBon[0]+(j+1)*diceSiz, svBon[1]+(i+1)*diceSiz
        ];
      currX=svBon[1]+(i+1)*diceSiz;
      currY=svBon[0]+(j+1)*diceSiz;
      i++;
      if(currX > svBon[3]){
        i=0, j++;
      }
      cuttingRectangle.push(tempDup);
    }
  }
  else{
    while(currY <= svBon[2]){
      var tempDup=tempRectangle.duplicate ();
      tempDup.visibleBounds=[
        svBon[0]+j*selH/devH, svBon[1]+i*selW/devW,
        svBon[0]+(j+1)*selH/devH, svBon[1]+(i+1)*selW/devW
        ];
      currX=svBon[1]+(i+2)*selW/devW;
      currY=svBon[0]+(j+1)*selH/devH;
      i++;
      if(currX > svBon[3]){
        i=0, j++;
      }
      cuttingRectangle.push(tempDup);
    }
  }
  tempRectangle.remove();
  lastRec=cuttingRectangle.pop();
  lastRec.remove();
  return cuttingRectangle;
}
//変形パートFn-2
function transformEachObj(obj){
  for(var t=0; t < obj.length; t++){
    //変形マトリクスオブジェクトをつくる
    var tranMatrix=app.transformationMatrices.add({
      clockwiseShearAngle: mtxSh-2*mtxSh*Math.random(), //シアー
      counterclockwiseRotationAngle: mtxRo-2*mtxRo*Math.random(), //回転
      horizontalScaleFactor: 1+(mtxSc-1)*Math.random(), //水平方向の拡大縮小
      verticalScaleFactor: 1+(mtxSc-1)*Math.random(), //垂直方向の拡大縮小
      horizontalTranslation: mtxTr-2*mtxTr*Math.random(), //水平方向の移動
      verticalTranslation: mtxTr-2*mtxTr*Math.random() //垂直方向の移動
      });
    //変形
    obj[t].transform(
      CoordinateSpaces.PASTEBOARD_COORDINATES, //座標スペースはドキュメント全体
      AnchorPoint.TOP_LEFT_ANCHOR,//アンカーポイントは左上
      tranMatrix,//変形マトリクス
      );
  }
}
```