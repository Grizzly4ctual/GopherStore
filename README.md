# Gopher Store: Project Overview
Gopher Store is a lightweight relational database crafted entirely in Go, designed as a learning project to explore the inner workings of database systems, storage mechanisms, and transaction management. The main goal was to build a practical system while deepening understanding of how databases handle data and ensure reliability.

# Table of Contents
- Installation

- Features

- Roadmap

- Supported Commands

- Contributing

- License

# Installation
## Prerequisites
- Go (version 1.17 or newer)

- Git

- Linux or macOS (development and testing have focused on NixOS, but other Unix-like systems should work as well).

### Getting the Code
Clone the Gopher Store repository to your local machine:
```
git clone https://github.com/yourusername/gopherstore.git && cd gopherstore
```
git clone https://github.com/yourusername/gopherstore.git && cd gopherstore
### Building the Project
Install dependencies and compile the project with:

```
go mod tidy
go build -o gopherstore
```

### Running Gopher Store
To start the server, simply run:

```
./gopherstore
```

# Features

Gopher Store is designed as a practical, hands-on relational database in Go. Here are the updated features it supports:

- **Table-Based Data Storage:** Organizes information in tables with rows and columns, allowing for structured data management.
- **Primary and Foreign Key Support:** Each table can have a unique identifier (primary key), and tables can be related using foreign keys for data integrity.
- **ACID Transactions:** Ensures atomicity, consistency, isolation, and durability for all operations, keeping your data safe and reliable.
- **Simple SQL Parser:** Supports a subset of SQL for creating tables, inserting data, selecting, updating, and deleting records.
- **Efficient Data Retrieval:** Uses B+ tree indexing for fast lookups and efficient range queries.
- **Concurrent Reads and Writes:** Multiple users can read and write data at the same time, supporting multi-user access.
- **Basic Query Processing:** Handles straightforward queries and simple WHERE conditions for data filtering.
- **Planned Enhancements:** Future updates will include support for more data types, joins, advanced queries, and improved indexing.



# Supported Commands

| Command    | Description                                         |
|------------|-----------------------------------------------------|
| CREATE     | Create a new table or index                         |
| DROP       | Remove a table or index                             |
| INSERT     | Add new records to a table                          |
| SELECT     | Retrieve data from one or more tables               |
| UPDATE     | Modify existing records in a table                  |
| DELETE     | Remove records from a table                         |
| BEGIN      | Start a new transaction                             |
| COMMIT     | Save all changes made in the current transaction    |
| ROLLBACK   | Undo all changes made in the current transaction    |
| QUIT/EXIT  | Exit the database shell                             |

# License

Gopher Store is open source and released under the MIT License.


