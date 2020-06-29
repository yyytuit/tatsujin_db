# 既存のコード体系を新しい体系に変換して集計する

## 非定型的な集計を行う業務で、既存コード体系を分析用のコード体系に変形して、その新体系の単位で集計したいという要件。

- 北海道、青森、、、沖縄という県単位で人口が記録されているテーブルがあった場合、これを東北、関東、九州といった地方単位で人口を集計したい場合。
  ![CKk%UEcgQGK9Dw0A0Jz91w_thumb_49](https://user-images.githubusercontent.com/51355545/86020075-69094d00-ba62-11ea-99e8-a6feb60bce9f.jpg)

  こんなとき、地方コードという列を持つビューを定義する、というのも一つの方法。

  しかし、それだと集計に使いたいコード体系の数だけ列を追加しなければならないのと、アドホック(特定の目的へ)な変更も困難。

  case 式を使うと、次のように記述できる

  ```sql
  select * from PopTbl;
  pref_name | population
  -----------+------------
  徳島      |        100
  香川      |        200
  愛媛      |        150
  高知      |        200
  福岡      |        300
  佐賀      |        100
  長崎      |        200
  東京      |        400
  群馬      |         50
  (9 rows)

  SELECT CASE pref_name
          WHEN '徳島' THEN '四国'
          WHEN '香川' THEN '四国'
          WHEN '愛媛' THEN '四国'
          WHEN '高知' THEN '四国'
          WHEN '福岡' THEN '九州'
          WHEN '佐賀' THEN '九州'
          WHEN '長崎' THEN '九州'
          ELSE 'その他' END AS district,
       SUM(population)
  FROM PopTbl
  GROUP BY CASE pref_name
             WHEN '徳島' THEN '四国'
             WHEN '香川' THEN '四国'
             WHEN '愛媛' THEN '四国'
             WHEN '高知' THEN '四国'
             WHEN '福岡' THEN '九州'
             WHEN '佐賀' THEN '九州'
             WHEN '長崎' THEN '九州'
             ELSE 'その他' END;
  district | sum
  ----------+-----
  九州     | 600
  四国     | 650
  その他   | 450
  (3 rows)
  ```

  - 人口階級ごとに都道府県の数を調べたい場合。

  ```sql
  SELECT CASE WHEN population < 100 THEN '01'
            WHEN population >= 100 AND population < 200 THEN '02'
            WHEN population >= 200 AND population < 300 THEN '03'
            WHEN population >= 300 THEN '04'
            ELSE NULL END AS pop_class,
       COUNT(*) AS cnt
  FROM PopTbl
  GROUP BY CASE WHEN population < 100 THEN '01'
               WHEN population >= 100 AND population < 200 THEN '02'
               WHEN population >= 200 AND population < 300 THEN '03'
               WHEN population >= 300 THEN '04'
               ELSE NULL END
  ORDER BY pop_class asc;

  pop_class | cnt
  -----------+-----
  01        |   1
  02        |   3
  03        |   3
  04        |   2
  (4 rows)
  ```

* 上記は大変便利だが、SELECT 句と GROUP BY 句の２箇所に CASE 式を書かなければならない。

  修正が入った場合、片方のみ修正し、もう片方忘れてしまう可能性がある。

  なのでその２つをまとめて書くと以下のようになる。

  ```sql
  SELECT CASE pref_name
          WHEN '徳島' THEN '四国'
          WHEN '香川' THEN '四国'
          WHEN '愛媛' THEN '四国'
          WHEN '高知' THEN '四国'
          WHEN '福岡' THEN '九州'
          WHEN '佐賀' THEN '九州'
          WHEN '長崎' THEN '九州'
          ELSE 'その他' END AS district,
       SUM(population)
  FROM PopTbl
  GROUP BY district;

   district | sum
  ----------+-----
  九州     | 600
  四国     | 650
  その他   | 450
  (3 rows)
  ```

- 上記のように district で GROUP BY 句を使うことができる。

  ただし、これは PostgreSQL と MySQl では使えるが、Oracle や Db2、SQL Server では使えない。

# 異なる条件の集計を 1 つの SQL で行う

- 異なる条件の集計といのは、例えば県別人口を保持するテーブルに性別列を加えたテーブルから、男女別・県別の人数の合計を求めるというケース。

![X8%3imixQQisu7%BVYSoRA_thumb_4a](https://user-images.githubusercontent.com/51355545/86023444-a5d74300-ba66-11ea-83b5-33ee71bb5e9b.jpg)

```sql
SELECT pref_name,
       population
  FROM PopTbl2
 WHERE sex = '1';
pref_name | population
-----------+------------
 徳島      |         60
 香川      |        100
 愛媛      |        100
 高知      |        100
 福岡      |        100
 佐賀      |         20
 長崎      |        125
 東京      |        250
(8 rows)


SELECT pref_name,
       population
  FROM PopTbl2
 WHERE sex = '2';
 pref_name | population
-----------+------------
 徳島      |         40
 香川      |        100
 愛媛      |         50
 高知      |        100
 福岡      |        200
 佐賀      |         80
 長崎      |        125
 東京      |        150
(8 rows)

SELECT pref_name,
       --男性の人口
       SUM( CASE WHEN sex = '1' THEN population ELSE 0 END) AS cnt_m,
       --女性の人口
       SUM( CASE WHEN sex = '2' THEN population ELSE 0 END) AS cnt_f
  FROM PopTbl2
 GROUP BY pref_name;
pref_name | cnt_m | cnt_f
-----------+-------+-------
 東京      |   250 |   150
 高知      |   100 |   100
 徳島      |    60 |    40
 香川      |   100 |   100
 長崎      |   125 |   125
 愛媛      |   100 |    50
 佐賀      |    20 |    80
 福岡      |   100 |   200
(8 rows)
```

- この SQL のいい点は 2 次元表の形に整形できること。

  単純に GROUP BY で集約しただけだと、ホスト言語や Excel などのアプリケーション上でクロス表の形に整形しなければいけない。

  しかし、表側が県名、表頭が性別というクロス表の形式で結果が出力されている。

  集計表を作るときに非常に便利。

  WHERE 句で条件分岐させるのは素人のやること。プロは SELECT 句で分岐させる

* 人口の合計を計算しているのではないが、なぜ SUM が必要か？

  以下のコードのようにレコードの集約は行われないので、元の PopTbl2 テーブルのレコード数そのままが結果に出てくることになる。

```sql
SELECT pref_name,
       --男性の人口
      CASE WHEN sex = '1' THEN population ELSE 0 END AS cnt_m,
       --女性の人口
      CASE WHEN sex = '2' THEN population ELSE 0 END AS cnt_f
  FROM PopTbl2;
pref_name | cnt_m | cnt_f
-----------+-------+-------
 徳島      |    60 |     0
 徳島      |     0 |    40
 香川      |   100 |     0
 香川      |     0 |   100
 愛媛      |   100 |     0
 愛媛      |     0 |    50
 高知      |   100 |     0
 高知      |     0 |   100
 福岡      |   100 |     0
 福岡      |     0 |   200
 佐賀      |    20 |     0
 佐賀      |     0 |    80
 長崎      |   125 |     0
 長崎      |     0 |   125
 東京      |   250 |     0
 東京      |     0 |   150
(16 rows)
```

## CHECK 制約で複数の列の条件関係を定義する

- CHECK 制約と CASE 式の組み合わせは非常に相性がいい。

- 例えば「女性社員の給料は 20 万円以下」という給与体系を持つ会社があったとする。

```sql
CONSTRAINT check_salary CHECK
   ( CASE WHEN sex = '2'
          THEN CASE WHEN salary <= 200000
                    THEN 1 ELSE 0 END
     ELSE 1 END = 1 )
```

- 上記のように CASE 式を入れ子にして、「社員の性別が女性ならば、給料は 20 万以下である」という命題を表現している。

  これは命題論理で条件法(conditional)と呼ばれる論理式で、形式的に書けば「P→Q」となる。

* ただし、条件法と論理積は違うので注意が必要

  論理積とは「P かつ Q」を意味する。

  CHECK 制約では以下のようになる

```sql
CONSTRAINT check_salary CHECK
   ( sex = '2' AND salary <= 200000 )
```

- では条件法と論理積は何が違うのだろう。

  それは条件法は男性も働けるが、論理積だと男性は働けない。

  ![7hj73TXlTZOCecW5bucjPw_thumb_4b](https://user-images.githubusercontent.com/51355545/86028680-28630100-ba6d-11ea-97d4-1665fbd9febc.jpg)

## 条件分岐させた UPDATE

- 数値型の列に対して、現在の値を判定対象として別の値へ変えたいというケースを考える。

  問題は UPDATE の条件が複数に分岐するケース。

  社員の給料を格納する人事部のテーブルを使うとする

```sql
CREATE TABLE Personnel
(name VARCHAR(32) PRIMARY KEY,
salary INTEGER NOT NULL);

INSERT INTO Personnel VALUES('相田', 270000);
INSERT INTO Personnel VALUES('神崎', 324000);
INSERT INTO Personnel VALUES('木村', 220000);
INSERT INTO Personnel VALUES('斉藤', 290000);

select * from Personnel;
 name | salary
------+---------
 相田 | 270000
 神崎 | 324000
 木村 | 220000
 斉藤 | 290000
```

1. 現在の給与が 30 万以上の社員は、10%の減給
1. 現在の給与が 25 万以上 28 万未満の社員は、20%昇給とする

```sql
--条件1
UPDATE Personnel
   SET salary = salary * 0.9
 WHERE salary >= 300000;


--条件2
UPDATE Personnel
   SET salary = salary * 1.2
 WHERE salary >= 250000 AND salary < 280000;
```

- この式だと間違っている。なぜなら、条件 1 で 30 万以上の給料の人が 27 万になる。そうすると条件 2 でもヒットし、32 万となり大幅アップしてしまうのである。

  逆に書いても同じである。

* なので以下のような CASE 分を書くことでこの問題を解決できる

```sql
UPDATE Personnel
   SET salary = CASE WHEN salary >= 300000
                     THEN salary * 0.9
                     WHEN salary >= 250000 AND salary < 280000
                     THEN salary * 1.2
                ELSE salary END;
select * from Personnel;
 name | salary
------+--------
 相田 | 324000
 神崎 | 291600
 木村 | 220000
 斉藤 | 290000
(4 rows)
```

- 上記のように記述することで、1 文で書くことができる。

  ただし、ELSE 後の salary を書かないと、条件に当てはまらなかった社員の給料が NULL になってしまうので注意が必要。

* このテクニックを使うと主キーの値を入れ替えるという荒技もできる。

  普通 a と b という主キーの値を入れ替えるためには、ワーク用の値へ一度どちらかを退避させる必要がある。

```sql
CREATE TABLE SomeTable
(p_key CHAR(1) PRIMARY KEY,
 col_1 INTEGER NOT NULL,
 col_2 CHAR(2) NOT NULL);

INSERT INTO SomeTable VALUES('a', 1, 'あ');
INSERT INTO SomeTable VALUES('b', 2, 'い');
INSERT INTO SomeTable VALUES('c', 3, 'う');

SELECT * FROM SomeTable;
 p_key | col_1 | col_2
-------+-------+-------
 a     |     1 | あ
 b     |     2 | い
 c     |     3 | う
(3 rows)
```

- 上記のような場合、a と b を入れ替えるには次のような SQL を書く必要がある

```sql
--1．aをワーク用の値dへ退避
UPDATE SomeTable
   SET p_key = 'd'
 WHERE p_key = 'a';

--2．bをaへ変換
UPDATE SomeTable
   SET p_key = 'a'
 WHERE p_key = 'b';

--3．dをbへ変換
UPDATE SomeTable
   SET p_key = 'b'
 WHERE p_key = 'd';

 SELECT * FROM SomeTable;
 p_key | col_1 | col_2
-------+-------+-------
 c     |     3 | う
 a     |     2 | い
 b     |     1 | あ
```

- まず無駄が多いということと、d も常に利用できるかわからない。

```sql
UPDATE SomeTable
   SET p_key = CASE WHEN p_key = 'a'
                    THEN 'b'
                    WHEN p_key = 'b'
                    THEN 'a'
               ELSE p_key END
 WHERE p_key IN ('a', 'b');
ERROR:  duplicate key value violates unique constraint "sometable_pkey"
DETAIL:  Key (p_key)=(b) already exists.
```

- 本書で上記を紹介しているが、どうやら主キーの入れ替えはエラーがでる。

## テーブル同士のマッチング
