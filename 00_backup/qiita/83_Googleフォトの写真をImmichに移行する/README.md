https://qiita.com/kijuky/items/4aae80873921003aeec3

---

# 背景

https://qiita.com/kijuky/items/78e2afd53788ebf1a17d

Googleフォトの容量が少なくなってきたので、NASにImmichを建てて運用しています。GoogleフォトのデータはGoogleデータエクスポートで取得してImmichにアップロードします。通常、写真のメタデータはExifに存在しますが、Googleフォト上で管理されていたメタデータについては supplemental-metadata.json から取得します。

# Googleデータエクスポート

Googleフォトのデータを取得する場合、GoogleフォトAPIを使うことができません。移行のために写真を一括取得したい場合は、Googleデータエクスポートを利用する必要があります。

## Googleフォトのデータ

Googleデータエクスポートで対象になるGoogleフォトのデータは次のとおりです。

- タイムライン上にある写真（Photos from `yyyy` というフォルダ）
    - 自分でアップロードした写真
    - 回転後の写真（ `-編集済み` というサフィックスがつく）
    - 保存したコラージュ（ `-COLLAGE` というサフィックスがつく）
    - 共有アルバムから保存した写真（メタデータから共有されたことはわかるが、誰から共有されたかはわからない）
    - モーションフォトに対応する動画（拡張子が `.mp4` になる）
- アルバム（アルバム名のフォルダ。重複した場合 `(\d+)` のサフィックスがつく）
    - 自分が追加した写真（上の タイムライン上にある写真 と重複する）
- アーカイブした写真（アーカイブ というフォルダ）
- コンビニ印刷した写真（Print Order 000000000000000000000 というフォルダ)
    - print-order.pdf というプレビュー
- アップロードに失敗した動画（Failed Videos というフォルダ）

注意として、**共有アルバムで共有された写真は保存されません**（タイムラインに保存したら「タイムライン上にある写真」に保存されます）。**自分が一枚も写真を追加していないアルバムはフォルダも作成されません**。ただし、これを嫌って共有された写真をタイムラインに保存すると、その写真は容量加算対象になります。そもそも容量不足で移行しているため、保存していない共有された写真は個別に保存したほうがよいかもしれません。

なお、アルバムに関するメタデータは `アルバム/メタデータ.json` というファイルに保存されています。

## supplemental-metadata.json とは

Googleデータエクスポートによって取得したGoogleフォトのデータには写真データのメタデータは `写真ファイル.supplemental-metadata.json` というファイルに保存されています。

```json:例
{
  "title": "2018-07-30 12.26.03.jpg",
  "description": "",
  "imageViews": "41",
  "creationTime": {
    "timestamp": "1537795814",
    "formatted": "2018/09/24 13:30:14 UTC"
  },
  "photoTakenTime": {
    "timestamp": "1532921163",
    "formatted": "2018/07/30 3:26:03 UTC"
  },
  "geoData": {
    "latitude": 35.667099,
    "longitude": 139.7314453,
    "altitude": 104.0,
    "latitudeSpan": 0.0,
    "longitudeSpan": 0.0
  },
  "geoDataExif": {
    "latitude": 35.667099,
    "longitude": 139.7314453,
    "altitude": 104.0,
    "latitudeSpan": 0.0,
    "longitudeSpan": 0.0
  },
  "people": [{
    "name": "XXX"
  }, {
    "name": "YYY"
  }],
  "favorited": true,
  "url": "https://photos.google.com/photo/XXXXXXXXXXXX-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "googlePhotosOrigin": {
    "webUpload": {
      "computerUpload": {
      }
    }
  }
}
```

各種フィールドの内容はこんな感じ。

