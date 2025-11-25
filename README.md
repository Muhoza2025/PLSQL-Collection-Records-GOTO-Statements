# PLSQL-Collection-Records-GOTO-Statements
# Student Book Borrowing System (PL/SQL)

This project implements a library management logic using Oracle PL/SQL. It validates borrowing requests based on student quotas and specific genre restrictions stored in nested tables.

## Project Description

The system manages student records and enforces two primary rules before a book can be borrowed:
1.  *Quota Management:* Prevents students from borrowing more than a specified limit (default is 3 books).
2.  *Content Filtering:* Uses a Nested Table collection to store specific restricted genres for each student (e.g., restricting "Horror" or "Fantasy").

## Database Objects

* **T_GENRES**: A custom collection type (TABLE OF VARCHAR2) for handling multiple restrictions per student.
* **STUDENTS**: The main table storing student demographics, current book counts, and the nested table of restrictions.
* **CHECK_BORROWING**: A stored procedure that accepts a Student ID and Book Genre, runs validation logic, and outputs the result.

## Source Code

Copy the script below to create the schema, insert data, and test the procedure in Oracle SQL Developer.

```sql
set serveroutput on;

-------------------------------------------
-- 1. CLEANUP (REMOVED DROP STATEMENTS)
-------------------------------------------
-- No cleanup needed since DROP statements were removed


-------------------------------------------
-- 2. CREATE TYPE: Collection of restricted genres
-------------------------------------------
create or replace type t_genres as table of varchar2(40);
/


-------------------------------------------
-- 3. CREATE TABLE: Students with restricted reading genres
-------------------------------------------
create table students (
    student_id number primary key,
    student_name varchar2(50),
    age number,
    borrowed_books number,
    restricted_genres t_genres
) nested table restricted_genres store as nt_genres;
/


-------------------------------------------
-- 4. INSERT DATA (ALL NEW NAMES)
-------------------------------------------
insert into students 
values (1, 'Aline', 12, 1, t_genres());

insert into students 
values (2, 'Patrick', 10, 3, t_genres('horror'));

insert into students 
values (3, 'Sandra', 16, 0, t_genres('romance','fantasy'));

insert into students 
values (4, 'Lewis', 8, 2, t_genres());

insert into students 
values (5, 'Bella', 11, 4, t_genres('mystery'));
/


-------------------------------------------
-- 5. PROCEDURE: Check book borrowing eligibility
-------------------------------------------
create or replace procedure check_borrowing(
    p_student_id in number,
    p_book_genre in varchar2,
    p_max_books in number default 3
) is
    v_student students%rowtype;
    v_blocked boolean := false;
begin
    -- Fetch student
    select * into v_student
    from students
    where student_id = p_student_id;

    -- CONDITION 1: Borrow limit
    if v_student.borrowed_books >= p_max_books then
        dbms_output.put_line('Borrowing Denied: ' || v_student.student_name ||
                             ' has reached the maximum number of borrowed books.');
        return;
    end if;

    -- CONDITION 2: Restricted genres check
    if v_student.restricted_genres is not null 
       and v_student.restricted_genres.count > 0 then
        
        for i in 1..v_student.restricted_genres.count loop
            if lower(v_student.restricted_genres(i)) = lower(p_book_genre) then
                v_blocked := true;
                exit;
            end if;
        end loop;
    end if;

    if v_blocked then
        dbms_output.put_line('Borrowing Denied: ' || v_student.student_name ||
                             ' is restricted from reading ' || p_book_genre || ' books.');
        return;
    end if;

    -- SUCCESS
    dbms_output.put_line('Borrowing Approved: ' || v_student.student_name ||
                         ' may borrow a ' || p_book_genre || ' book.');

exception
    when no_data_found then
        dbms_output.put_line('Error: Student with ID ' || p_student_id || ' does not exist.');
    when others then
        dbms_output.put_line('Error: ' || sqlerrm);
end check_borrowing;
/


-------------------------------------------
-- 6. TEST THE PROCEDURE (UPDATED NAMES)
-------------------------------------------
begin
    dbms_output.put_line('--- TEST 1: Aline (Allowed) ---');
    check_borrowing(1, 'mystery');

    dbms_output.put_line(chr(10) || '--- TEST 2: Patrick (Restricted Genre: Horror) ---');
    check_borrowing(2, 'horror');

    dbms_output.put_line(chr(10) || '--- TEST 3: Sandra (Restricted: Fantasy) ---');
    check_borrowing(3, 'fantasy');

    dbms_output.put_line(chr(10) || '--- TEST 4: Lewis (Allowed) ---');
    check_borrowing(4, 'science');

    dbms_output.put_line(chr(10) || '--- TEST 5: Bella (Borrow Limit Exceeded) ---');
    check_borrowing(5, 'romance');
end;
--- TEST 1: Aline (Allowed) ---
Borrowing Approved: Aline may borrow a mystery book.

--- TEST 2: Patrick (Restricted Genre: Horror) ---
Borrowing Denied: Patrick has reached the maximum number of borrowed books.

--- TEST 3: Sandra (Restricted: Fantasy) ---
Borrowing Denied: Sandra is restricted from reading fantasy books.

--- TEST 4: Lewis (Allowed) ---
Borrowing Approved: Lewis may borrow a science book.

--- TEST 5: Bella (Borrow Limit Exceeded) ---
Borrowing Denied: Bella has reached the maximum number of borrowed books.
