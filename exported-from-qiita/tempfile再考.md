---
title: tempfile再考
tags: UNIX Ruby
author: yuroyoro
slide: false
---
何気なく使ってる一時ファイル(tempfile)ですが、このような事があったので少し実装を調べてみました。

<blockquote class="twitter-tweet" lang="en"><p>テストが高速化されたことで、tmp fileをunixtimeで自前生成していたtest caseが軒並みfailするようになったので、テスト用に生成するファイルはtmpfile(3)使いましょうという知見</p>&mdash; ⁰⁰⁰⁰null (@yuroyoro) <a href="https://twitter.com/yuroyoro/status/509895197371531265">September 11,  2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

# 一時ファイル(tempfile)に期待される挙動

1. ユニークで推測しにくいファイル名で生成される
1. プロセス終了時に削除される
1. 他のプロセスからは見えない

実際に一時ファイルを作成する際は、これらの要件のうちのいくつかを期待していると思います。
アプリケーションからこのような特性を持つ一時ファイルを生成するためには、通常はなんらかのライブラリを利用するはずです。

というか、普通にunixtimeを元にしたファイル名で`open`して`write`して、使い終わったら`close/unlink`するような素朴な実装は問題があります。

が、利用するライブラリの実装や使い方によっては、上記の要件全てを満たさないことがあるので注意が必要です。


セキュアにtmpfileを取り扱う方法については、以下の文書を見るとよいでしょう。

