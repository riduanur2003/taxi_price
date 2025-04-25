# Adding a Database to Your Go Taxi Booking System

I'll show you how to integrate **PostgreSQL** (but you can adapt this for MySQL/SQLite) with your Go backend. We'll cover:

## **1. Setting Up PostgreSQL**
### Install PostgreSQL
- **Linux (Debian/Ubuntu)**:
  ```bash
  sudo apt update
  sudo apt install postgresql postgresql-contrib
  sudo systemctl start postgresql
  ```

- **macOS** (using Homebrew):
  ```bash
  brew install postgresql
  brew services start postgresql
  ```

- **Windows**: Download from [postgresql.org](https://www.postgresql.org/download/windows/)

### Create Database and User
```bash
sudo -u postgres psql
```
```sql
CREATE DATABASE taxi_booking;
CREATE USER taxi_user WITH PASSWORD 'securepassword';
GRANT ALL PRIVILEGES ON DATABASE taxi_booking TO taxi_user;
\q
```

---

## **2. Go Database Setup**
### Install Required Packages
```bash
go get github.com/lib/pq  # PostgreSQL driver
go get github.com/jmoiron/sqlx  # Easier SQL handling
```

### **`backend/database/database.go`**
```go
package database

import (
	"database/sql"
	"fmt"
	_ "github.com/lib/pq"
	"github.com/jmoiron/sqlx"
)

const (
	host     = "localhost"
	port     = 5432
	user     = "taxi_user"
	password = "securepassword"
	dbname   = "taxi_booking"
)

func InitDB() (*sqlx.DB, error) {
	psqlInfo := fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=disable",
		host, port, user, password, dbname)

	db, err := sqlx.Connect("postgres", psqlInfo)
	if err != nil {
		return nil, err
	}

	// Test the connection
	err = db.Ping()
	if err != nil {
		return nil, err
	}

	fmt.Println("Successfully connected to PostgreSQL!")
	return db, nil
}
```

---

## **3. Database Schema**
### Create Tables
Run this in `psql` or via a migration tool:
```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Drivers table
CREATE TABLE drivers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    license_number VARCHAR(50) UNIQUE NOT NULL,
    current_location GEOGRAPHY(POINT),
    available BOOLEAN DEFAULT TRUE
);

-- Bookings table
CREATE TABLE bookings (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    driver_id INTEGER REFERENCES drivers(id),
    pickup_address TEXT NOT NULL,
    dropoff_address TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## **4. Integrating with Go Handlers**
### **`backend/models/user.go`**
```go
package models

import "github.com/jmoiron/sqlx"

type User struct {
	ID           int    `db:"id"`
	Email        string `db:"email"`
	PasswordHash string `db:"password_hash"`
}

func CreateUser(db *sqlx.DB, email, passwordHash string) error {
	_, err := db.Exec(
		"INSERT INTO users (email, password_hash) VALUES ($1, $2)",
		email, passwordHash,
	)
	return err
}

func GetUserByEmail(db *sqlx.DB, email string) (*User, error) {
	var user User
	err := db.Get(&user, "SELECT * FROM users WHERE email = $1", email)
	return &user, err
}
```

### **`backend/handlers/auth.go`** (Updated)
```go
package handlers

import (
	"taxi-app/backend/models"
	"net/http"
)

func RegisterHandler(db *sqlx.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		email := r.FormValue("email")
		password := r.FormValue("password")

		hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
		
		if err := models.CreateUser(db, email, string(hashedPassword)); err != nil {
			http.Error(w, "Registration failed", http.StatusInternalServerError)
			return
		}

		w.WriteHeader(http.StatusCreated)
		w.Write([]byte("User created successfully"))
	}
}
```

---

## **5. Using the Database in Main**
### **Updated `main.go`**
```go
package main

import (
	"taxi-app/backend/database"
	"taxi-app/backend/handlers"
	"net/http"
)

func main() {
	// Initialize database
	db, err := database.InitDB()
	if err != nil {
		panic(err)
	}
	defer db.Close()

	// Setup routes
	http.HandleFunc("/register", handlers.RegisterHandler(db))
	http.HandleFunc("/login", handlers.LoginHandler(db))
	http.HandleFunc("/book", handlers.BookingHandler(db))

	http.ListenAndServe(":8080", nil)
}
```

---

## **6. Database Operations Examples**
### **Booking a Taxi**
```go
// backend/models/booking.go
package models

type Booking struct {
	ID            int    `db:"id"`
	UserID        int    `db:"user_id"`
	DriverID      int    `db:"driver_id"`
	PickupAddress string `db:"pickup_address"`
	DropoffAddress string `db:"dropoff_address"`
	Status        string `db:"status"`
}

func CreateBooking(db *sqlx.DB, booking Booking) error {
	_, err := db.Exec(`
		INSERT INTO bookings 
		(user_id, driver_id, pickup_address, dropoff_address) 
		VALUES ($1, $2, $3, $4)`,
		booking.UserID, booking.DriverID, 
		booking.PickupAddress, booking.DropoffAddress,
	)
	return err
}
```

### **Finding Available Drivers**
```go
func GetAvailableDrivers(db *sqlx.DB) ([]Driver, error) {
	var drivers []Driver
	err := db.Select(&drivers, 
		"SELECT * FROM drivers WHERE available = TRUE")
	return drivers, err
}
```

---

## **7. Database Best Practices**
1. **Use connection pooling**:
   ```go
   db.SetMaxOpenConns(25)
   db.SetMaxIdleConns(25)
   db.SetConnMaxLifetime(5 * time.Minute)
   ```

2. **Handle transactions**:
   ```go
   tx, err := db.Beginx()
   // ... do queries
   err = tx.Commit()
   ```

3. **Prevent SQL injection**:
   - Always use parameterized queries (`$1, $2` in PostgreSQL)
   - Never concatenate SQL strings

4. **Add indexes** for frequently queried columns:
   ```sql
   CREATE INDEX idx_drivers_location ON drivers USING GIST(current_location);
   ```

---

## **8. Testing Database Queries**
```go
func TestGetUser(t *testing.T) {
    db := setupTestDB()
    user, err := models.GetUserByEmail(db, "test@example.com")
    
    if err != nil {
        t.Fatalf("Failed: %v", err)
    }
    
    if user.Email != "test@example.com" {
        t.Errorf("Expected test@example.com, got %s", user.Email)
    }
}
```

---

## **Next Steps**
1. **Add migrations** (use [golang-migrate](https://github.com/golang-migrate/migrate))
2. **Implement caching** (Redis) for frequently accessed data
3. **Set up backups** for your database

Would you like me to cover any of these in detail?
