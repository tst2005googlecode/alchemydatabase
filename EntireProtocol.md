## SQL EXAMPLE ##
Bash code:
```
  CLI="./redisql-server"
  $CLI CREATE TABLE foo "(id INT, val FLOAT, name TEXT)"
  $CLI CREATE INDEX foo_val_index ON foo \(val\)
  $CLI INSERT INTO foo VALUES \(1,1.1111111,one\)
  $CLI INSERT INTO foo VALUES \(2,2.2222222,two\)
  $CLI SELECT "*" FROM foo WHERE "id = 1"
  $CLI SCAN "*" FROM foo 
  $CLI UPDATE foo SET val=9.999999 WHERE "id = 1"
  $CLI DELETE FROM foo WHERE "id = 2"
  $CLI DROP INDEX foo_val_index
  $CLI DESC foo
  $CLI DUMP foo
```

Returns:
```
OK
OK
OK
OK
1. "1,1.111111045,one"
1. "1,1.111111045,one"
2. "2,2.22222209,two"
(integer) 1
(integer) 1
(integer) 1
1. "id | INT | INDEX: foo:id:index [BYTES: 0]"
2. "val | FLOAT "
3. "name | TEXT "
4. "INFO: KEYS: [NUM: 1 MIN: 1 MAX: 1] BYTES: [BT-DATA: 13 BT-TOTAL: 4158 INDEX: 0]"
1. "1,9.999999046,one"
```

Using tcpdump and sifting thru the results
```
sudo tcpdump -t -l -q -A -s 1500 -i lo 'tcp port 6379' 
```

Line Protocol Request then Response:
```
*4
$6
CREATE
$5
TABLE
$3
foo
$30
(id INT, val FLOAT, name TEXT)

+OK

*6
$6
CREATE
$5
INDEX
$13
foo_val_index
$2
ON
$3
foo
$5
(val)

+OK

*5
$6
INSERT
$4
INTO
$3
foo
$6
VALUES
$17
(1,1.1111111,one)

+OK

*5
$6
INSERT
$4
INTO
$3
foo
$6
VALUES
$17
(2,2.2222222,two)

+OK

$6
SELECT
$1
*
$4
FROM
$3
foo
$5
WHERE
$6
id = 1

*1
$17
1,1.111111045,one

*4
$4
SCAN
$1
*
$4
FROM
$3
foo

*2
$17
1,1.111111045,one
$16
2,2.22222209,two

*6
$6
UPDATE
$3
foo
$3
SET
$12
val=9.999999
$5
WHERE
$6
id = 1

:1

*5
$6
DELETE
$4
FROM
$3
foo
$5
WHERE
$6
id = 2

1

*3
$4
DROP
$5
INDEX
$13
foo_val_index

:1

*2
$4
DESC
$3
foo

*4
$41
id | INT | INDEX: foo:id:index [BYTES: 0]
$12
val | FLOAT 
$12
name | TEXT 
$79
INFO: KEYS: [NUM: 1 MIN: 1 MAX: 1] BYTES: [BT-DATA: 13 BT-TOTAL: 4158 INDEX: 0]

*2
$4
DUMP
$3
foo

*1
$17
1,9.999999046,one
```