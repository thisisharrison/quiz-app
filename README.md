# Quiz App

- [Key Operations](#key-operations)
- [List of Entities](#list-of-entities)
- [Required Indexes](#required-indexes)
- [Collection Relationship Diagram](#collection-relationship-diagram)
- [Documents](#document)
- [API Endpoints](#api-endpoints)
- [Database queries](#database-queries)

## Key Operations

Ranked by most frequent read/write

Teachers

1. Assign quiz to student(s)
2. Grade Students answers
3. Create quizzes
4. Add and Remove questions
5. Make changes to quizzes
6. Add and Remove multiple choice options

Students

1. Record answers
2. Read questions from quiz
3. Read assigned quizzes

Assumptions

1. Quizzes can be reused
2. A quiz can only be authored by one teacher
3. Question is unique to a quiz
4. Username field for login

## List of Entities

- users
- quizzes
- questions
- assignments
- submissions

## Required indexes

### User

- `{"username": 1}, {unique: true}`: usernames are unique. We want username
  indexed for reducing query time on Auth process.

### Quiz

- `{"teacher": 1}`: Indexing on `Quiz` for Teachers to quickly query their own
  quizzes.

- `{"quiz": 1, "question": 1}, {unique: true}`: Question in a quiz should be
  unique.

### Assignment

- `{"teacher": 1, "student": 1, "quiz": 1, "submission": 1, "due_date": 1}, {unique: true}`:
  One quiz can only be assigned by a teacher to a student with the exact due
  date once. If quizzes can be retaken or reused, these should have different
  `due_date`.

### Submission

- `{"student": 1, "quiz": 1, "question": 1}, {unique: true}`: There should only
  be one submission by a student per question and per quiz.This allows quick
  read and write to submitted answers for Student and Teacher.

## Collection Relationship Diagram

![ERD-Diagram](https://raw.githubusercontent.com/thisisharrison/quiz-app/main/Quiz-App.png)

### `users`

| column name      | data type | details               |
| :--------------- | :-------: | :-------------------- |
| `id`             |  integer  | required, primary key |
| `fname`          |  string   | required              |
| `lname`          |  string   | required              |
| `email`          |  string   | required              |
| `username`       |  string   | required, indexed     |
| `hashedPassword` |  string   | required              |
| `role`           |  string   | required              |

`role` can be string of `'teacher'` or `'student'`. By making this a string
instead of boolean, we can have accommodate future roles.

### `quizzes`

teachers(users) -1-N- quizzes

quizzes -1-N- questions

quizzes -1-N- submissions

quizzes -1-N- assignments

| column name     | data type | details                           |
| :-------------- | :-------: | :-------------------------------- |
| `id`            |  integer  | required, primary key             |
| `teacher`       |  integer  | required, foreign key, indexed    |
| `title`         |  string   | required                          |
| `questions`     |   array   | required, QuestionSchema, indexed |
| `num_questions` |  integer  | computed, default 0               |

`questions` is an array holding Question references. In the [`Quiz`](#quiz)
document we can include the question's prompt in this field. This is useful if
we want for displaying all the questions to users at a glance.

### `questions`

| column name | data type | details                        |
| :---------- | :-------: | :----------------------------- |
| `id`        |  integer  | required, primary key          |
| `quiz`      |  integer  | required, foreign key, indexed |
| `prompt`    |  string   | required                       |
| `type`      |  string   | required                       |
| `options`   |   array   | not required, default null     |
| `solution`  |  integer  | not required, default null     |
| `weight`    |  integer  | required, default 1            |

`weight` is for Teacher to assign weighting of a question. It is default to `1`
but this design allows for higher points questions.

`type` can be `'multiple-choice'` or `'short-answer'`.

### `assignments`

users(student and teacher) -1-N- assignments

quizzes -1-N- assignments

assignments -1-1- submission

| column name     | data type | details                        |
| :-------------- | :-------: | :----------------------------- |
| `id`            |  integer  | required, primary key          |
| `teacher`       |  integer  | required, foreign key, indexed |
| `student`       |  integer  | required, foreign key, indexed |
| `quiz`          |  integer  | required, foreign key, indexed |
| `submission`    |  integer  | foreign key, indexed           |
| `due_date`      |  integer  | required, indexed              |
| `submit_status` |  boolean  | required, default false        |
| `grade_status`  |  string   | required, default 'pending'    |
| `tot_score`     |  integer  | required, default 0            |

While `due_date` can be `Date` type, MongoDB stores time in UTC by default. We
can convert UTC to users' local timezones. However, we can utilize unix
timestamps and package like [momentjs](https://momentjs.com/) to calculate date
and render date logic more simply.

To check if due date has passed, we can simply check if current unix timestamp
is greater than due date timestamp. This simplifies the logic especially if
Teacher assigns quizzes to Student across different timezones.

`submit_status`, `grade_status`, and `tot_score` are from
[`submissions`](#submissions). We make copy of these fields in `assignments` as
they are not frequent write fields - only once per quiz submission. This
benefits the Student to quickly see all their submission statuses and grades. It
also benefits the Teacher when he/she wants to calculate class average. We can
count the number of rows with `quiz` equals `quiz_id`, and sum the `tot_score`
to get the average.

This model assumes quizzes can be reused. Another design could be storing
`tot_score` and `num_submissions` in [`quizzes`]. Every submission increments
`num_submissions`. When Teacher finishes grading, `tot_score` from `submissions`
are added to `tot_score` in `quizzes`. We can then calculate the average with
these two fields.

### `submissions`

students(users) -1-N- submissions

quizzes -1-N- submissions

| column name     | data type | details                              |
| :-------------- | :-------: | :----------------------------------- |
| `id`            |  integer  | required, primary key                |
| `student`       |  integer  | required, foreign key, indexed       |
| `quiz`          |  string   | required, foreign key, indexed       |
| `answers`       |   array   | QuestionSchema, foreign key, indexed |
| `submit_status` |  boolean  | required, default false              |
| `grade_status`  |  string   | required, default 'pending'          |
| `tot_score`     |  integer  | computed, default 0                  |

`answers` is an array holding Question references. Detail explanation in the
document of [`Submission`](#submission).

## Document

### Quiz

We have an array that holds extended reference to questions. This is structured
in an array to maintain order of the questions. If Teacher chooses to randomize
the questions, this model is works too.

Including the prompt allows Students and Teachers to view all questions at a
glance. I assumed prompts rarely change as well. However, if we do not need this
feature, we can change this to an array of `ObjectId`. We could use `$lookup`,
`populate`, or `virtual` to return more information.

```js
{
  _id: <ObjectId>,
  teacher: <ObjectId>,
  title: 'Fun quiz',
  questions: [
    {
      question: <ObjectId>,
      prompt: 'What is a prime number'
    }
  ],
  num_questions: 2,
}
```

### Question

For multiple choice questions, we store options as an array. The front end will
take care of randomizing options and labelling them 'A', 'B', 'C' and so on. The
solution is the index of the correct option.

In the first example, options are `['3','6','8','1']`. The solution is `0`,
which points to `'3'`. The front end can randomize and display this:

- A: 6
- B: 3
- C: 1
- D: 8

In the front end, we can use `data-*` to store the indexes of these answer. When
Student submits 'B' as his/her answer ([Submission](#submission)), he/she will
submit the string `'0'`. We then parse string to integer and check if submitted
index matches the solution's.

```js
// Multiple choice with one correct answer
{
  _id: <ObjectId>,
  quiz: <ObjectId>,
  prompt: 'What is a prime number?',
  type: 'multiple-choice',
  options: ['3','6','8','1'],
  solution: 0,
  weight: 1
}
// Short answer question
{
  _id: <ObjectId>,
  quiz: <ObjectId>,
  prompt: 'What is a hash function?',
  type: 'short-answer',
  options: null,
  solution: null,
  weight: 1,
}
```

### Assignment

```js
{
  _id: <ObjectId>,
  teacher: <ObjectId>,
  student: <ObjectId>,
  quiz: <ObjectId>,
  submission: <ObjectId>,
  due_date: 1622029501,
  submit_status: true,
  grade_status: 'complete',
  tot_score: 1
}
```

### Submission

```js
{
  _id: <ObjectId>,
  student: <ObjectId>,
  quiz: <ObjectId>,
  answers: [
    // submission to a multiple choice with one correct answer
    {
      question: <ObjectId>,
      solution: "1",
      correct: false,
      score: 0
    },
    // submission to a short answer question
    {
      question: <ObjectId>,
      solution: "Hash function returns a value used to index into a hash table",
      correct: true,
      score: 1
    }
  ],
  submit_status: true,
  grade_status: 'complete',
  tot_score: 1,
}
```

## API Endpoints

### `quizzes`

- `GET /api/quizzes` - returns quizzes authored by teacher
- `GET /api/quizzes/:id` - returns single quiz
- `POST /api/quizzes` - creates a quiz
- `PATCH /api/quizzes/:id` - edits a quiz
- `DELETE /api/quizzes/:id` - removes a quiz

### `questions`

- `GET /api/questions/:id` - returns a single question
- `GET /api/quizzes/:quizId/questions` - returns all questions of a quiz
- `POST /api/quizzes/:quizId/questions` - creates a question in a quiz
- `PATCH /api/questions/:id` - edits a question
- `DELETE /api/questions/:id` - removes a question

### `assignments`

- `GET /api/assignments/` - returns assigned quizzes to user
- `POST /api/assignments/` - assigns students from request body a quiz
- `PATCH /api/assignments/:id` - allows Teacher to edit assignments' due date
  (e.g. an extension is given)
- `DELETE /api/assignments/:id` - allows Teacher to delete assignments
- `GET /api/assignments/quizzes/:quizId` - returns summary of quiz (eg. average
  grade) to Teacher

### `submissions`

- `GET /api/submissions/:id/` - reads a student's submission
- `POST /api/submissions/:id/questions/:questionId` - Student submits an answer,
  Teacher grades a question
- `PATCH /api/submissions/:id/questions/:questionId` - edits an answer, edits a
  grading
- `DELETE /api/submissions/:id/questions/:questionId` - removes an answer,
  removes a grade

### `users`

- `GET /api/register` - creates an account
  - generates a hash password from the provided password using
    [bcrypt](https://github.com/kelektiv/node.bcrypt.js)
- `POST /api/login` - redirects to homepage
  - finds user with the username
  - uses `bcrypt.compare` the password string with `user.hashedPassword`
  1. We are using JWT
     - Create a payload and sign the payload into a jwt string payload using a
       secret key
     - Store token in header for future requests
  2. We are using Session Token
     - Generate 16 byte random hex string and save to `user.token` field
     - Store token in header for future requests
- `DELETE /api/signout` - signs out a user
  1. We are using JWT
     - We don't need this API endpoint
     - In the front end, we will delete our Bearer token from our
       'Authorization' header
  2. We are using Session Token
     - Reset `user.token` to new hex string
     - In the front end, we will delete our Bearer token from our
       'Authorization' header
