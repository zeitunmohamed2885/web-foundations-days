
# Library Books REST API

## Base URL

`https://api.example.com/api/v1`

Resource: `/books`

### 1. List all books

- **Method:** GET
- **Path:** `/books`
- **Description:** Returns a list of all books in the library.
- **Success status:** `200 OK`
- **Example request body:** Not required.

### 2. Get one book

- **Method:** GET
- **Path:** `/books/{id}`
- **Description:** Returns details of a book identified by its ID.
- **Success status:** `200 OK`
- **Example request body:** Not required.

### 3. Create a book

- **Method:** POST
- **Path:** `/books`
- **Description:** Creates a new book in the library.
- **Success status:** `201 Created`
- **Example request body:**

```json
{
  "title": "Things Fall Apart",
  "author": "Chinua Achebe",
  "published_year": 1958
}
```

### 4. Update a book

- **Method:** PUT
- **Path:** `/books/{id}`
- **Description:** Replaces the details of an existing book.
- **Success status:** `200 OK`
- **Example request body:**

```json
{
  "title": "Things Fall Apart",
  "author": "Chinua Achebe",
  "published_year": 1958
}
```

### 5. Delete a book

- **Method:** DELETE
- **Path:** `/books/{id}`
- **Description:** Deletes the specified book from the library.
- **Success status:** `204 No Content`
- **Example request body:** Not required.

### 6. List books by author

- **Method:** GET
- **Path:** `/books?author=Chinua%20Achebe`
- **Description:** Returns books written by the specified author.
- **Success status:** `200 OK`
- **Example request body:** Not required. The author is supplied as a query parameter.

### 7. Search books by title

- **Method:** GET
- **Path:** `/books?title=Things%20Fall%20Apart`
- **Description:** Returns books whose titles match the supplied search term.
- **Success status:** `200 OK`
- **Example request body:** Not required. The title is supplied as a query parameter.

## Error Responses

### 400 Bad Request

- **Description:** The request contains invalid data or missing required fields.
- **Example:** Creating a book without a required title or supplying an invalid publication year.

### 404 Not Found

- **Description:** The requested resource does not exist.
- **Example:** Requesting `GET /books/9999` when book ID `9999` does not exist.