| フィールド | 説明 | 必須 |
| :--- | :--- | :---: |
| title | ファイル名 | ⚫︎ |
| description | 説明 | ⚫︎ |
| imageViews | 写真が表示された回数 | ⚫︎ |
| creationTime | 写真がGoogleフォトにアップロードされた時刻 | ⚫︎ |
| photoTakenTime | 撮影時刻 | ⚫︎ |
| geoData | 位置情報。不明な場合は 0.0 が設定される。 | ⚫︎ |
| geoDataExif | 位置情報 | |
| people | 写真に写っている人。列挙されるのは、名前を設定した人のみ | |
| favorited | お気に入りかどうか。写真がお気に入りの場合のみフィールドが存在し、値は常に true | |
| url | URL | ⚫︎ |
| googlePhotosOrigin | 写真がどのようにGoogleフォトにアップロードされたかがわかるかも | ⚫︎ |

## ファイルの命名

**ここが最悪でした**。

- 写真は基本的にはアップロードされたときのファイル名
    - ファイル名が重複した場合は `photo(1).jpg` のように `(\d+)` サフィックスがつく
- メタデータは `photo.jpg.supplemental-metadata.json` というファイル名
    - ファイル名が重複した場合は `photo.jpg.supplemental-metadata(1).json` というファイル名
        - `photo(1).jpg.supplemental-metadata.json` ではないのに注意
    - ファイル名が長くなった場合、 `.supplemental-metadata` 部分が後ろからちょん切れる。例：`long-long-long-long-photo.jpg.supplemental-met.json`
    - ファイル名が長くなった場合、 `.supplemental-metadata` 部分がなくなる。例：`long-long-long-long-long-long-long-long-photo.jpg.json`
    - ファイル名が長くなった場合、末尾の一文字が取れることがある
        - ？？意味がわからない
- モーションフォトに対応する動画は拡張子が `.mp4` になる
    - `motion-photo.jpg` なら `motion-photo.mp4`
    - 拡張子の前に拡張子っぽい文字列があると、`.mp4` がつかない
        - `photo.mp.jpg` なら `photo.mp`
            - バグじゃね？
- モーションフォト、編集済みの写真に対応するメタデータファイルはない
    - それらの元データに対応するメタデータを紐づける必要がある

この命名のせいで、ある写真に対応するメタデータjsonを検索することが極端に難しくなっています。これに関するエレガントな検索方法を知っている方がいればコメント欄で教えてください。私はとりあえず下記の方針で概ね満足した結果を得ています。

- `photo.*\.json` に引っかかるファイルを検索する
    - この時点で対象ファイルが1つしかなかったら、それが対応するメタデータファイル
    - ファイルが複数ヒットした場合は
        - サフィックスに `(\d+)` を加えて絞り込む
        - モーションフォトや `編集済み` の場合は特定文言を切り取って元ファイル名を極力復元して絞り込む
- 拡張子が変な場合はそれをモーションフォトに対応するmp4ファイルとみなして、拡張子 `.mp4` を付与する

# Immich移行

## Googleフォトデータの整理

Googleデータエクスポートで得られたファイルを次のように配置します。

- Googleフォト
    - アルバム1
        - 写真1.jpg
        - ...
    - アルバム2
    - ...
    - upload.sc

ここで `upload.sc` は下記で説明する、immichに写真をアップロードするScalaCLIです。

## アップロードスクリプト

このアップロードスクリプトは次のことを行います。

- `Googleフォト` というタグを作成
- `Googleフォト/共有された写真` というタグを作成
- `Googleフォト/自動生成` というタグを作成
- フォルダに対応するアルバムを作成
- フォルダ内の対応アセット（写真・動画）をアップロード
    - ファイル名、撮影時刻、日付、お気に入り情報をメタデータから反映
    - `Googleフォト` というタグを設定
    - もしそれが共有された写真ならさらに `Googleフォト/共有された写真` というタグを設定
    - もしそれが自動生成された写真ならさらに `Googleフォト/自動生成` というタグを設定

