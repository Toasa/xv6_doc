# Operating system interfaces

* user空間内の実行中のプロセスが、自身のメモリのみアクセスできるようにさせるため、kernelはCPUのhardware protection 機構を使う
* kernelはそのようなprotectionに必要なhardware privilegeも合わせて実行する
* userプログラムはそのような権限なしで実行する（？）
* userプログラムがシステムコールを呼び出すと、hardwareはprivilege levelを上げ、kernel内のpre-arranged function(?)を実行していく

* system callの集まりは、userプログラムから見えるinterface
* xv6 kernelが提供する20個のsystem call:
	* `fork()`: processの作成
	* `exit()`: 現在実行中のプロセスの終了
	* `wait()`: 子プロセスの終了を待つ
	* `kill(pid)`: プロセスIDがpidのプロセスを終了させる
	* `getpid()`: 実行中のプロセスのpidを返す
	* `sleep(n)`: n clock、sleepする
	* `exec(filename, *argv)`: fileをloadし、実行する
	* `sbrk(n)`: プロセスのメモリをn byteまで増やす
	* `open(filename, flags)`: read/write flagでfileを開く
	* `read(fd, buf, n)`: fileからn byte読みbuf
	* `write(fd, buf, n)`: nbyte書き込む
	* `close(fd)`:
	* `dup(fd)`: fdを複製する
	* `pipe(p)`: pipeを作りp内のfdを返す
	* `chdir(dirname)`:
	* `mkdir(dirname)`:
	* `mknod(name, major, minor)`: deviceファイルを作る
	* `fstat(fd)`: openしているファイルの情報を返す
	* `link(f1, f2)`: file f1に別の名前f2をつける
	* `unlink(filename)`: fileを消す


・この章の残りでxv6のサービスである、process, memory, file descripter, pipe, file systemについての概要を述べる  
・またshell(主要なuser interface)がどのように、それらのサービスを使用するかを述べる  
・shellのsystem callの使用は、上の５つのサービスがどれほど注意深く設計されたかを示す

・shellはuserプログラムである(kernelの一部ではない)、このことはsystem call interfaceの力を示している：
・つまり、shellは特別なものではなく、簡単に置き換えうるもの
・結果的に、モダンなUnix systemはさまざまなshellを持っている
・xv6のshellはUnix Bourne shellのessenceを簡単に実装したものである

## process, memory
・xv6のプロセスはuser空間のメモリ（instructino, data, stack）とkernelへのper-process state private(?)から構成される
・xv6はプロセスの実行を"time share"できる（実行待ちのプロセスがあると、利用できるCPUをスイッチする）
・プロセスが実行中でないときは、xv6はそのプロセスのCPUレジスタをsaveし、プロセスが次に実行されるとき、restoreされる
・kernelは各プロセスに対し、一意の番号、pidを付与する

・`fork()` system callを使い、プロセスは新しいプロセス、子プロセスを生成する。子プロセスは、それを呼び出したプロセス、親プロセス、と全く同じメモリ内に存在する
・`fork()` が親プロセスから呼ばれた場合、子プロセスのpidを返す、子プロセスから呼ばれた場合、０を返す

・`exit()` system callはプロセスの実行をやめさせ、リソース(メモリ、openしたfile)の開放を行う

・`wait()` sys callは現プロセスのexitした子プロセスのpidを返す。
・子を持たないプロセスがexit()を呼んだとき、wait() sys callが
・・・
・子プロセスは親とはじめは同じmemory contentを持っているが、親と子は異なるメモリとレジスタで実行される：
・変数の変化は他（のプロセス）に影響を与えない
・例えば、wait()の返り値が親プロセスのpidへとstoreされるとき、子プロセスのpidは変化しない
・子プロセスのpidは0のまま

