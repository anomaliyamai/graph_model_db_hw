# Домашняя работа по графовым базам данных

### Обсуждение предметной области

В данной работе рассматривается расписание занятий студентов и преподавателей.

### Построение ER-диаграммы

На картинке ниже отображено следующее:

1) Сущности с атрибутами:

    a. **Student** - студент с атрибутами вида:  
        - **ID** - уникальный идентификатор студента.  
        - **Surname** - фамилия студента.  
        - **Name** - имя студента.  
        - **Patronymic** - отчество студента.  
    b. **Group** - студенческая группа с атрибутами вида:  
        - **ID** - уникальный идентификатор группы.  
        - **Number** - номер группы.  
    с. **Class** - занятие с атрибутами вида:  
        - **ID** - уникальный идентификатор занятия.  
        - **Classroom** - аудитория.  
        - **Beginning** - начало занятия.  
        - **Duration** - продолжительность занятия.  
    d. **Subject** - предмет с атрибутами вида:  
        - **ID** - уникальный идентификатор предмета.  
        - **Name** - название предмета.  
    e. **Teacher** - преподаватель с атрибутами вида:  
        - **ID** - уникальный идентификатор преподавателя.  
        - **Surname** - фамилия преподавателя.  
        - **Name** - имя преподавателя.  
        - **Patronymic** - отчество преподавателя.

2) Отношения:  
    a. **Member** - каждый студент является членом какой-то из групп, группа состоит из конкретных студентов.  
    b. **Held** - занятие проводится для какой-то группы, конкретная группа посещает занятие.  
    c. **Taught** - на занятии преподается какой-то предмет, конкретный предмет преподается на конкретном занятии.  
    d. **Conducted** - занятие проводится каким-то преподавателем, конкретный преподаватель проводит занятие.


![alt text](ertikz.jpg "ER-diagram created in Tikz")


### Построение диаграммы отношений


![alt](erd.jpg "Entity Relationship Diagram")


### Создание таблиц средствами SQL и наполнение данных

Создадим все таблицы (работаю с СУБД PostgreSQL версии 16):  

`CREATE TABLE studentgroup (id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL);`

`CREATE TABLE subject (id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL);`

`CREATE TABLE teacher (surname VARCHAR(255) NOT NULL, name VARCHAR(255) NOT NULL, patronymic VARCHAR(255), id SERIAL PRIMARY KEY);`

`CREATE TABLE student (surname VARCHAR(255) NOT NULL, name VARCHAR(255) NOT NULL, patronymic VARCHAR(255), studentgroupid INTEGER REFERENCES studentgroup(id), id SERIAL PRIMARY KEY);`

`CREATE TABLE class (studentgroupid INTEGER REFERENCES studentgroup(id), duration INTERVAL NOT NULL, beginning TIMESTAMP NOT NULL, teacherid INTEGER REFERENCES teacher(id), subjectid INTEGER REFERENCES subject(id), classroom VARCHAR(255), capacity INTEGER, id SERIAL PRIMARY KEY);`

Наполним их данными:

`insert into studentgroup (name) values ('2110'), ('216'), ('213');`

`insert into student (surname, name, patronymic, studentgroupid) values ('Afanasev', 'Oleg', 'Dmitrievich', (SELECT id from studentgroup WHERE name='2110')), ('Ivanov', 'Pavel', 'Alekseevich', (SELECT id from studentgroup WHERE name='216')), ('Konstantin', 'Vedernikov', NULL, (SELECT id from studentgroup WHERE name='216')), ('Zabelin', 'Maksim', 'Maksimovich', (SELECT id from studentgroup WHERE name='213')), ('Nuzhdin', 'Anton', 'Olegovich', (SELECT id from studentgroup WHERE name='213')), ('Ryabov', 'Eduard', NULL, (SELECT id from studentgroup WHERE name='2110'));`

`insert into teacher (surname, name, patronymic) values ('Ivanov', 'Andrey', 'Alexandrovich'), ('Lipovskiy', 'Roman', NULL), ('Sokolov', 'Evgeniy', 'Andreevich');`

`insert into subject (name) values ('Machine Learning'), ('Concurrency'), ('Industrial Development');`

`insert into class (studentgroupid, duration, beginning, teacherid, subjectid, classroom, capacity) values ((SELECT id from studentgroup WHERE name='2110'), '01:20:00', '2024-10-10 11:30:30', (select id from teacher where surname = 'Ivanov'), (select id from subject where name = 'Industrial Development'), 'R504', 30), ((SELECT id from studentgroup WHERE name='2110'), '01:20:00', '2024-10-10 11:35:30', (select id from teacher where surname = 'Lipovskiy'), (select id from subject where name = 'Concurrency'), 'R504', 30);`

Теперь напишем запросы, отвечающие на вопросы предметной области к реляционной
БД:

1. Получить среднее количество студентов в группе;  
   `SELECT studentgroup.name, COUNT(student.id) / COUNT(DISTINCT studentgroup.id) AS avg_student_count FROM student JOIN studentgroup ON student.studentgroupid = studentgroup.id GROUP BY studentgroup.name;`  

2. Получить среднее количество предметов на студента и
преподавателя;  
   `SELECT student.studentgroupid, AVG(( SELECT COUNT(DISTINCT subjectid) FROM class WHERE studentgroupid = student.studentgroupid )) AS avg_subject_count FROM student GROUP BY student.studentgroupid;` (на студента)  
   `SELECT teacherid, AVG(subject_count) AS avg_subject_count FROM ( SELECT teacherid, COUNT(DISTINCT subjectid) AS subject_count FROM class GROUP BY teacherid, subjectid ) AS subject_counts GROUP BY teacherid;` (на преподавателя)  
3. Получить конфликты в расписании по конкретной группе, конкретной
группе и предмету;  
    `select * from class where (beginning < (select beginning from class where teacherid = 1 AND studentgroupid = 1 AND subjectid = 3) + (select duration from class where teacherid = 1 AND studentgroupid = 1 AND subjectid = 3) AND beginning > (select beginning from class where teacherid = 1 AND studentgroupid = 1 AND subjectid = 3)) OR (beginning < (select beginning from class where teacherid = 1 AND studentgroupid = 1 AND subjectid = 3) AND beginning + duration > (select beginning from class where teacherid = 1 AND studentgroupid = 1 AND subjectid = 3));`
