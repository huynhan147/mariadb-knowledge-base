[Nguồn](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Compound (Composite) Indexes - MariaDB Knowledge Base
# Chỉ mục hỗn hợp (Composite Index) - Kiến thức MariaDB cơ bản

## A mini-lesson in "compound indexes" ("composite indexes")
## Một bài học nhỏ trong "Chỉ mục hỗn hợp" ("composite Index")

Tài liệu này bắt đầu có vẻ tầm thường và nhàm chán, nhưng xây dựng lên nhiều thông tin thú vị hơn, có lẽ bạn không nhận ra về cách mà chỉ mục (index) của MariaDB và MySQL hoạt động.

Nó cũng giải thích [EXPLAIN][1] ( đến một mức độ nào đó).

( Hầu hết điều này cũng áp dụng cho các loại cơ sở dữ liệu không phải MySQL)

## Truy vấn để thảo luận

Câu hỏi là "Andrew Johnson đã trở thành tổng thống Mỹ khi nào?".

Bảng `Presidents` có sẵn trông giống như:
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" đã được chọn cho bài học này bởi vì những sự trùng lặp)

index(nhiều index) nào là tốt nhất cho câu hỏi đó? Cụ thể hơn, cái nào là tốt nhất cho
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

Một vài INDEX để thử...

* Không index
* INDEX(first_name), INDEX(last_name) (hai index riêng biệt) 
* "Index Merge Intersect"
* INDEX(last_name, first_name) (một index "hỗn hợp") 
* INDEX(last_name, first_name, term) (một index bao hàm) 
* Biến thể 

## Không index

Tốt thôi, Tôi đang vớ vẩn một chút ở đây. Tôi có một KHÓA CHÍNH (PRIMARY KEY) tại `seq`, nhưng nó không có lợi ích trong truy vấn mà chúng ta đang học.
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Or, using the other form of display:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Ngụ ý là scan bảng
    possible_keys: NULL
              key: NULL       <-- Ngụ ý là không có index nào hữu ích, do đó scan bảng
          key_len: NULL
              ref: NULL
             rows: 44         <-- Điều này là có bao nhiêu hàng trong bảng, vậy scan bảng
            Extra: Using where
    

## Chi tiết triển khai

Đầu tiên, hãy giới thiệu cách InnoDB lưu trữ và sử dụng index.

* Dữ liệu và KHÓA CHÍNH (PRIMARY KEY) nhóm lại cùng nhau trên BTree.
* Tra cứu BTree khá nhanh và hiệu quả. Cho một bảng với hàng triệu hàng có lẽ có 3 cấp độ của BTree, và hai level cao nhất được lưu vào cache.
* Mỗi index thứ cấp trong một BTree khác, với KHÓA CHÍNH ở lá.
* Việt lấy liên tục ( theo index ) các phần tử từ một BTree là vô cùng hiệu quả bởi vì chúng được lưu trữ liên tục.
* Với lợi ích đơn giản, chúng ta có thể đếm mỗi lần tra cứu BTree như 1 đơn vị công việc, và loại bỏ scan phần tử liên tục. Nó xấp xỉ con số truy cập của ổ đĩa cho một bảng lớn trong một hệ thống bận.

Với MyISAM, KHÓA CHÍNH không được lưu trữ với dữ liệu, vậy suy nghĩ nó giống như một khóa thứ cấp ( quá đơn giản ).

## INDEX(first_name), INDEX(last_name)

Người mới, mỗi lần anh ấy học về việc đánh index, quyết định để lập index của nhiều cột, một cái một lần. Nhưng...

MySQL hiếm khi sử dụng nhiều hơn một index trong một lần trong một truy vấn. Vậy nó sẽ phân tích những index có thể.

* first_name -- có hai hàng có thể (một tra cứu BTree, sau đó scan liên tục)
* last_name -- có hai hàng có thể. Giả sử nó chọn last_name. Đây là những bước cho việc SELECT:
1. Sử dụng INDEX(last_name), tìm 2 index với last_name = 'Johnson'.
2. Lấy KHÓA CHÍNH (đã ngầm thêm vào mỗi index thứ cấp trong )InnoDB; lấy (17, 36). 
3. Tiếp cận dữ liệu sử dụng seq = (17, 36) để lấy những hàng cho Andrew Johnson và Lyndon B. Johnson. 
4. Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả những trừ hàng mong muốn.
5. Cung cấp câu trả lời (1865-1869). 
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 cần 2+3*30 bytes
              ref: const
             rows: 2                  <-- Hai 'Johnson's
            Extra: Using where

## "Index Merge Intersect" 

OK, vậy bạn trở thành cực kỳ thông minh và quyết định rằng MySQL nên đủ thông minh để sử dụng tên index giống nhau để có câu trả lời. Điều này được gọi là "Intersect".
1. Sử dụng INDEX(last_name), tìm 2 index với last_name = 'Johnson'; nhận được (7, 17) 
2. Sử dụng INDEX(first_name), tìm 2 index với first_name = 'Andrew'; nhận được (17, 36) 
3. "And" hai danh sách cùng nhau (7,17) & (17,36) = (17) 
4. Tiếp cận dữ liệu sử dụng seq = (17) để có được hàng cho Andrew Johnson. 
5. Cung cấp câu trả lời (1865-1869). 
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    

Mệnh đề EXPLAIN lỗi để cho ra thông tin chi tiết của bao nhiêu hàng được thu thập từ mỗi index, vân vân.

## INDEX(last_name, first_name)

Đó họ là "hỗn hợp" hoặc "composite" index khi nó có nhiều hơn một cột.
1. Đi sâu vào BTree để đánh chỉ mục có được chính xác index của hàng cho Johnson+Andrew; có được seq = (17). 
2. Tiếp cận dữ liệu sử dụng seq = (17) để có được hàng cho Andrew Johnson. 
3. Cung cấp câu trả lời (1865-1869). Nó tốt hơn nhiều. Trong thực tế nó được gọi là "best".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- The length of both fields
              ref: const,const     <-- The WHERE clause gave constants for both
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "Covering": INDEX(last_name, first_name, term)

Surprise! We can actually do a little better. A "Covering" index is one in which _all_ of the fields of the SELECT are found in the index. It has the added bonus of not having to reach into the "data" to finish the task. 1\. Drill down the BTree for the index to get to exactly the index row for Johnson+Andrew; get seq = (17). 2\. Deliver the answer (1865-1869). The "data" BTree is not touched; this is an improvement over "compound".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Everything is similar to using "compound", except for the addition of "Using index".

## Variants

* What would happen if you shuffled the fields in the WHERE clause? Answer: The order of ANDed things does not matter. 
* What would happen if you shuffled the fields in the INDEX? Answer: It may make a huge difference. More in a minute. 
* What if there are extra fields on the the end? Answer: Minimal harm; possibly a lot of good (eg, 'covering'). 
* Reduncancy? That is, what if you have both of these: INDEX(a), INDEX(a,b)? Answer: Reduncy costs something on INSERTs; it is rarely useful for SELECTs. 
* Prefix? That is, INDEX(last_name(5). first_name(5)) Answer: Don't bother; it rarely helps, and often hurts. (The details are another topic.) 

## More examples:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

## Postlog

Refreshed -- Oct, 2012; more links -- Nov 2016