・`exec()` はプロセスのメモリの呼び出し(calling)を、ファイルシステム内にstoreされたfileからloadされた、新しいメモリimageへと,
置き換える
・そのfileは必ず特定のformatを持たなければならない。そのformatは、fileの一部がもつ命令、一部が持つdata, 命令が始まる場所、その情報をもつ。
・xv6はelf formatを使用する
・`exec()`　が成功すると、calling programを返さない、そしてそのかわりに、fileからloadされた命令(instruction) が、elf headerで宣言されたentry pointで実行し始める
・`exec()` は２つ引数をとる：実行可能なファイルの名前とstring引数の配列
・例えば

    ```c
    char *argv[3];
	argv[0] = "echo";
	argv[1] = "hello";
	argv[2] = 0;
	exec("/bin/echo", argv);
	printf("exec error\n");


このコードはcalling programを、/bin/echoプログラムのインスタンスへと置き換える。/bin/echoプログラムは引数のlist `echo hello`とともに走る
・ほとんどのプログラムは第一引数を無視する、それは慣習的にプログラムの名前になる

・xv6のshellは、上記の呼び出しを使用し、userのためにプログラムを実行する
・shellの主な構造はsimple:

    ```c
    // Read and run input commands.
    while(getcmd(buf, sizeof(buf)) >= 0) {
        // ++++
        // 例ではecho helloなのでこのifは飛ばされる
        // ++++
        if (buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' ') {

            // Chdir must be called by the parent, not the child.
            buf[strlen(buf)−1] = 0; // chop \n
            if(chdir(buf+3) < 0)
                printf(2, "cannot cd %s\n", buf+3);
            continue;
        }
        // ++++
        // ここでfork()する
        // ++++
        if(fork1() == 0)
            runcmd(parsecmd(buf));
        wait();
    }
    exit();
    ```

・main()内のloopはuserからの入力をgetcmd()でreadする
・そして、fork()が呼ばれると、shell processのcopyがつくられる
・子プロセスがコマンドを実行中、親プロセスはwait()する

・例えば、userが `echo hello`とshellにtypeしたとき、runcmd()は`echo hello`を引数とし、呼び出される
・runcmd()は実際にコマンドを実行する
・コマンド`echo hello`はsys call exec(8626)をよぶ
・exec(8626)が成功すると、子プロセスはruncmd()のかわりにecho()を実行するだろう
・ある時点で、echoはexit()を呼び、main()内のwait()から戻り、親プロセスへとわたる
・fork()とexec()がなぜ一つのsys callではないのかと不思議に思うだろう。
+++++
何重か連続してfork()したいからでは？
+++++
プロセスを生成し、プログラムをloadするために、分割されたsys callは、cleverなデザインだと、あとで分かるだろう

+++++++++++++++++++++++++++++++++++++++++++++++++++++++
shellの動き  
・shellが動くプロセスがある（親プロセスとする）  
・親プロセスがコマンドを受け取る  
・親プロセスが子プロセスを生成するため、`fork()`を実行する  
・子プロセスがコマンドを実行する  
・コマンド実行中、親プロセスは`wait()`する  
・コマンドから`exit()`が呼ばれ、親プロセスが動き出す  
+++++++++++++++++++++++++++++++++++++++++++++++++++++++

・xv6は明示的に、ほとんどuser空間のメモリを割り当てる：
`fork()`は親が生成した子プロセスのためのメモリを割り当てる。`exec()`はfileを実行するのに十分なメモリを割り当てる
・（`malloc()`など）実行中もっとメモリが必要な際に、プロセスは`sbrk(n)`を呼び、自身のdataメモリをn byteまで増やすことができる
・`sbrk()`新しいメモリの位置を返す

・xv6には'user'という概念や、他userからとある'user'を守るという概念はない。Unixの用語を使えば、すべてのxv6のプロセスはroot userで実行される(!)

## I/O and File descriptors
・file descriptor(fd)は整数であり、プロセスがそこからread()したり、そこへwrite()するときにkernelが管理するオブジェクトを表すもの
・プロセスはfile, ディレクトリ, デバイスをopen()したり、pipeを生成したり、すでに存在するfdを生成すること、でfdを得る
・以降、簡単のため、file descriptorを"file"とする
・file descriptor interface はfile, pipe, device間の違いを抽象化し、それらをすべて、byteのstreamのように見させる

・内部では、xv6 kernelはfdをper-process table(?)内のindexをとして使用する。すべてのプロセスは0から始まるfdたちの空間を持っている(?)
・慣例により、プロセスはfd 0 (標準入力)からreadし、fd1 (標準出力)へwriteし、fd2（標準エラー出力)へエラーをwriteする
・いづれわかるが、shellは、I/O リダイレクトとpipelineの実装のために、取り決め(convention)を悪用(exploit)する
・shellはいつも3つのfdをopenにしておく、3つのfdはconsoleではデフォルトのfd

・`read()`と`write()`sys callは、fdによって名前付けされたopen files、からreadしたり、そこへwriteしたりする
・・・
・各々のfile関連のfdはoffsetを持つ
・`read()`sys callは現在のfile offsetからデータを読み、読んだbyte数分offsetを進める
・そのあとの`read()`は、はじめの`read()`から返されたbyte数に続く、byteを返していく
・readするbyteがもうないとき、`read()`はfileの終わりを示すため, 0を返す

・`write(fd, buf, n)`はbufからn byte分をfdに書き込み、書いた分のbyte数をreturnする
・エラーが発生すると、n byteより少ないbyteが書き込まれる
・`read()`同様、`write()`も現在のfile offsetにデータを書き込み、書き込んだ分のbyte数だけ、offsetを進める
・・・

catコマンドのプログラムの一部に以下がある

    ```c
    char buf[512];
    int n;
    for (;;)
        n = read(0, buf, sizeof buf);
        if (n == 0)
            break;
        if (n < 0) {
            fprintf(2, "read error\n");
            exit();
        }
        if (write(1, buf, n) != n) {
            fprintf(2, "write error\n");
            exit();
        }
    }
    ```
+++++
はじめopen()を呼んでいないから変だなと思ったが、そもそもfd=0(標準入力)はずっとopen()されているので、その必要はなかった
+++++
・重要なこととして、catコマンドは （file, console, pipeなど)どこから読んでいるのかわかっていない、ということがある
・printingも同様

・・・

・close() sys callは使用されているfdを開放する。<u>それはopen(), pipe(), dup() sys callによって再度そのfdが使用されるためである。</u>新たに割り当てられるfdは、現行プロセスで<u>使用されていない</u>最も小さい整数が、いつも割り振られる
+++++
よって直前で、close(0)を呼び、open()をすると、必ず、戻り値のfdは0になる
+++++

・fdとfork()を使えば、I/O リダイレクトは簡単に実装できる
・fork()は親プロセスのfd tableをコピーする、それは子プロセスが親と全く同じfileをopenするためである
・exec() sys callは呼び出すプロセスのメモリは置き換えるが、file tableは保持する
・これはshellの、forkによるI/O redirectの実装、を可能にしている：
選ばれたfdを（子プロセスが）再オープンし、新しいプログラムを実行する
・以下はshellがコマンド`cat < input.txt`を実行するコード：
    ```c
	char *argv[2];
	argv[0] = "cat";
	argv[1] = 0;
	if (fork() == 0) {

		// その次のopen()の戻り値でfd=0とするために、close(0)を行う
		close(0);
		open("input.txt", O_RDONLY);
		exec("cat", argv);
	}
    ```

・<u>子プロセスがfd = 0をcloseしたあと、open()により、新たに開かれたinput.txtのためのfdの使用を保証される：  
0は利用できる最小のfdとなる</u>
***
・そのとき、<u>catはinput.txtをfd=0として</u>実行する

・xv6のI/O リダイレクトに関するshellの動きは、(8630)の通り
    ```c
	case REDIR:
		rcmd = (struct redircmd*)cmd;
		close(rcmd->fd);
		if (open(rcmd->file, rcmd->mode) < 0) {
			printf(2, "open %s faied\n", rcmd->file);
			exit();
		}
		runcmd(rcmd->cmd);
		break;
    ```
・この時点で、shellは既に子shellをfork()しており、runcmd()は新しいプログラムをloadするためにexec()をよぶ
・ここで、「なぜfork()とexec()を分割した２つのsys callとすること」がなぜgood ideaなのかを述べる
・なぜならば、分割していると、shellは子プロセスをforkでき、子プロセスがopen(), close(), dup()を使うことで、stdinとstdoutのfdを変更することができ、その後にexec()できるから
・プログラムに変更がないときexec-ed(?)であることが求められる
・もし、fork()とexec()が一つのsys callなら

・・・

---
### プロセス、パイプ、リダイレクトについて

` echo hello world > hello.txt`
というコマンドのとき。
・echo コマンドが直接fileに書き込んでいるのではない


・shellは子プロセスの生成をkernelに依頼する
・kernelはfork()を実行し、shellの子プロセスを生成する

・shellはecho コマンドの実行の前に、子プロセスのfd=1の出力をhello.txtにむけるように、kernelに依頼する
・kernelはhello.txtのファイルパスや、どこまで読み書きしたか(offset)などのファイル管理情報メモを作成する（`open()`）
・kernelはファイルの管理情報メモに番号を振り管理している(fd)、仮にここで5とする。(`open()`の戻り値が5だった)
・kernelはリダイレクト指定に従い、fd=1の管理情報メモに、fd=5の管理情報メモをコピーする(dup2(fd=5, 1))
・以降、kernelはfd=1の出力依頼に対し、fd=5のhello.txtに書き込むようになる
・kernelはfd=5(hello.txt)と紐づく管理情報メモを破棄する(`close()`)
	→fd=1の管理情報メモがあるので、同内容のfd=5の管理情報メモは不要
・kernelは以上の作業を行い、shellにもどる

・shellはechoコマンドの実行をkernelに依頼する
・kernelは子プロセスをechoコマンドに置き換え実行する

・echoコマンドは、kernelにfd=1への出力を依頼する
・kernelはfd=1の管理情報メモを見て、fd=5のhello.txtへ書き込む

---

#### fork()の仕組み  
・fork()は実行中のプロセスをコピーする。
・コピー後、親プロセスには、コピーし生成した、子プロセス
のpidを返す  
・子プロセスには0を返す  
・エラーが出たら、-1を返す  

---

#### リダイレクトの内部処理  
・例: `echo hello world > hello.txt`
    ```c
	pid = fork();
	if (pid == 0) {
		// 子プロセス
		fd = open("hello.txt, O_CREAT | O_WRONLY, 0666);

		// 1(標準出力)はfdと同じファイルを参照するようになる
		dup2(fd, 1);
		close(fd);
		execl("/bin/echo", "hello", "world", NULL);
	} else {
		// 親プロセス
		wait();
	}
    ```
---

#### パイプの内部処理  
・例 `echo hello world | wc`
    ```c
	// fd[1] ===> pipe ===> fd[0]
	int fd[2];
	pipe(fd);
	if (fork() == 0) {
		// pipeの入り口を標準出力にコピー
        // 1(標準出力)はfd[1]と同じファイルを参照する
		dup2(fd[1], 1);
		close(fd[0]));
		close(fd[1]));

		// echo hello worldの実行
		execl("/bin/echo", "hello", "world", NULL);
	} else if (fork() == 0) {
		// pipeの出口を標準入力にコピー
        // 0(標準入力)はfd[0]と同じファイルを参照する
		dup2(fd[0], 0);
		close(fd[0]));
		close(fd[1]));

		// wcの実行
		execl("/bin/wc", NULL);
	} else {
		// 親プロセス
		close(fd[0]);
		close(fd[1]);
		wait();
		wait();
	}
    ```
---

・`dup()` sys callは今あるfdを複製する。  
・２つのfdはoffsetを共有する、これはfork()によってfdを生成したときと同じ挙動である
・以下はhello worldが出力される
```c
    fd = dup(1);
    write(1, "hello", 6);
    write(fd, "world\n", 6);