```scala:upload.sc
//> using dep com.softwaremill.sttp.client4::core:4.0.12
//> using dep com.lihaoyi::upickle:4.3.2

import java.nio.file.*
import java.security.MessageDigest
import java.time.* 
import java.time.format.*
import java.util.UUID

import scala.util.chaining.*

// sstpの設定
import sttp.client4.*
import sttp.client4.httpurlconnection.HttpURLConnectionBackend
import sttp.model.*
val backend = HttpURLConnectionBackend()

// upickleの設定
import upickle.default.{ReadWriter => RW, macroRW}

val IMMICH_API_URL = "http://サーバーIP:ポート番号/api" // docker でやる場合は "http://immich_server.immich_default:2283/api"
val IMMICH_TOKEN = "immichのユーザー設定で作ったトークン" // album.create,album.read,asset.upload,asset.update,tag.create,tag.read,tag.asset,albumAsset.create
val DEVICE_ASSET_ID = UUID.randomUUID.toString
val DEVICE_ID = "scala-script"
val TAKEOUT_GOOGLE_PHOTO = "."

@main def program =
  val googlePhotoTag = getOrCreateTag("Googleフォト", "#00FF00")
  val sharedPhotoTag = getOrCreateTag("Googleフォト/共有された写真", "#0000FF")
  val autoGeneratedTag = getOrCreateTag("Googleフォト/自動生成", "#00FF00")

  val wd = Paths.get(TAKEOUT_GOOGLE_PHOTO).toAbsolutePath.normalize

  for
    albumPath <- Files.list(wd).toArray(Array.ofDim[Path]).toSeq
    if Files.isDirectory(albumPath)
    albumName = albumPath.getFileName.toString
    if !albumName.startsWith(".") // 隠しファイルは対象外
  do
    val album = getOrCreateAlbum(albumName)
    val assetPaths = Files.list(albumPath).toArray(Array.ofDim[Path]).toSeq
    val metadataJsonNames = assetPaths.map(_.getFileName.toString).filter(_.endsWith(".json"))
    for
      assetPath <- assetPaths
      //ファイル名に含まれる濁音が、濁点が一文字になっているとmacからファイルが見えなくなる。mac上からリネームして対応するのは難しい？
      if Files.isRegularFile(assetPath)
      assetFileName = assetPath.getFileName.toString
      if List(".jpg", ".jpeg", ".png", ".gif", ".dng", ".heic", ".mp4", ".mp", ".mp~2", ".m4v", ".avi", ".mov", ".wmv", ".insp", ".insv", ".3gp").exists(assetFileName.toLowerCase.endsWith)
      assetBaseName = assetFileName.substring(0, assetFileName.lastIndexOf("."))
      normalizedAssetName6 = assetBaseName.replaceFirst("""(-編集済み)?(\(\d+\))?$""", "?.*").replaceAll("""\(""", """\\(""").replaceAll("""\)""", """\\)""")
      normalizedAssetName1 = assetFileName.replaceFirst("""-編集済み\.""", ".").replaceAll("""\(""", """\\(""").replaceAll("""\)""", """\\)""")
      normalizedAssetName2 = assetBaseName.replaceFirst("""(-編集済み)?(\(\d+\))$""", ".*$2").replaceAll("""\(""", """\\(""").replaceAll("""\)""", """\\)""")
      normalizedAssetName3 = assetBaseName.replaceAll("""\(""", """\\(""").replaceAll("""\)""", """\\)""")
      normalizedAssetName4 = assetBaseName.init.replaceAll("""\(""", """\\(""").replaceAll("""\)""", """\\)""")
      normalizedAssetName5 = assetBaseName.replaceFirst("""\(\d+\)$""", "?").replaceAll("""\(""", """\\(""").replaceAll("""\)""", """\\)""")
      metadataJsonRegex = s"$normalizedAssetName6\\.json"
      metadataJsonRegex1 = s"$normalizedAssetName1\\..*[-a-z]\\.json" // .supplimental-metadata.json にマッチさせる。回転などでできた「編集済み」ファイルは、元ファイルのメタデータにマッチさせる。
      metadataJsonRegex2 = s"$normalizedAssetName2\\.json"  // photo(1).jpg に対応するメタデータは photo.supplemental-metadata(1).json に命名されるので、それにマッチさせる。
      metadataJsonRegex3 = s"$normalizedAssetName3\\..*\\.json" // モーションフォトに対応する.mp4ファイルは、元ファイルのメタデータにマッチさせる。
      metadataJsonRegex4 = s"$normalizedAssetName4\\.json" // 元ファイル名が長い場合、元ファイルの末尾1文字が欠けることがある。
      metadataJsonRegex5 = s"$normalizedAssetName5\\.json"
      metadataJsonName =
        metadataJsonNames.filter(_.matches(metadataJsonRegex)) match
          case Nil => None
          case Seq(single) => Some(single)
          case candidates =>
            candidates.sorted.find(_.matches(metadataJsonRegex1))
              .orElse(candidates.find(_.matches(metadataJsonRegex2)))
              .orElse(candidates.find(_.matches(metadataJsonRegex3)))
              .orElse(candidates.find(_.matches(metadataJsonRegex4)))
              .orElse(candidates.find(_.matches(metadataJsonRegex5)))
              .orElse(candidates.find(an => an == s"$assetBaseName.json" || an == s"${assetBaseName.init}.json")) // 元ファイル名が長い場合、.supplemental-metadata の部分文字列が存在しない場合がある。
    do
      uploadAsset(assetPath, metadataJsonName.map(albumPath.resolve), album, googlePhotoTag, sharedPhotoTag, autoGeneratedTag)

final case class Album(albumName: String, id: String)
object Album:
  implicit val rw: RW[Album] = macroRW

final case class CreateAlbum(albumName: String)
object CreateAlbum:
  implicit val rw: RW[CreateAlbum] = macroRW

def getOrCreateAlbum(albumName: String): Album =
  val response =
    quickRequest
      .header("x-api-key", IMMICH_TOKEN)
      .get(uri"$IMMICH_API_URL/albums")
      .send(backend)
  val albums = upickle.read[Seq[Album]](response.body)
  albums.find(_.albumName == albumName) match
    case Some(album) => album
    case None =>
      val createResponse =
        quickRequest
          .header(Header.contentType(MediaType.ApplicationJson))
          .header("x-api-key", IMMICH_TOKEN)
          .post(uri"$IMMICH_API_URL/albums")
          .body(upickle.write(CreateAlbum(albumName)))
          .send(backend)
      upickle.read[Album](createResponse.body)

final case class SupplementalMetadata(title: String, description: String, creationTime: SupplementalMetadataTime, photoTakenTime: SupplementalMetadataTime, favorited: Boolean = false)
object SupplementalMetadata:
  implicit val rw: RW[SupplementalMetadata] = macroRW

final case class SupplementalMetadataTime(timestamp: String, formatted: String):
  def timestampInstant = Instant.ofEpochSecond(timestamp.toLong)
  def formattedZonedDateTime = ZonedDateTime.parse(formatted, DateTimeFormatter.ofPattern("yyyy/MM/dd H:mm:ss z"))
  def formattedAsJST = formattedZonedDateTime.withZoneSameInstant(ZoneId.of("Asia/Tokyo")).toOffsetDateTime.toString
object SupplementalMetadataTime:
  implicit val rw: RW[SupplementalMetadataTime] = macroRW

final case class UploadFailed(statusCode: Int)
object UploadFailed:
  implicit val rw: RW[UploadFailed] = macroRW

final case class UploadedAsset(id: String, status: String)
object UploadedAsset:
  implicit val rw: RW[UploadedAsset] = macroRW

final case class UpdateAsset(dateTimeOriginal: String, description: String)
object UpdateAsset:
  implicit val rw: RW[UpdateAsset] = macroRW

final case class AddAssetsToAlbums(albumIds: Seq[String], assetIds: Seq[String])
object AddAssetsToAlbums:
  implicit val rw: RW[AddAssetsToAlbums] = macroRW

final case class Tag(id: String, value: String)
object Tag:
  implicit val rw: RW[Tag] = macroRW

final case class CreateTag(color: String, name: String)
object CreateTag:
  implicit val rw: RW[CreateTag] = macroRW

final case class TagAssets(ids: Seq[String])
object TagAssets:
  implicit val rw: RW[TagAssets] = macroRW

def uploadAsset(assetPath: Path, metadataJsonPath: Option[Path], album: Album, googlePhotoTag: Tag, sharedPhotoTag: Tag, autoGeneratedTag: Tag): Unit =
  val metadataJsonText = metadataJsonPath.map(Files.readString(_))
  val metadata = metadataJsonText.map(upickle.read[SupplementalMetadata](_))
  val assetBlob = Files.readAllBytes(assetPath)
  val checksum = MessageDigest.getInstance("SHA1").digest(assetBlob).map("%02x".format(_)).mkString
  val deviceAssetId = multipart("deviceAssetId", DEVICE_ASSET_ID)
  val deviceId = multipart("deviceId", DEVICE_ID)
  val fileCreatedAt = multipart("fileCreatedAt", metadata.map(_.photoTakenTime.formattedAsJST).getOrElse(""))
  val fileModifiedAt = multipart("fileModifiedAt", metadata.map(_.creationTime.formattedAsJST).getOrElse(""))
  val filename = multipart("filename", metadata.map(_.title).getOrElse(assetPath.getFileName.toString))
  val isFavorite = multipart("isFavorite", metadata.map(_.favorited).getOrElse(false).toString)

  // ファイル名とメタデータのタイトルの拡張子が異なり、ファイル名がメタデータのタイトルの部分文字列であった場合、そのファイルをタイトルにリネームしたファイルを対象にする（処理後に消す）
  val assetFileName = assetPath.getFileName.toString
  val assetFileExt = assetFileName.substring(assetFileName.lastIndexOf("."))
  val renamedAssetPathOpt =
    metadata match
      case Some(meta) if !meta.title.endsWith(assetFileExt) && meta.title.startsWith(assetFileName.replaceFirst("""\(\d+\)\.""", ".")) =>
        val renamedAssetPath = Paths.get(s"./$assetFileName.mp4")
        Files.copy(assetPath, renamedAssetPath)
        Some(renamedAssetPath)
      case _ =>
        None
  val assetData = multipartFile("assetData", renamedAssetPathOpt.getOrElse(assetPath))

  def requestUploadAsset(): Option[UploadedAsset] =
    val uploadRequest = 
      basicRequest
        .header(Header.contentType(MediaType.MultipartFormData))
        .header("x-api-key", IMMICH_TOKEN)
        .header("x-immich-checksum", checksum)
        .post(uri"$IMMICH_API_URL/assets")
        .multipartBody(assetData, deviceAssetId, deviceId, fileCreatedAt, fileModifiedAt, filename, isFavorite)
    val response = uploadRequest.send(backend)
    response.body match
      case Right(body) =>
        Some(upickle.read[UploadedAsset](body))
      case Left(errorMessage) =>
        upickle.read[UploadFailed](errorMessage) match
          case UploadFailed(statusCode) if statusCode / 100 == 5 =>
            None
          case UploadFailed(statusCode) =>
            throw Exception(s"${assetFileName}\n$errorMessage")

  // 5xxの場合、5秒待ってリトライする（3回まで）。
  val uploadedAsset =
    (1 to 3).iterator
      .map(_ => requestUploadAsset().orElse { Thread.sleep(5000); None })
      .collectFirst { case Some(uploadedAsset) => uploadedAsset }
      .getOrElse(throw Exception(s"${assetFileName}\nUpload failed after 3 attempts."))

  metadata.foreach(updateAsset(uploadedAsset.id, _))

  addAssetToAlbum(uploadedAsset.id, album.id)

  tagAsset(uploadedAsset.id, googlePhotoTag)

  if metadataJsonText.exists(_.contains("\"fromSharedAlbum\":")) then
    tagAsset(uploadedAsset.id, sharedPhotoTag)

  if metadataJsonText.exists(_.contains("\"composition\":")) then
    tagAsset(uploadedAsset.id, autoGeneratedTag)

  // リネームしたファイルを消す
  renamedAssetPathOpt.foreach: renamedAssetPath =>
    Files.delete(renamedAssetPath)

def updateAsset(assetId: String, metadata: SupplementalMetadata): Unit =
  val dateTimeOriginal = metadata.photoTakenTime.formattedAsJST
  val description = metadata.description

  val response =
    quickRequest
      .header(Header.contentType(MediaType.ApplicationJson))
      .header("x-api-key", IMMICH_TOKEN)
      .put(uri"$IMMICH_API_URL/assets/$assetId")
      .body(upickle.write(UpdateAsset(dateTimeOriginal, description)))
      .send(backend)

def addAssetToAlbum(assetId: String, albumId: String): Unit =
  val response =
    quickRequest
      .header(Header.contentType(MediaType.ApplicationJson))
      .header("x-api-key", IMMICH_TOKEN)
      .put(uri"$IMMICH_API_URL/albums/assets")
      .body(upickle.write(AddAssetsToAlbums(Seq(albumId), Seq(assetId))))
      .send(backend)

def getOrCreateTag(tagName: String, color: String): Tag =
  quickRequest
    .header("x-api-key", IMMICH_TOKEN)
    .get(uri"$IMMICH_API_URL/tags")
    .send(backend)
    .body
    .pipe(upickle.read[Seq[Tag]](_))
    .find(_.value == tagName)
    .getOrElse:
      val createTagResponse =
        quickRequest
          .header(Header.contentType(MediaType.ApplicationJson))
          .header("x-api-key", IMMICH_TOKEN)
          .post(uri"$IMMICH_API_URL/tags")
          .body(upickle.write(CreateTag(color, tagName)))
          .send(backend)
      upickle.read[Tag](createTagResponse.body)

// unused
def tagAsset(assetId: String, tagName: String, color: String): Unit =
  tagAsset(assetId, getOrCreateTag(tagName, color))

def tagAsset(assetId: String, tag: Tag): Unit =
  val response =
    quickRequest
      .header(Header.contentType(MediaType.ApplicationJson))
      .header("x-api-key", IMMICH_TOKEN)
      .put(uri"$IMMICH_API_URL/tags/${tag.id}/assets")
      .body(upickle.write(TagAssets(Seq(assetId))))
      .send(backend)
```

