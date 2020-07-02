# 既存のコード体系を新しい体系に変換して集計する

## 非定型的な集計を行う業務で、既存コード体系を分析用のコード体系に変形して、その新体系の単位で集計したいという要件。

- 北海道、青森、、、沖縄という県単位で人口が記録されているテーブルがあった場合、これを東北、関東、九州といった地方単位で人口を集計したい場合。
  ![CKk%UEcgQGK9Dw0A0Jz91w_thumb_49](https://user-images.githubusercontent.com/51355545/86020075-69094d00-ba62-11ea-99e8-a6feb60bce9f.jpg)

  こんなとき、地方コードという列を持つビューを定義する、というのも一つの方法。

  しかし、それだと集計に使いたいコード体系の数だけ列を追加しなければならないのと、アドホック(特定の目的へ)な変更も困難。

  case 式を使うと、次のように記述できる

  ```sql
  SELECT * from PopTbl;
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

- 次のような資格予備校の講座一覧に関するテーブルと月々に開講されている講座を管理するテーブルを考える。

![スクリーンショット 2020-07-01 23 27 17](https://user-images.githubusercontent.com/51355545/86255789-83723080-bbf2-11ea-8407-f955b71043e4.png)

```sql
CREATE TABLE CourseMaster
(course_id   INTEGER PRIMARY KEY,
 course_name VARCHAR(32) NOT NULL);

INSERT INTO CourseMaster VALUES(1, '経理入門');
INSERT INTO CourseMaster VALUES(2, '財務知識');
INSERT INTO CourseMaster VALUES(3, '簿記検定');
INSERT INTO CourseMaster VALUES(4, '税理士');

SELECT * FROM CourseMaster;
 course_id | course_name
-----------+-------------
         1 | 経理入門
         2 | 財務知識
         3 | 簿記検定
         4 | 税理士
(4 rows)

CREATE TABLE OpenCourses
(month       INTEGER ,
 course_id   INTEGER ,
    PRIMARY KEY(month, course_id));

INSERT INTO OpenCourses VALUES(200706, 1);
INSERT INTO OpenCourses VALUES(200706, 3);
INSERT INTO OpenCourses VALUES(200706, 4);
INSERT INTO OpenCourses VALUES(200707, 4);
INSERT INTO OpenCourses VALUES(200708, 2);
INSERT INTO OpenCourses VALUES(200708, 4);

SELECT * FROM OpenCourses;
 month  | course_id
--------+-----------
 201806 |         1
 201806 |         3
 201806 |         4
 201807 |         4
 201808 |         2
 201808 |         4
(6 rows)
```

OpenCourses のある月に、CourseMaster テーブルの講座が存在するかどうかのチェックを行う。
このマッチングの条件を CASE 式によって書くことができる。

```sql
--テーブルのマッチング：IN述語の利用
SELECT course_name,
       CASE WHEN course_id IN
                    (SELECT course_id FROM OpenCourses
                      WHERE month = 201806) THEN '○'
            ELSE '×' END AS "6 月",
       CASE WHEN course_id IN
                    (SELECT course_id FROM OpenCourses
                      WHERE month = 201807) THEN '○'
            ELSE '×' END AS "7 月",
       CASE WHEN course_id IN
                    (SELECT course_id FROM OpenCourses
                      WHERE month = 201808) THEN '○'
            ELSE '×' END AS "8 月"
  FROM CourseMaster;

          ELSE '×' END AS "8 月"
postgres-#   FROM CourseMaster;
 course_name | 6 月 | 7 月 | 8 月
-------------+------+------+------
 経理入門    | ○    | ×    | ×
 財務知識    | ×    | ×    | ○
 簿記検定    | ○    | ×    | ×
 税理士      | ○    | ○    | ○
(4 rows)
```

```sql
--テーブルのマッチング：EXISTS述語の利用
SELECT CM.course_name,
       CASE WHEN EXISTS
                  (SELECT course_id FROM OpenCourses OC
                    WHERE month = 201806
                      AND OC.course_id = CM.course_id) THEN '○'
            ELSE '×' END AS "6 月",
       CASE WHEN EXISTS
                  (SELECT course_id FROM OpenCourses OC
                    WHERE month = 201807
                      AND OC.course_id = CM.course_id) THEN '○'
            ELSE '×' END AS "7 月",
       CASE WHEN EXISTS
                  (SELECT course_id FROM OpenCourses OC
                    WHERE month = 201808
                      AND OC.course_id = CM.course_id) THEN '○'
            ELSE '×' END AS "8 月"
  FROM CourseMaster CM;

  course_name | 6 月 | 7 月 | 8 月
-------------+------+------+------
 経理入門    | ○    | ×    | ×
 財務知識    | ×    | ×    | ○
 簿記検定    | ○    | ×    | ×
 税理士      | ○    | ○    | ○
(4 rows)
```

- どちらのクエリも月数が増えても SELECT 句を修正するだけでよいので拡張性に富むクエリ。

  IN と EXISTS どちらを使っても、結果は変わらない。

  パフォーマンスは EXISTS のほうがよい。

  サブクエリで(month, course_id)という主キーのインデクスが利用できるため、特に OpenCourses テーブルサイズが大きい場合は優位。

## CASE 式の中で集約関数を使う

- 学生と所属クラブを一覧するテーブルを考える。主キーは学生番号、クラブ ID

![スクリーンショット 2020-07-02 0 04 06](https://user-images.githubusercontent.com/51355545/86259907-93404380-bbf7-11ea-9651-8e255ad170aa.png)

```sql
CREATE TABLE StudentClub
(std_id  INTEGER,
 club_id INTEGER,
 club_name VARCHAR(32),
 main_club_flg CHAR(1),
 PRIMARY KEY (std_id, club_id));

INSERT INTO StudentClub VALUES(100, 1, '野球',        'Y');
INSERT INTO StudentClub VALUES(100, 2, '吹奏楽',      'N');
INSERT INTO StudentClub VALUES(200, 2, '吹奏楽',      'N');
INSERT INTO StudentClub VALUES(200, 3, 'バドミントン','Y');
INSERT INTO StudentClub VALUES(200, 4, 'サッカー',    'N');
INSERT INTO StudentClub VALUES(300, 4, 'サッカー',    'N');
INSERT INTO StudentClub VALUES(400, 5, '水泳',        'N');
INSERT INTO StudentClub VALUES(500, 6, '囲碁',        'N');

SELECT * FROM StudentClub;
 std_id | club_id |  club_name   | main_club_flg
--------+---------+--------------+---------------
    100 |       1 | 野球         | Y
    100 |       2 | 吹奏楽       | N
    200 |       2 | 吹奏楽       | N
    200 |       3 | バドミントン | Y
    200 |       4 | サッカー     | N
    300 |       4 | サッカー     | N
    400 |       5 | 水泳         | N
    500 |       6 | 囲碁         | N
(8 rows)
```

- 学生は複数のクラブに所属している場合もある。

  また 1 つにしか所属してない場合もある。

  複数クラブを掛け持ちしている学生は、主なクラブを示す列に Y または N の値が入る。

  1 つだけのクラブに専念している学生の場合は N が入る。

- 次のような条件でクエリを発行する。

  1. １つだけのクラブに所属している学生については、そのクラブ ID を取得する

  1. 複数おクラブを掛け持ちしている学生については、主なクラブの ID を取得する。

  複数のクラブに所属しているか否かは集計結果に対する条件なので HVING 句を使う。

```sql
--条件1：1つのクラブに専念している学生を選択
SELECT std_id, MAX(club_id) AS main_club
  FROM StudentClub
 GROUP BY std_id
HAVING COUNT(*) = 1;

std_id | main_club
--------+-----------
    500 |         6
    400 |         5
    300 |         4
(3 rows)

--条件2：クラブをかけ持ちしている学生を選択
SELECT std_id, club_id AS main_club
  FROM StudentClub
 WHERE main_club_flg = 'Y';

 std_id | main_club
--------+-----------
    100 |         1
    200 |         3
(2 rows)
```

- 上記は CASE 式で 1 つの SQL で書くことができる

```sql
SELECT std_id,
       CASE WHEN COUNT(*) = 1 --1つのクラブに専念する学生の場合
            THEN MAX(club_id)
            ELSE MAX(CASE WHEN main_club_flg = 'Y'
                          THEN club_id
                          ELSE NULL END) END AS main_club
  FROM StudentClub
 GROUP BY std_id;

std_id | main_club
--------+-----------
    500 |         6
    200 |         3
    400 |         5
    300 |         4
    100 |         1
(5 rows)
```

- CASE 式の中に集約関数を書いて、さらにその中に CASE 式を書くという入れ子構造です。

  通常は集約結果に対する条件は HAVING 句を使うが、CASE 式を使うと SELECT 句でも同等の条件分岐がかける。

  HAVING 句で条件分岐させるのは素人のやること、プロは SELECT 句で分岐させる

### CASE 式はどこにでもかける

- SELECT 句

- WHERE 句

- GROUP BY 句

- HAVING 句

- ORDER BY 句

- PARTITION BY 句

- CHECK 制約の中

- 関数の引数

- 述語の引数

- 他の式の中

## CASE 式について詳しくなりたい場合は以下の本がオススメ

1. ジョー・セルコ「プログラマのための SQL 第 4 版」
1. ジョー・セルコ「SQL パズル 第 2 版」

# 2 章演習

## xyz の最大値

```sql
CREATE TABLE Greatests
(key CHAR(1) PRIMARY KEY,
 x   INTEGER NOT NULL,
 y   INTEGER NOT NULL,
 z   INTEGER NOT NULL);

INSERT INTO Greatests VALUES('A', 1, 2, 3);
INSERT INTO Greatests VALUES('B', 5, 5, 2);
INSERT INTO Greatests VALUES('C', 4, 7, 1);
INSERT INTO Greatests VALUES('D', 3, 3, 8);
```

- まずは x と y で比較

```sql
SELECT key,
CASE WHEN x > y THEN x
ELSE y
END AS greatest
FROM Greatests;

 key | greatest
-----+----------
 A   |        2
 B   |        5
 C   |        7
 D   |        3
(4 rows)
```

- 上記だと z を含めた 3 つ以上の列の比較は難しい。やれなくもないが入れ子になって複雑になる。← 正解だった

```sql
SELECT key,
CASE WHEN x > y THEN (CASE WHEN x > z THEN x ELSE z END )
ELSE (CASE WHEN y > z THEN y ELSE z END) END AS greatest
FROM Greatests;
 key | greatest
-----+----------
 A   |        3
 B   |        5
 C   |        7
 D   |        8
(4 rows)
```

## 合計と再掲を表頭に出力する

```sql
select sex as "性別",
sum(population) as "全国"
from PopTbl2
group by sex;
性別 | 全国
------+------
2 | 845
1 | 855
(2 rows)
```

- 上記まではわかったがここから先が分からなかった。

とりあえず徳島だけ出そうと考えた。

エラーによると"poptbl2.pref_name"は aggregate(集約)機能をつけなければならないと。

今一分からず答えをみたら、SUM()でくくればよかった。

```sql
SELECT sex AS "性別",
SUM(population) AS "全国",
CASE pref_name WHEN '徳島' THEN population ELSE 0 END AS "徳島"
FROM PopTbl2 GROUP BY sex;

ERROR:  column "poptbl2.pref_name" must appear in the GROUP BY clause or be used in an aggregate function
LINE 3: CASE pref_name WHEN '徳島' THEN population ELSE 0 END AS "徳...
```

- 下記が正解

```sql
SELECT sex AS "性別",
SUM(population) AS "全国",
SUM(CASE pref_name WHEN '徳島' THEN population ELSE 0 END) AS "徳島",
SUM(CASE pref_name WHEN '香川' THEN population ELSE 0 END) AS "香川",
SUM(CASE pref_name WHEN '愛媛' THEN population ELSE 0 END) AS "愛媛",
SUM(CASE pref_name WHEN '高知' THEN population ELSE 0 END) AS "高知",
SUM(CASE WHEN pref_name IN ('徳島', '香川', '愛媛', '高知') THEN population ELSE 0 END) AS "四国"
FROM PopTbl2
GROUP BY sex;

性別 | 全国 | 徳島 | 香川 | 愛媛 | 高知 | 四国
------+------+------+------+------+------+------
2 | 845 | 40 | 100 | 50 | 100 | 290
1 | 855 | 60 | 100 | 100 | 100 | 360
(2 rows)
```

## ORDER BY でソート列を作る

```sql
SELECT key FROM Greatests
ORDER BY key;
 key
-----
 A
 B
 C
 D
(4 rows)
```

- 上記を B A C D の順番に並び変えるには？

```sql
SELECT key
  FROM Greatests
 ORDER BY CASE key
          WHEN 'B' THEN 1
          WHEN 'A' THEN 2
          WHEN 'D' THEN 3
          WHEN 'C' THEN 4
          ELSE NULL END;

key
-----
 B
 A
 D
 C
(4 rows)
```

- なるほど、ORDER BY 句は数字が割り当てられているということですね。

なので下記のような sort_col が hidden で与えられているということですかね?

```sql
SELECT key,
CASE key
WHEN 'B' THEN 1
WHEN 'A' THEN 2
WHEN 'D' THEN 3
WHEN 'C' THEN 4
ELSE NULL END AS sort_col
FROM Greatests
ORDER BY sort_col;

key | sort_col
-----+----------
 B   |        1
 A   |        2
 D   |        3
 C   |        4
(4 rows)
```