```

・2つのfdはoffsetを共有する, `fork()`または`dup()`によってもとのfdからそれらが生成されたときに  
・一方で同じファイルに対し`open()`をしても、offsetは共有されない
・`dup()`によりshellは次のようなコマンドを実行できる
```
ls file > tmp1 2>&1
```
・`2>&1`はshellに, fd=2はfd=1から複製(`dup()`)されたことを伝える
・つまり上のコマンドは、`ls`の標準出力をfileへ出力し、標準エラー出力を標準出力へだす

## Pipes
・pipeとは小さなkernel buffer
    ```c
    int p[2];
    char *argv[2];

    argv[0] = "wc";
    argv[1] = 0;

    pipe(p);
    if (fork() == 0) {
        close(0);
        dup(p[0]);
        close(p[0]);
        close(p[1]);
        exec("/bin/wc", argv);
    } else {
        close(p[0]);
        write(p[1], "hello world\n", 12);
        close(p[1]);
    }
    ```
`pipe()`sys callは新しいパイプと配列p内のreadとwriteのfdのrecordを生成する。
・子プロセスはreadの終了をfd = 0に複製し、fdをcloseする、そして`wc`を実行する。`wc`が標準入力からreadするとき、それはpipeからreadする。
・親プロセスはpipeのread側をcloseし、pipeに書き込み、pipeのwrite側をcloseする

## File system
・xv6のfile systemはuninterpretedなバイト配列であるデータファイルと、ディレクトリという。ディレクトリは他のデータファイルや他のディレクトリへの参照。ディレクトリはtree構造をしている。

・`mknod()`で新しいデバイスファイルを生成する
    ```c
    mkdir("/dir");
    fd = open("/dir/file", O_CREATE | O_WRONLY);
    close(fd);
    mknod("/console", 1, 1);
    ```
`mknod()`はfile systemの中にファイルを生成する。しかし、fileには中身はない。そのかわり、fileのメタデータは

`fstat()`はfdが指していたオブジェクトについての情報を取り出す。

・inodeはlinkにより、複数の名前を持てる
・`link()` sys callは同じinodeのfileに別の名前をつける

・・・

・file systemのためのshellコマンドはuser levelのプログラムとして実装されている。mkdir, ln, rmなど。
・このデザインは誰でもshellにuser commandを追加することで拡張できるようにしている。
・後からみると、このような工夫は当たり前の用に見える。しかし、Unixと同時期にデザインされた他のOSは、そのようなuser commandをshellの中に組み込んでしまった。（そしてshellをkernelの中にいれてしまった）
・一つ例外がある、
+++++
つまり、user levelではないshell commandがある
+++++
それは`cd`であり、shellの中にbuilt-inされている。
・`cd`はshell自身の現在のworkingディレクトリを変更しないといけない。
・もし`cd`が単にregular commandであったならば、shellは子プロセスをforkし、子プロセスが`cd`を実行し、`cd`コマンドは**子プロセス**のworkingディレクトリを変更するだろう。
・そして、親プロセス（つまり、shell）のworkingディレクトリは変化しないままになる


## Real world
