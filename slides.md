---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: LT for YAPC::Kyoto 2023
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: fade-out
# use UnoCSS
css: unocss
---

# Tokyo Cabinet

## @hiratara 

## FreakOut, Inc.

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
layout: image-right
image: /office_pics04.jpg
---

# FreakOut について

- https://www.fout.co.jp/
- 広告を主軸に色々やってます
- がっつり Perl を使っています
- Tokyo から来ました

<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/guide/syntax#embedded-styles
-->

----

# Tokyo Cabinet について

<div grid="~ cols-2 gap-4">
<div>

- http://fallabs.com/tokyocabinet/
- Mikio さんが 15 年前に開発した DBM
    - 開発はすでに終了している
- Hash, B+ Tree, Fixed-length, Table の実装
- 姉妹プロダクト
    - [QDBM](http://fallabs.com/qdbm/)
    - [Kyoto Cabinet](http://fallabs.com/kyotocabinet/) 
    - 🆕 [Tkrzw](http://fallabs.com/tkrzw/)

</div>
<div>
<img src="/tclogo.png" />
</div>
</div>

---

# FreakOut と Tokyo Cabinet (1)

- オフラインデータベースとして利用
    - 必要な情報を出力
    - web server の足元に配置し、高速アクセス
- めちゃくちゃ速い
    - Perl のハッシュのキーアクセスに毛が生えた程度
- 扱いがすごく楽
    - ファイル単位で生成～コピー、再配置が可能
    - エンディアンに左右されない

---

# FreakOut と Tokyo Cabinet (2)

- 同梱の Perl binding がある
- 良い代替品がない（し、困ってもいない）

<div v-click>
<img src="/no_tkrzw_perl.png" />
</div>

---
layout: image-right
image: /tc_format.png
---

# Tokyo Cabinet の仕様

- [ドキュメント](http://fallabs.com/tokyocabinet/spex-ja.html) によく書かれている
- [mixi engineer blog](https://mixiengineer.hatenablog.com/entry/2007/10665/)
- 実装には色々なノウハウが詰め込まれているが、ファイルフォーマットは非常にシンプル

---

# Rust で再実装

- 自分が長年使っているものをよく知りたい
- Rust を書く練習をしたい
- 「ハッシュデータベース」の「読み込み」に的を絞って実装
- `tchmgr` の `get` `list` 辺りの機能が目標

---

# Rust の良い点 (1) 

## (GCの言語と比べて) 動作が高速

```
$ time tchmgr list -nl casket.tch | wc -l
1680800

real    0m1.306s
user    0m0.618s
sys     0m0.804s
$ time rstc list casket.tch | wc -l
1680800

real    0m2.910s
user    0m1.069s
sys     0m3.849s
```

---

# Rust の良い点 (2) 

## (Cと比べて) ジェネリクスが強力

* `R` → ファイル以外の読み込みに対応
* `B` → 32bit/64bit のフォーマットに対応
* Zero Cost Abstractions: コンパイル時に展開

```rust
pub struct TCHDBImpl<B, R> {
    pub reader: R,
    pub header: Header,
    pub bucket_offset: u64, // always be 256
    pub free_block_pool_offset: u64,
    bucket_type: PhantomData<fn() -> B>,
}
```

---

# Rust の良い点 (3) 

## crates.io の資源が使える

<div grid="~ cols-2 gap-4">
<div>

* [binrw](https://crates.io/crates/binrw)
    * バイナリデータの高速な読み書き
    * わかりやすい宣言的な記述

* [structopt](https://crates.io/crates/structopt)
    * コマンドライン引数の宣言的なパース
</div>
<div>
```rust
#[derive(BinRead, Debug)]
#[br(little)]
pub struct Header {
    #[br(count = 32, assert(magic_number.starts_with(b"ToKyO CaBiNeT")))]
    pub magic_number: Vec<u8>,
    #[br(assert(database_type == 0))]
    pub database_type: u8,
    pub additional_flags: u8,
    pub alignment_power: u8,
    pub free_block_pool_power: u8,
    #[br(pad_after = 3)]
    pub options: u8,
    pub bucket_number: u64,
    pub record_number: u64,
    pub file_size: u64,
    #[br(pad_after = 56)]
    pub first_record: u64,
    #[br(count = 128)]
    pub opaque_region: Vec<u8>,
}
```
</div>
</div>

---

# Tokyo Cabinet の細かいバグ (1)

ドキュメントがおかしい。

> データベースタイプ
> ハッシュ表（0x01）かB+木（0x02）か固定長（0x03）かテーブル（0x04）

```c
enum {                                   /* enumeration for database type */
  TCDBTHASH,                             /* hash table */
  TCDBTBTREE,                            /* B+ tree */
  TCDBTFIXED,                            /* fixed-length */
  TCDBTTABLE                             /* table */
};
```

---

# Tokyo Cabinet の細かいバグ (2)

コメントが逆。

```c
/* Load the free block pool from the file.
   The return value is true if successful, else, it is false. */
static bool tchdbsavefbp(TCHDB *hdb){
```

```c
/* Save the free block pool into the file.
   The return value is true if successful, else, it is false. */
static bool tchdbloadfbp(TCHDB *hdb){
```

---

# Tokyo Cabinet の細かいバグ (3)

* aptで入るdebian/ubuntuの Tokyo Cabinet の endian が逆
    * 他の環境で生成したDBとの互換がない
    * `--enable-swab` オプションは big endian 環境では動かない
* [Bug#667979: libtokyocabinet9: TokyoCabinet got endianness in DB wrong on both big- and little-endian architectures](https://debian-bugs-dist.debian.narkive.com/I4IA9otI/bug-667979-libtokyocabinet9-tokyocabinet-got-endianness-in-db-wrong-on-both-big-and-little-endian)
    * 11 years ago

---

# まとめ

* 開発が終わった OSS が活躍していることもある（自己責任）
* Rust はいいぞ
* 再実装は勉強になる
