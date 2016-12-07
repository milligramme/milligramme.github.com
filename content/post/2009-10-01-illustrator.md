+++
title = "配置画像の情報を収集"
date = "2009-10-01T00:00:00+09:00"
tags = ["illustrator", "extendscript"]
+++

Illustrator書類に配置された配置画像のファイル名と変倍率のチェックをするのに、書類点数も配置画像数もそこそこ多かったので、Ai用スクリプトをかいてみた。

多分、人力でやるよりは早くできたと思う。

開いているIllustrator書類に対して、別レイヤに変倍率とファイル名を書出し処理します。

終わったら、eps書類を複製保存、オリジナルは保存せずに閉じます。

クリッピングマスクとグループ化あたりのエラー処理がツメ甘し、グループ化したものや複数画像にマスクしたものなどは取りこぼします、日本語ファイル名はサポートしません（濁点が化ける場合がありあます、OSXだけ？）。

![/images/2010/09/644-ducky_info.jpg](/images/2010/09/644-ducky_info.jpg)

変倍率の出し方はkamisetoさんの  [マトリックスの世界：JavaScriptからイラストレーターに貼付けてある画像の拡大率と角度を得る&nbsp;- なにする？DTP+WEB](http://d.hatena.ne.jp/kamiseto/20090501/1241163399) を参考にしました。

```js
/**
配置画像の情報収集
"collect placed image infomation"
使い方：
Illustrator書類を開いて実行、複数ファイル可。
開いている全てのIllustrator書類上の配置画像の変倍率とファイル名を別レイヤに表示、
その後EPSで複製保存をして、オリジナルは保存せずに閉じます。
グループ化したもの、複数画像をまとめてマスクしたものなどでは取りこぼしがあるかも
しれません。
動作確認：OS10.4.11 Illustrator CS3
milligramme
www.milligramme.cc
*/
var doc = app.documents;
var CR = String.fromCharCode(13);//改行
var SEPA = String.fromCharCode(47);//スラッシュ
for(var i = doc.length-1; i >= 0; i--){
  //情報ラベル枠色
  var framColor = new CMYKColor();
  framColor.cyan = 0;
  framColor.magenta = 0;
  framColor.yellow = 100;
  framColor.black = 0;
  var pValue = 0;
  var infoArray = new Array();
  var positionArray = new Array();
  var lay = doc[i].layers;
  for(var ly = 0; ly < lay.length; ly++){
    lay[ly].locked = false;
    var pgItm = lay[ly].pageItems;
    for(var pgi = 0; pgi < pgItm.length; pgi++){
      pgItm[pgi].locked = false;
      //$.writeln ("doc"+i+"_"+pgItm[pgi].typename+"__"+(pgi+1)+"/"+pgItm.length);
      var mA, mB, mC, mD, posX, posY, nm;
      if(pgItm[pgi].typename == "PlacedItem" || (pgItm[pgi].typename == "GroupItem" && pgItm[pgi].placedItems.length >= 1)){
        if(pgItm[pgi].typename == "PlacedItem"){
          mA = pgItm[pgi].matrix.mValueA;
          mB = pgItm[pgi].matrix.mValueB;
          mC = pgItm[pgi].matrix.mValueC;
          mD = pgItm[pgi].matrix.mValueD;
          posX = pgItm[pgi].position[0];
          posY = pgItm[pgi].position[1];
          nm = decodeURI(pgItm[pgi].file);
          nm = nm.replace(nm,nm.substr (nm.lastIndexOf(SEPA)+1, nm.length-nm.lastIndexOf(SEPA)));
        }
        if(pgItm[pgi].typename == "GroupItem" && pgItm[pgi].placedItems.length >= 1){
          mA = pgItm[pgi].placedItems[0].matrix.mValueA;
          mB = pgItm[pgi].placedItems[0].matrix.mValueB;
          mC = pgItm[pgi].placedItems[0].matrix.mValueC;
          mD = pgItm[pgi].placedItems[0].matrix.mValueD;
          nm = decodeURI(pgItm[pgi].placedItems[0].file);
          nm = nm.replace(nm,nm.substr (nm.lastIndexOf(SEPA)+1, nm.length-nm.lastIndexOf(SEPA)));
          if(pgItm[pgi].clipped == true){//画像がクリップされているとき
            posX = pgItm[pgi].pageItems[0].position[0];
            posY = pgItm[pgi].pageItems[0].position[1];
          }
          else{
            posX = pgItm[pgi].position[0];
            posY = pgItm[pgi].position[1];
          }
        }
        var pgItmHs = Math.round((Math.sqrt((Math.pow(mA,2)) + (Math.pow(mB,2)))*100000))/1000;
        var pgItmVs = Math.round((Math.sqrt((Math.pow(mC,2)) + (Math.pow(mD,2)))*100000))/1000;
        infoArray.push(pgItmHs+"×"+pgItmVs+"%"+CR+nm);//変倍率とファイル名を配列に追加
        positionArray.push([posX,posY]);//情報ラベルのポジションを配列に追加
        pValue++;
      }
    }//for pgi
  }//for ly
  //情報ラベル
  if(doc[i].layers[0].name != "infoLay"){
    var infoLayer = doc[i].layers.add();
    infoLayer.name = "infoLay";
  }
  for(var j = 0; j < pValue; j++){
    var infText = infoLayer.textFrames.add();
    infText.contents = infoArray[j];
    infText.textRange.size = 9;
    infText.textRange.characterAttributes.autoLeading = false;
    infText.textRange.characterAttributes.leading = 10;
    infText.position = positionArray[j];
    var tempRect = infoLayer.pathItems.rectangle(infText.top, infText.left, infText.width, infText.height);
    tempRect.filled = true;
    tempRect.fillColor = framColor;
    infText.zOrder (ZOrderMethod.BRINGTOFRONT);
    //$.writeln ("doc"+i+"_placed"+j+"__"+(j+1)+"/"+pValue)
  }
  //別名保存名をとりあえずepsで
  var removeExtName = doc[i].name.toString().replace(/\..{2,4}$/,"");//拡張子をとる
  var oldName = removeExtName.replace(/ \[更新済み\]?/,"");// [更新済み]があるならとる
  var newPath = doc[i].path.toString()+"/"+oldName+"_.eps";//アンダーバー付きのepsに
  var savePath = new File(newPath);
  var saveOpt = new EPSSaveOptions();
  //ここからeps保存オプション
  saveOpt.compatibility = Compatibility.ILLUSTRATOR13;//AI CS3
  saveOpt.preview = EPSPreview.COLORTIFF;//プレビューTIFFカラー
  saveOpt.overPrint = PDFOverprint.PRESERVEPDFOVERPRINT;//オーバープリント保持
  saveOpt.embedAllFonts = true;
  saveOpt.embedLinkedFiles = false;//配置画像を含まない
  saveOpt.includeDocumentThumbnails = true;
  saveOpt.cmykPostScript = true;
  saveOpt.compatibleGradientPrinting = false;
  //saveOpt.flattenOutput = OutputFlattening.PRESERVEAPPEARANCE//プリンタの初期設定値を使用？
  saveOpt.EPSPostScriptLevelEnum = EPSPostScriptLevelEnum.LEVEL3//ポストスクリプトレベル3
  //別名保存、元データは保存せずに閉じる
  doc[i].saveAs(savePath, saveOpt);
  doc[i].close(SaveOptions.DONOTSAVECHANGES);
}//for doc

```