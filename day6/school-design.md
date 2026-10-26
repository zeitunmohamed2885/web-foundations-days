# School Database Design

## 1. Database Overview

The school database manages students, courses, and enrolments. It stores student details, course information, and the grades students receive in their courses.

The database contains three tables: `students`, `courses`, and `enrolments`.

## 2. Table Design

### Students

The `students` table stores information about each student.

- `student_id`: Primary key that uniquely identifies each student.
- `student_name`: Student's full name; required.
- `email`: Student's email address; required and unique.

The unique constraint on `email` prevents two students from registering with the same email address.

### Courses

The `courses` table stores information about available courses.

- `course_id`: Primary key that uniquely identifies each course.
- `course_name`: Course name; required and unique.

The unique constraint on `course_name` prevents duplicate course names.

### Enrolments

The `enrolments` table records the courses students take and their grades.

- `enrolment_id`: Primary key that uniquely identifies each enrolment.
- `student_id`: Foreign key referencing `students(student_id)`.
- `course_id`: Foreign key referencing `courses(course_id)`.
- `grade`: Stores the student's grade and may remain empty until a grade is assigned.

The combination of `student_id` and `course_id` is unique. This prevents the same student from being enrolled in the same course more than once.

## 3. Relationships Between Tables

### One-to-Many Relationship

One student can have many enrolment records, but each enrolment belongs to one student. This is a one-to-many relationship between `students` and `enrolments`.

Similarly, one course can have many enrolment records, but each enrolment refers to one course. This is a one-to-many relationship between `courses` and `enrolments`.

### Many-to-Many Relationship

Students and courses have a many-to-many relationship. A student can enrol in several courses, and each course can have several students.

The `enrolments` table acts as a junction table between `students` and `courses`. It resolves the many-to-many relationship into two one-to-many relationships and stores additional information, such as grades, for each student-course combination.

## 4. Index Choice

The database uses an index on `enrolments(course_id)` to improve queries that search for enrolments by course, such as listing all students taking a particular course or counting students per course.

The index can help SQLite locate matching enrolment records more efficiently as the database grows.

## 5. SQL vs NoSQL

SQL databases are suitable for this school database because the data is structured and has clear relationships between students, courses, and enrolments. Relational databases support primary keys, foreign keys, unique constraints, and joins, helping maintain data integrity and retrieve related information. A NoSQL database can be useful for flexible or rapidly changing data structures, but a relational SQL database is a natural choice for this system because its data and relationships are well defined.