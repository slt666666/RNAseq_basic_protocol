# RNAseq解析基礎

# 必要なソフト
## ソフトを入れるためのツール
入れておくと以降のソフトやプログラムをインストールしやすくなる。)
* miniconda
* Anacondaでも可能

## リードのQuality Control
* fastp 
* FaQCs, FastQCなどでも可能

## gff file, gtf  fileの扱い
* gffread

## mapping
* Hisat2
* STARなど他にもいろいろある

## mapping後のファイル(samファイル, bamファイル)の扱い
* samtools

## bamファイルから遺伝子毎のカウントなど
* subread(featureCountsなどの機能を含む)

## (発現差異解析(DEG)などをしたければ)  
* Rパッケージ"edgeR"
* Rパッケージ"DESeq2"
* その他統計検定用のパッケージなど

(これらのパッケージを自作のスクリプトで動かす。)
<br><br>

***
***

# 実際にコマンドを動かす
(Macなら主にターミナル上でコマンドを使用する)
## 基本的なコマンド
* ls
* cd
* mkdir
* mv
* cp

## bashスクリプト
以下紹介していくコマンドはMacのターミナルで直接打ち込んでも良いが、
XXX.shというファイルにコマンドを記述しておき、
~~~
bash XXX.sh
~~~
のように実行することで、XXX.shに記述したコマンド群を実行することが出来る。
後々再実行などするとき用にスクリプトとして残しておく方がベター

## 1. fastqデータのQuality control
* fastpを使うのであれば(オプションは任意のものを設定してください)
~~~
fastp -i pair1.fq -I pair2.fq -3 -o out_pair1.fq.gz -O out_pair2.fq.gz -h report.html -q 15 -n 10 -t 1 -T 1 -l 20
~~~

ターミナルで直接動かさず、bashスクリプトに記述する場合は以下の様に改行して見やすくできる。
~~~
fastp \
    -i pair1.fq.gz \
    -I pair2.fq.gz \
    -3 \
    -o out_pair1.fq.gz \
    -O out_pair2.fq.gz \
    -h report.html \
    -q 15 \
    -n 10 \
    -t 1 \
    -T 1 \
    -l 20 \
~~~

* FaQCsを使うのであれば
~~~
FaQCs \
    -1 pair1.fq.gz \
    -2 pair2.fq.gz \
    -t 8 \
    -min_L 50 \
    -d output/ \
    -discard 1 \
    --polyA \
    -avg_q 20
~~~

## 2. リード(fastq)をリファレンスゲノムにマッピング(Hisat2の場合)
### 2.1 リファレンスのgffファイルをgtf(が手に入らない場合)に変換する
~~~
gffread Niben.genome.v0.4.4.gene_models.gff -T -o Niben.genome.v0.4.4.gene_models.gtf
~~~

### 2.2 (Hisat2を使用する場合) exon & splice siteのファイルを作成
~~~
hisat2_extract_exons.py Niben.genome.v0.4.4.gene_models.gtf > Niben044_exon.txt 
hisat2_extract_splice_sites.py Niben.genome.v0.4.4.gene_models.gtf > Niben044_ss.txt
~~~

### 2.3 リファレンスゲノムのindexファイルを作成(ゲノムサイズによっては死ぬほど時間がかかる)
~~~
hisat2-build \
    --ss Niben044_ss.txt \
    --exon Niben044_exon.txt \
    Niben.genome.v0.4.4.scaffolds.nrcontigs.fasta \
    Niben044_index
~~~

### 2.4 マッピング
~~~
hisat2 \
    -k 1 -p 12 \
    --no-mixed \
    --no-discordant \
    -x Niben044_index \
    -1 out_pair1.fq.gz \
    -2 out_pair2.fq.gz \
    -S out.sam
~~~

### 2.5 samファイル(はサイズがデカいので)をbamファイルに変換
~~~
samtools view -F 004 -bS out.sam > out.bam
~~~

samファイルは非常にサイズがデカいので、"いったんsamファイルを作成し、bamファイルに変換する"という手順より、いきなりbamファイルとして作成したい時が多い。そういう場合はパイプというものを使う。2.4と2.5のコマンドをパイプ `|` でつなぐことで、2.4の出力結果をそのまま2.5のコマンドに適用するみたいな感じ。これでsamファイルを一旦作成することなく直接bamファイルが作成できる。

~~~
hisat2 \
    -k 1 -p 12 \
    --no-mixed \
    --no-discordant \
    -x Niben044_index \
    -1 out_pair1.fq.gz \
    -2 out_pair2.fq.gz | samtools view -F 004 -bS - > out.bam
~~~

### 2.6 bamファイルをソート & index作成(他のプログラム(featureCountsとか)を使うために必要)
~~~
samtools sort out.bam -o out.sort.bam
samtools index out.sort.bam
~~~

## 3. 遺伝子毎などの単位でread count 
~~~
featureCounts \
    -p \
    -t exon \
    -g gene_id \
    -a Niben.genome.v0.4.4.gene_models.gtf \
    -o read_count.txt \
    out.sort.bam
~~~

## 4. TPMやFPKM等の計算、DEG解析など
Rスクリプトやpythonスクリプト
