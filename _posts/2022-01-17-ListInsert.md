---
title: "insert의 다중처리"
excerpt: "database의 트랜잭션 횟수를 줄여보자!"
categories: project
tags: project
last-modified-at: {{ page.last_modified_at }}
---

openApi에서 썼던 list값으로 여러객체를 insert를 통해 db에 넣는 방식을 다시 한번 써야했다.  
저번의 방식은 seq.nextval을 사용할 경우 단일 값으로 처리되어 일일이 insert를 했으나
이벤에는 seq.nextval로 primary key값을 설정하고 list값을 insert하는 방법을 찾아보았다.
```xml
<insert id="insertRoom" parameterType="java.util.List">
    insert into trip_train_room(room_no, room_name, room_type, root_seat_count, train_no)
    select train_room_seq.nextval, A.* from (
    <foreach collection="list" item="room" separator="UNION ALL ">
        select #{room.name}, #{room.type}, #{room.seatCount}, #{room.trainNo}
        from dual  
    </foreach> ) A
</insert>

```
하나씩 뜯어보면서 공부해 보자.  
insert의 값을 넣는 방법은
```sql
    insert into table1 (칼럼1, 칼럼2, 칼럼3, ...)
    values
    (시퀀스이름.nextval, 값, 값, ...)
```
이 기본적으로 배운 방법이다. 하지만 이 방법은 하나의 data만 입력하는 방법이다. 
  
여러 행을 insert하는 방법  
```sql
    insert into table1 (칼럼1, 칼럼2, 칼럼3, ...)
    select * from table2
    -- 값이 있는 table2  
```
이 방법은 서브 쿼리를 사용하여 여러 데이터를 가져와서 데이터를 한꺼번에 입력하는 방법이다.
이를 **ITAS**라고 한다.  
table1, 2의 칼럼개수와 데이터 형이 동일해야 한다.  
<br/>
<br/>
<br/>
* * *

```sql
    insert into trip_train_room(room_no, room_name, room_type, root_seat_count, train_no)
    select train_room_seq.nextval, A.* from (select #{room.name}, #{room.type}, #{room.seatCount}, #{room.trainNo}
                                            from dual ) A
    -- 와 같은 방식이다.  여기서 UNION ALL이 무엇 일까?
```
+ 인라인 뷰 방식: parameter값(db에 넣을 값)을 조회된 결과로 **가상의 테이블**을 만든다.  
+ 서브쿼리로 sequence와 data를 가져와서 한꺼번에 data를 입력할 것 이다.  
<br/>
아마도  

| room.name | room.type | room.seatCount | room.trainNo |
|---|---|---|---|
| 1호차 | 일반실 | 30 | 1 |

이러한 값을 가진 객체한개가 insert될 것이다.
<br/>
<br/>   
```xml
    insert into trip_train_room(room_no, room_name, room_type, root_seat_count, train_no)
    select train_room_seq.nextval, A.* from (
    <foreach collection="list" item="room" separator="UNION ALL ">
        select #{room.name}, #{room.type}, #{room.seatCount}, #{room.trainNo}
    </foreach> ) A
```
+ foreach로 list에 있는 모든 객체를 반복하면서 행이 조회되고
+ UNION ALL를 통해 모든 조회된 모든 행이 연결되면서 하나의 가상테이블이 만들어질 것이다.  
+ foreach인 아닌 외부에서 seq을 넣으므로 seq도 정상적으로 값을 주고 모든 행의 값이 insert되는 것이다.  

* * *
<br/>
<br/>
<br/>
extra **INSERT ALL**   
하나의 insert문으로 여러개의 테이블에 데이터를 입력할 수 있다.
```sql
    insert all
        into table1 values (칼럼1, 칼럼2, 칼럼3)
        into table2 (칼럼1, 칼럼5, 칼럼4) VALUES (칼럼1, 칼럼5, 칼럼4)
    select 칼럼1, 칼럼2, 칼럼3, 칼럼4, 칼럼5
    FROM table3
    where 조건문;
```
또한 when then절을 만족하는 서브쿼리의 결과행을 각각 테이블에 입력한다.
```sql
    INSERT ALL
        WHEN 조건 THEN
            INTO table1 
            VALUES(칼럼1, 칼럼2, 칼럼3)
        WHEN 칼럼0=n THEN
            INTO table2 
            VALUES(칼럼1, 칼럼2, 칼럼3)
        SELECT 칼럼0 칼럼1, 칼럼2, 칼럼3
        FROM table3;
```
[출처](http://www.gurubee.net/lecture/2688)