このスクリプトを次のように呼び出します。

```sh
scala upload.sc
```

全てのアップロードした写真には `Googleフォト` というタグがついているので、アップロードした写真をやっぱり消したい場合はタグから一括削除できます。

# まとめ

GoogleフォトのデータはGoogleデータエクスポートで一括ダウンロードでき、メタデータを紐づけることでimmich上にメタデータも含めて移行することができます。メタデータの紐付けはファイル命名規則が複雑なため、スクリプトで良い感じに調整する必要が出てきます。immichへのアップロードを含めたimmichの操作はapiがあるため、sttpのようなhttpクライアントで自動化ができます。

紐付けに苦戦するとは思わなかったので、最初はChatGPTに、得意なPythonで書いていいよと指示したのですが、命名規則が複雑なのとそのロジックをPythonで書ける自信がなかった＆ScalaCLIってまだ使ったことなかったのでScalaCLIを使ってみたくてスクリプトはScalaで書いてみました。やはり慣れている言語はデバッグしやすくていいですね。jsonの型付けのためのcase classの定義がノイジーですが、スクリプトならujsonとして扱っても良かったのかなぁとか思いました。

以上です。Googleフォトの容量が気になっていたり、immichに興味を持っていたり、scala-cliどんなもんか興味がある人は参考になれば幸いです。

# 参考文献

https://qiita.com/kapyba_ch/items/364f68807ad3a0d6f763
