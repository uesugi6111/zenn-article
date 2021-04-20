---
title: "JavaからRustをJNA(Java Native Access)で実行する"
emoji: "👏"
type: "tech" 
topics: ["Java","Rust","JNA","Maven","ffi"]
published: false
---
## はじめに
### 動機
システムプログラミング言語(?)であるRustの話を聞いていると、FFI([Foreign function interface](https://ja.wikipedia.org/wiki/Foreign_function_interface "FFI"))の話が少なからず出てきます。
ということで普段書いているJavaから呼び出します。

### 対象読者
JavaとRustが多少読め、MavenというJava用のプロジェクト管理用ツールが存在していることを知っている方。
難しいことはしておらず、プロジェクトの雛形を作成したレベルの話です。

## 環境
開発環境はVSCodeを使用します。
RustとJavaの実行環境はDockerで構築します。
**DockerFiile**
ベースはMicrosoft提供のJavaの開発環境のサンプル(Debean 10)を、フォークして自分用に弄ったものを使用しています。変更箇所としてはJavaのversionを14→11に変更した程度です。
[弄ったもの](https://github.com/uesugi6111/vscode-remote-try-java/blob/master/.devcontainer/Dockerfile "DockerFile")

そこにRustのコンパイラをインストールします。以下を追記。


```Docker:DockerFiile
ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH

RUN set -eux; \
    \
    url="https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init"; \
    wget "$url"; \
    chmod +x rustup-init; \
    ./rustup-init -y --no-modify-path --default-toolchain nightly; \
    rm rustup-init; \
    chmod -R a+w $RUSTUP_HOME $CARGO_HOME; \
    rustup --version; \
    cargo --version; \
    rustc --version;

RUN apt-get update &&  apt-get install -y lldb python3-minimal libpython3.7 python3-dev gcc \
    && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/* /tmp/library-scripts

```

内容としては
- 環境変数の追加
- 必要なrustのコンポーネントをインストール
- Rustで必要になるデバッガ、python、gccのインストール
を行っています

インストールするRustをここで**nightly**としている理由は後述します。
## Rust
今回Rust側で作成したファイルは以下です

```:tree
workspace
│  Cargo.toml
│
├─sample-jna
│  │  Cargo.toml
│  │
│  └─src
│          lib.rs
│
└─scripts
       cargo-build.sh
```

workspaceのトップレベルでcargoのコマンドが使いたかったため、このような構成となりました。

以下解説
### Cargo.toml
```toml:Cargo.toml
[workspace]
members = ["sample-jna"]

[profile.release]
lto = true
```
上2行でworkspace内のsample-jnaディレクトリをプロジェクトとして認識しています。
**lto = true**はbuild時のファイルサイズ削減用のオプションです。

### sample-jna/Cargo.toml
```toml:Cargo.toml
[package]
name = "sample-jna"
version = "0.1.0"
authors = ["uesugi6111 <59960488+aburaya6111@users.noreply.github.com>"]
edition = "2018"

[lib]
crate-type = ["cdylib"]
```
**[package]**は**cargo new** で作成されるもので問題ありません。
**[lib]**の**crate-type**がコンパイル後のタイプとなります。別言語から呼び出す想定のダイナミックライブラリは、**cdylib**を指定するよう[リファレンス](https://doc.rust-lang.org/reference/linkage.html "linkage")に書かれているので、それに従います。

### lib.rs
ここがライブラリ本体となります。今回は以前書いて残していたエラトステネスの篩に似たアルゴリズムで引数までの素数を列挙し、その個数を返すだけのプログラムを用意しました。

```Rust:lib.rs
#[no_mangle]
pub extern fn sieve_liner(n: i32) -> i32{
    let mut primes = vec![];
    let mut d = vec![0i32; n as usize + 1];
    
    for i in 2..n + 1 {
        if d[i as usize] == 0 {
            primes.push(i);
            d[i as usize] = i;
        }
        for p in &primes {
            if p * i > n {
                break;
            } 
            d[(*p * i) as usize] = *p;
        }
    }
    
    primes.len() as i32
}
```
通常のコンパイルでは関数名は他の名称に変換されてしまい、ほかプログラムなどから呼び出す際に、名前がわからなくなってしまいます。それを防ぐために**#[no_mangle]**(直訳:切り刻み無し)を関数に付与します。

### cargo-build.sh
```shell:cargo-build.sh
#!/bin/bash
cargo build --release -Z unstable-options --out-dir ./src/main/resources
```
ライブラリのbuildスクリプトになります。
**--release** 
releaseオプションでのbuildを指定します。
**-Z unstable-options --out-dir ./src/main/resources**
build後に出力するディレクトリを指定するオプションとなっています。しかしこのオプションが使えるのは**nightly**のみとなっています。
そのためDockerで構築する環境へのインストールは、**nightly**指定にしています。

ディレクトリの指定先はJava側でコンパイルされたときにjarファイル内に配置される場所に設定しました。

## Java
Java側で作成したファイルは以下になります。

```
workspace
│  pom.xml
└─src
   └─main
      ├─java
      │  └─com
      │      └─mycompany
      │          └─app
      │                  App.java
      │
      └─resources
```
やけにディレクトリが深いですが、特に意味はありません。

### pom.xml
以下を**\<dependencies>**へ追記します

```xml:pom.xml
    <dependency>
        <groupId>net.java.dev.jna</groupId>
        <artifactId>jna</artifactId>
        <version>5.6.0</version>
    </dependency>
```


### App.java
```Java:App.java

package com.mycompany.app;

import java.util.ArrayList;
import java.util.List;

import com.sun.jna.Library;
import com.sun.jna.Native;

public class App {
    private static final int N = 100000000;

    public interface SampleJna extends Library {
        SampleJna INSTANCE = Native.load("/libsample_jna.so", SampleJna.class);

        int sieve_liner(int value);
    };

    public static void main(String[] args) {
        System.out.println("N = " + N);
        System.out.println("FFI  :" + executeFFI(N) + "ms");
        System.out.println("Java :" + executeJava(N) + "ms");

    }

    public static long executeFFI(int n) {
        long startTime = System.currentTimeMillis();
        SampleJna.INSTANCE.sieve_liner(n);
        return System.currentTimeMillis() - startTime;

    }

    public static long executeJava(int n) {
        long startTime = System.currentTimeMillis();
        sieveLiner(n);
        return System.currentTimeMillis() - startTime;

    }

    public static int sieveLiner(int n) {
        List<Integer> primes = new ArrayList<>();

        int d[] = new int[n + 1];
        for (int i = 2; i < n + 1; ++i) {

            if (d[i] == 0) {
                primes.add(i);
                d[i] = i;
            }
            for (int p : primes) {
                if (p * i > n) {
                    break;
                }
                d[p * i] = p;
            }
        }

        return primes.size();
    }

}

```
Rust側に実装したロジックと同様のものを実装し、実行時間を比較します。
ライブラリの呼び出しは
**SampleJna INSTANCE = Native.load("/libsample_jna.so", SampleJna.class);**
で行っています。
Native.load(ライブラリのPath,ライブラリを定義したのinterface)という形で記述するようです。
今回ライブラリはmain/resources直下に配置する予定なので絶対パス(?)で表記しています。

## Maven
ここまでで本来動作確認はできるのですが、jarにすることを考えた際の設定もしてみました。
jarを作成するまでの流れ
- Rust をコンパイルしてJava側のresourcesディレクトリに配置
- Java側のコンパイル

これをMavenの機能を利用し、ワンアクションで行います。

### maven-assembly-plugin
jarに依存ライブラリを含める

```xml:maven-assembly-plugin
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <archive>
            <manifest>
              <addClasspath>true</addClasspath>
              <classpathPrefix>/</classpathPrefix>
              <mainClass>com.mycompany.app.App</mainClass>
            </manifest>
          </archive>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```

### exec-maven-plugin
maven の処理中でシェルスクリプトを実行するために必要となります。


```xml:exec-maven-plugin
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>1.3.2</version>
        <executions>
          <execution>
            <id>dependencies</id>
            <phase>generate-resources</phase>
            <goals>
              <goal>exec</goal>
            </goals>
            <configuration>
              <workingDirectory>${project.basedir}</workingDirectory>
              <executable>${project.basedir}/scripts/cargo-build.sh </executable>
            </configuration>
          </execution>
        </executions>
      </plugin>
```
少し解説
**phase**　
シェルスクリプトを実行するタイミングを設定します。Mavenではライフサイクルという概念が存在するので、実行したいタイミングに合ったものを指定します。[リファレンス](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference "Lifecycle_Reference")
**executable**
ここで実行したい対象を指定します。




### pom.xml
ここまで適応し終えたファイル

```xml:pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.mycompany.app</groupId>
  <artifactId>my-app</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>my-app</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter-api</artifactId>
      <version>5.7.0</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>net.java.dev.jna</groupId>
      <artifactId>jna</artifactId>
      <version>5.6.0</version>
    </dependency>
  </dependencies>
  <properties>
    <jdk.version>11</jdk.version>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <archive>
            <manifest>
              <addClasspath>true</addClasspath>
              <classpathPrefix>/</classpathPrefix>
              <mainClass>com.mycompany.app.App</mainClass>
            </manifest>
          </archive>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.0.0-M3</version>
      </plugin>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>1.3.2</version>
        <executions>
          <execution>
            <id>dependencies</id>
            <phase>generate-resources</phase>
            <goals>
              <goal>exec</goal>
            </goals>
            <configuration>
              <workingDirectory>${project.basedir}</workingDirectory>
              <executable>${project.basedir}/scripts/cargo-build.sh </executable>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

## 実行
workspaceのルートで以下を実行

```shell
mvn package
```
すると

```
[INFO] --- maven-assembly-plugin:3.3.0:single (make-assembly) @ my-app ---
[INFO] Building jar: /workspace/target/my-app-1.0-SNAPSHOT-jar-with-dependencies.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```
のようなログが出力されてコンパイルが完了します。

あとは表示されたパスに出力されているjarを実行してください。

```shell:例
java -jar ./target/my-app-1.0-SNAPSHOT-jar-with-dependencies.jar
```
出力

```
N = 100000000
FFI  :1668ms
Java :3663ms
```
10^8までの素数の数のカウントでかかった、JavaとFFI(Rust)の時間(ms)が出力されました。
Nを小さくするとJavaの方が早くなるため、大きなオーバーヘッドがあるのでしょうかわかりません。

# さいご
とりあえずは動いたということで一旦完了とします。
使ったソースになります
https://github.com/uesugi6111/java-rust

普段触らない部分の話が多くまだわからないことが多いですがぼちぼち調べていきます。
*謎*
- JNA以外のJavaからの呼び出し方法
- Rustの関数での単純な数値以外の返し方