[IPA ISEC　セキュア・プログラミング講座：C/C++言語編　第7章 データ漏えい対策：テンポラリファイル（Unix の一時ファイル）](https://www.ipa.go.jp/security/awareness/vendor/programmingv2/contents/c603.html)

上記の文書では、`mkstemp(3)` または `tmpfile(3)`を使用すべき、と書いてあります。が、この関数を利用しない実装のライブラリも存在します。
具体的にはRuby標準ライブラリの`Tempfile`モジュールですが、この実装とISO C標準ライブラリとの違いについては、後ほど解説します。

# tmpfile(3)

[tmpfile(3): create temporary file - Linux man page](http://linux.die.net/man/3/tmpfile)

`tmpfile(3)`は、`1`と`2`と`3`のすべての要件を満たします。が、glibcなどのメジャーな実装では、ファイルを`open`した直後に`unlink`してしまうので、具体的なファイル名を利用者が知る術はありません。
また、他のプロセスが`read/write`することも不可能です。

`open`後に`unlink`していますが、`FILE`構造体を通して操作することは可能です。明示的に`close`するかプロセスが終了することで削除されます。

glibcの実装を見ると、確かに一意なファイル名を生成して`open`した後に`unlink`しています。

```c:stdio-common/tmpfile.c
FILE *
tmpfile (void)
{
  char buf[FILENAME_MAX];
  int fd;
  FILE *f;

  if (__path_search (buf, FILENAME_MAX, NULL, "tmpf", 0))
    return NULL;
  int flags = 0;
#ifdef FLAGS
  flags = FLAGS;
#endif

  /*
     __gen_tempnameは一意なファイル名を生成してopenする 。
     KINDが__GT_FILEの場合は、open(O_CREAT|O_EXCL) を用いるのでファイル名の衝突はない。
   */
  fd = __gen_tempname (buf, 0, flags, __GT_FILE);
  if (fd < 0)
    return NULL;

  /* Note that this relies on the Unix semantics that
     a file is not really removed until it is closed.

     Unix環境では、unlinkしたとしても実際に削除されるのは
     closeされたタイミングであることに基づいている
     unlinkされているため、プロセス終了時にos側がfdを
     回収してcloseする。よって終了時に実体は削除される。
   */
  (void) __unlink (buf);

  if ((f = __fdopen (fd, "w+b")) == NULL)
    __close (fd);

  return f;
}

```

[sourceware.org Git - glibc.git/blob - stdio-common/tmpfile.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdio-common/tmpfile.c;h=022c2f650bc97ac467b5a5019552e6a6ae882c94;hb=HEAD)

なお、ここで使われている`__gen_tempname`関数は、後述する`mkstemp(3)`の実体でもあります。
つまり、glibcの実装では、`mkstemp(3)`を呼び出した後に`unlink`しているというわけです。

```c:sysdeps/posix/tempname.c
int
__gen_tempname (char *tmpl, int suffixlen, int flags, int kind)
{
  /* 中略 */

  value += random_time_bits ^ __getpid ();

  /* 一意なファイルがopenできるまでリトライしづづける */
  for (count = 0; count < attempts; value += 7777, ++count)
    {
      uint64_t v = value;

      /* Fill in the random bits.  */
      XXXXXX[0] = letters[v % 62];
      v /= 62;
      /* 以下XXXXXX[5]まで同様なので省略 */


      switch (kind)
        {
        case __GT_FILE:
          fd = __open (tmpl,
                       (flags & ~O_ACCMODE)
                       | O_RDWR | O_CREAT | O_EXCL, S_IRUSR | S_IWUSR);
          break;

        case __GT_DIR:
          /* 省略 */

        case __GT_NOCREATE:
         /* 省略 */

        default:
          assert (! "invalid KIND in __gen_tempname");
          abort ();
        }

      /* oepnできたらreturnする */
      if (fd >= 0)
        {
          __set_errno (save_errno);
          return fd;
        }
      /* ファイルが存在する(EEXSIT)以外のerrorはreturn */
      else if (errno != EEXIST)
        return -1;
    }

  /* We got out of the loop because we ran out of combinations to try.  */
  /* リトライ回数を超えた場合はエラー */
  __set_errno (EEXIST);
  return -1;
}
```

以下は、`tmpfile(3)`で取得した`FILE`構造体を通して操作するサンプルです。


```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

int
main(void)
{

    FILE *fp;
    int fd;
    struct stat stat_buf;
    char str[] = "foo";
    int c;

    printf("pid => %ld\n", (long)getpid());

    if ((fp = tmpfile()) == NULL){
        perror("tmpfile");
        return EXIT_FAILURE;
    }

    fwrite(str, 1, sizeof(str), fp );
    fflush(fp);
    rewind(fp);

    fd = fileno(fp);
    if (fstat(fd, &stat_buf) == 0) {
        printf("Created tempfile(inode) => %llu\n", stat_buf.st_ino);
    } else {
        perror("stat");
        return EXIT_FAILURE;
    }

    printf("Tempfile body => ");
    while ((c = getc(fp)) != EOF) {
        putc(c, stdout);
    }
    printf("\n");
    fflush(stdout);

    for(;;){}

    fclose(fp);
    return EXIT_SUCCESS;
}

```

なお、tmpfile用に一意なパス名を生成する`tmpnam(3)`という関数もありますが、こちらについては脆弱性があるため使用を避けるように、冒頭の資料でも示されています。

[tmpnam(3): create name for temporary file - Linux man page](http://linux.die.net/man/3/tmpnam)

# mkstemp(3)

[mkstemp(3): create unique temporary file - Linux man page](http://linux.die.net/man/3/mkstemp)

`mkstemp(3)`は、一意なファイル名でファイルをopenします。`tmpfile(3)`と異なり、`open`後に`unlink`しないため、生成されたファイルは外部のプロセスからアクセス可能ですし、プロセス終了後も削除されません。
冒頭に示した要件のうち、`1. ユニークで推測しにくいファイル名で生成される`のみを満たすということになります

以下のように、明示的にunlinkを呼び出すことで、`tmpfile(3)`と同じくプロセス終了後に削除されるようになります。

```c

    /* 一時ファイルopen */
    if ((fd = mkstemp(template)) < 0){
        perror("mkstemp");
        return EXIT_FAILURE;
    }

    dosomething();

    /* unlinkでプロセス終了後に削除されるようにする */
    unlink(template);

```

`tmpfile(3)`と異なる点は、`unlink`するタイミングを任意に選択することができる点でしょう。`mkstemp(3)`でファイルを生成しておいて、使い終わった任意のタイミングで`unlink`させることで、他のプロセスでもファイルにアクセスさせつつ、プロセス終了時に削除されるようになります。ただし、後述のRubyの`Tempfile`モジュールと同様の問題も発生しうるので注意が必要です。

なお、引数には、ファイル名のひな形となる文字列をわたすことになります。例えば、`foo_bar_XXXXXX`という文字列を渡した場合、`XXXXXX`が一意になるように置換されます。
この引数は`mkstemp(3)`側で書き換えられるので注意が必要です(詳細は「詳解Unixプログラミング」参照)。



glibcの実装では、`tmpfile(3)`で示した`__gen_tempname`関数を呼び出しているだけです。

```c:misc/mkstemp.c
int
mkstemps (template, suffixlen)
     char *template;
     int suffixlen;
{
  if (suffixlen < 0)
    {
      __set_errno (EINVAL);
      return -1;
    }

  return __gen_tempname (template, suffixlen, 0, __GT_FILE);
}
```

[sourceware.org Git - glibc.git/blob - misc/mkstemps.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=misc/mkstemps.c;h=d58fce36da436c658c2475f10c6791b56d80c5ad;hb=HEAD)

# Ruby : Tempfile

[class Tempfile](http://docs.ruby-lang.org/ja/2.1.0/class/Tempfile.html)


今度は、Rubyの`Tempfile`モジュールの実装を見てみましょう。

`tmpfile(3)`と異なり、Rubyの`Tempfile.open`関数は`unlink`を呼び出しません。
ですが、`Tempfile.open`で生成したファイルはプロセス終了時に削除されます。また、他のプロセスがアクセスすることが可能です。

冒頭に示した要件のうち、`1. ユニークで推測しにくいファイル名で生成される`と`2. プロセス終了時に削除される`を満たすということになります。

さて、プロセス終了時のファイルの回収についてです。
実装をのぞいてみると、`Tempfile`クラスのコンストラクタで、`ObjectScape.define_finalizer`を用いて、GCまたはプロセス終了時に`unlink`されるようになっているようです。

```ruby:lib/tempfile.rb
  def initialize(basename, *rest)
    if block_given?
      warn "Tempfile.new doesn't call the given block."
    end
    @data = []
    @clean_proc = Remover.new(@data)
    ObjectSpace.define_finalizer(self, @clean_proc)

    # 略
  end

  class Remover

    def call(*args)
      return if @pid != $$ #子プロセスではなにもしない

      path, tmpfile = *@data

      STDERR.print "removing ", path, "..." if $DEBUG

      tmpfile.close if tmpfile

      if path
        begin
          File.unlink(path) # ここでunlink
        rescue Errno::ENOENT
        end
      end

      STDERR.print "done\n" if $DEBUG
    end
  end

```

[ruby/tempfile.rb at trunk · ruby/ruby](https://github.com/ruby/ruby/blob/trunk/lib/tempfile.rb#L133)

`ObjectScape.define_finalizer`でunlinkされるということは、`tmpfile(3)`と異なり、OSがfdを回収する機構に依存していない、ということです。
`define_finalizer`で登録された処理が実行されずにプロセスが終了した場合は、ファイルはそのまま残ってしまいます。

以下のような簡単なプログラムを用意します。`Tempfile.open`して得られたファイル名を出力したあと無限loopに突入するだけの簡単なものです。

```ruby
#!/usr/bin/env ruby

require 'tempfile'

puts "RUBY_VERSION => #{RUBY_VERSION}"
puts "uname => #{`uname -a`}"
puts "pid => #{Process.pid}"

# Tempfile作成
f = Tempfile.open("hoge")

f.write("foo")
f.flush
f.rewind

puts "Created tempfile => #{f.path}"
puts "Tempfile body => #{f.read}"

# 夢幻LOOOOOOOOOOOOOOP
loop do
end

f.close
```

以下のように、このプログラムを実行すると、表示されたファイル名を他のプロセスから読み出すことができます。


```
# 端末その1で実行
ozaki@mbp-2 ( ꒪⌓꒪) $ ./tempfile_experiment1.rb
RUBY_VERSION => 2.1.1
uname => Darwin mbp-2.local 13.3.0 Darwin Kernel Version 13.3.0: Tue Jun  3 21:27:35 PDT 2014; root:xnu-2422.110.17~1/RELEASE_X86_64 x86_64
pid => 74791
Created tempfile => /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-74791-14fu29q
Tempfile body => foo


# 別な端末その2から読み込み
ozaki@mbp-2 ( ꒪⌓꒪) $ cat /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-74791-14fu29q
foo%

# 別な端末からSIGTERMで終了させる
ozaki@mbp-2 ( ꒪⌓꒪) $ kill -TERM 74791

# tempfileは削除されている
ozaki@mbp-2 ( ꒪⌓꒪) $ cat /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-74791-14fu29q
cat: /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-74791-14fu29q: No such file or directory

```

ここで、`SIGKILL`でプロセスを殺してみます。

```
# 端末その1で実行
ozaki@mbp-2 ( ꒪⌓꒪) $ ./tempfile_experiment1.rb
RUBY_VERSION => 2.1.1
uname => Darwin mbp-2.local 13.3.0 Darwin Kernel Version 13.3.0: Tue Jun  3 21:27:35 PDT 2014; root:xnu-2422.110.17~1/RELEASE_X86_64 x86_64
pid => 75487
Created tempfile => /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-75487-10b4zyz
Tempfile body => foo

# 別な端末その2から読み込み
ozaki@mbp-2 ( ꒪⌓꒪) $ cat /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-75487-10b4zyz
foo%

# 別な端末からSIGKILLで終了させる
ozaki@mbp-2 ( ꒪⌓꒪) $ kill -KILL 75487

# tempfileは残っている
ozaki@mbp-2 ( ꒪⌓꒪) $ cat /var/folders/xz/mwpm_6qj439dlsmz0rp73bd40000gp/T/hoge20140911-75487-10b4zyz
foo%
```

`SIGKILL`で終了させた場合、'ObjectScape.define_finalizer`で登録された処理は実行されず、ファイルは残ったままになります。

Rubyでの`Tempfile`でも、明示的に`unlink`を呼び出すことで、`tmpfile(3)`と同様に、確実にファイルを削除させることができるようになります。


```ruby
#!/usr/bin/env ruby

require 'tempfile'

puts "RUBY_VERSION => #{RUBY_VERSION}"
puts "uname => #{`uname -a`}"
puts "pid => #{Process.pid}"

# Tempfile作成
f = Tempfile.open("hoge")

f.write("foo")
f.flush
f.rewind

puts "Created tempfile => #{f.path}"

f.unlink # unlinkしておく
puts 'unlink'

puts "Tempfile body => #{f.read}"

# 夢幻LOOOOOOOOOOOOOOP
loop do
end

f.close
```

上記のプログラムでは、他のプロセスからファイルが見えなくなる代わりに、確実にファイルが削除されます。このことは、ドキュメントにもちゃんと書かれてあります。

[instance method Tempfile#delete](http://docs.ruby-lang.org/ja/1.9.3/method/Tempfile/i/delete.html)

# まとめ


- `tmpfile(3)`または`mkstemp(3)`を使おう
- `unlink`しておくことでプロセス終了時に削除される
- Rubyの`Tempfile`モジュールでは、プロセスが異常終了時にファイルが残る可能性がある
- 一時ファイルに期待する要件を整理し、適切なライブラリ・関数を使おうぞい

glibcとRubyでの一時ファイルの実装についてのぞいてみましたが、他の言語・ライブラリではどうなっているのか調べてみるのもよいでしょう。

<blockquote class="twitter-tweet" lang="en"><p>「添付ファイル」を意味する変数名を「tmpFile」にするのをやめろとあれほど！！！！！！</p>&mdash; ピザがっぱ (@erogappa) <a href="https://twitter.com/erogappa/status/324410285236559872">April 17, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Enjoy, tmpfile!!!

