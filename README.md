# Quiz App

- [Key Operations](#key-operations)
- [List of Entities](#list-of-entities)
- [Required Indexes](#required-indexes)
- [Collection Relationship Diagram](#collection-relationship-diagram)
- [Documents](#document)
- [API Endpoints](#api-endpoints)
- [Database queries](#database-queries)

## <a name="key-operations"></a>Key Operations

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

## <a name="list-of-entities"></a>List of Entities

- users
- quizzes
- questions
- submissions

## <a name="required-indexes"></a>Required indexes

- `{"username": 1}, {unique: true}`: usernames are unique. We want username
  indexed for reducing query time on Auth process.

- `{"teacher": 1}`: Indexing on `Quiz` for Teachers to quickly query their own
  quizzes.

- `{"quiz": 1, "question": 1}, {unique: true}`: Question in a quiz should be
  unique.

- `{"student": 1, "quiz": 1, "question": 1}, {unique: true}`: There should only
  be one submission by a student per question and per quiz.This allows quick
  read and write to submitted answers for Student and Teacher.

## <a name="collection-relationship-diagram"></a>Collection Relationship Diagram

![ERD-Diagram](https://raw.githubusercontent.com/thisisharrison/quiz-app/main/Quiz-App.png)

### `users`

| column name | data type | details               |
|:------------|:---------:|:----------------------|
| `id`        |  integer  | required, primary key |
| `fname`     |  string   | required              |
| `lname`     |  string   | required              |
| `email`     |  string   | required              |
| `username`  |  string   | required, indexed     |
| `password`  |  string   | required              |
| `role`      |  string   | required              |

### `quizzes`

teachers(users) -1-N- quizzes

quizzes -1-N- questions

| column name       | data type | details                        |
|:------------------|:---------:|:-------------------------------|
| `id`              |  integer  | required, primary key          |
| `teacher_id`      |  integer  | required, foreign key, indexed |
| `title`           |  string   | required                       |
| `due_date`        |  integer  | required, unix                 |
| `questions`       |   array   | required, QuestionSchema       |
| `num_questions`   |  integer  | computed, default 0            |
| `num_submissions` |  integer  | computed, default 0            |
| `tot_score`       |  integer  | computed, default 0            |

### `questions`

| column name | data type | details                        |
|:------------|:---------:|:-------------------------------|
| `id`        |  integer  | required, primary key          |
| `quiz_id`   |  integer  | required, foreign key, indexed |
| `prompt`    |  string   | required                       |
| `type`      |  string   | required                       |
| `options`   |   array   | not required, default null     |
| `solution`  |   array   | not required, default null     |
| `weight`    |  integer  | required, default 1            |

### `submissions`

students(users) -1-N- submissions

quizzes -1-N- submissions 

| column name  | data type | details                              |
|:-------------|:---------:|:-------------------------------------|
| `id`         |  integer  | required, primary key                |
| `student_id` |  integer  | required, foreign key, indexed       |
| `quiz_id`    |  string   | required, indexed                    |
| `answers`    |   array   | QuestionSchema, foreign key, indexed |
| `tot_score`  |  integer  | computed                             |

## <a name="document"></a>Document

### Quiz

We have an array that holds extended reference to questions. This is structured in an array to maintain order of the questions. If Teacher chooses to randomize the questions, this model is works too. 

Including the prompt allows Students and Teachers to view all questions at a glance. I assumed prompts rarely change as well. However, if we do not need this feature, we can change this to an array of `ObjectId`. We could use `$lookup`, `populate`, or `virtual` to return more information. 

```js
{
  _id: <ObjectId>,
  teacher: <ObjectId>,
  title: 'Fun quiz',
  due_date: 1622120400,
  questions: [
    {
      question: <ObjectId>,
      prompt: '...'
    }
  ],
  num_questions: 10,
  num_submissions: 15,
  tot_score: 150
}
```

### Question

For multiple choice questions, we can store options as an array. The front end will take care of randomizing options and labelling them 'A', 'B', 'C' and so on. 

We also want to allow questions with multiple correct answers. So the solution is an array. This array stores the index of the correct answer. 

In the first example, options are `['3','6','8','1']`. The solution is `[0]`, which points to `'3'`. The front end can randomize and display this: 

  - A: 6
  - B: 3
  - C: 1
  - D: 8

In the front end, we can use `data-*` to store the indexes of these answer. When Student submits 'B' as his/her answer ([Submission](#submission)), he/she will submit the string `'0'` (or `'0, 1'` in the case of multiple answers). We then split the string, parse to integer and check if submitted index(es) matches the solution's.

```js
// Multiple choice with one correct answer
// Solution is the index of the answer
{
  _id: <ObjectId>,
  quiz: <ObjectId>,
  prompt: 'What is a prime number?',
  type: 'multiple-choice',
  options: ['3','6','8','1'],
  solution: [0],
  weight: 1
}
// Multiple choice with more than one correct answer
{
  _id: <ObjectId>,
  quiz: <ObjectId>,
  prompt: 'What is a fruit?',
  type: 'multiple-choice',
  options: ['apple','orange','brocolli','eggplant'],
  solution: [0, 1],
  weight: 1,
}
// Text question
{
  _id: <ObjectId>,
  quiz: <ObjectId>,
  prompt: 'What is a hash function?',
  type: 'text',
  options: null,
  solution: null,
  weight: 1,
}
```

### <a name="submission"></a>Submission

```js
{
  _id: <ObjectId>,
  student: <ObjectId>,
  quiz: <ObjectId>,
  answers: [
    // submission to a multiple choice with one correct answer
    {
      question: <ObjectId>,
      answer: "1",
      correct: false,
      score: 0
    },
    // submission to a multiple choice with more than one correct answer
    {
      question: <ObjectId>,
      answer: "0, 1",
      correct: true,
      score: 1
    },
    // submission to a text question
    {
      question: <ObjectId>,
      answer: "Hash function returns a value used to index into a hash table",
      correct: true,
      score: 1
    }
  ],
  tot_score: 100
}
```

## <a name="api-endpoints"></a>API Endpoints

### `quizzes`

+ `GET /api/quizzes` - returns quizzes authored by teacher
+ `GET /api/quizzes/:id` - returns single quiz 
+ `POST /api/quizzes` - creates a quiz 
+ `PATCH /api/quizzes/:id` - edit a quiz 
+ `DELETE /api/quizzes/:id` - remove a quiz

### `questions`

+ `GET /api/questions/:id` - returns a single question 
+ `GET /api/quizzes/:quiz_id/questions` - returns all questions 
+ `POST /api/quizzes/:quiz_id/questions` - creates a question 
+ `PATCH /api/questions/:id` - edit a question 
+ `DELETE /api/questions/:id` - remove a question

### `submissions`

+ `POST /api/submissions/:id/questions/:question_id` - submit an answer, submit a score 
+ `PATCH /api/submissions/:id/questions/:question_id` - edit an answer, edit a score 
+ `DELETE /api/submissions/:id/questions/:question_id` - remove an answer, remove a score

## <a name="database-queries"></a>Database queries

<!-- To do -->

Teachers

1. Assign quiz to student(s)
2. Grade Students answers
<!-- insert into submission id and update question object in answer array -->
3. Create quizzes
4. Add and Remove questions
5. Make changes to quizzes
6. Add and Remove multiple choice options
<!-- upsert into question document -->

Students

1. Record answers
<!-- insert into submission answers array -->
2. Read questions from quiz
3. Read assigned quizzes
