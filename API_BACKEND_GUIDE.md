# Riddle Lock System - API Backend Guide

This document explains how to update your R Plumber API at `cardioid.co.nz/api` to support the riddle lock system.

## Overview

The system needs to:
1. Store riddles with their word answers and corresponding digits
2. Allow authorized users to submit new riddles
3. Allow authorized users to activate riddles (make them visible to players)
4. Allow public users to check their answers and receive digits
5. Fetch currently active riddles

## Database Schema

Create a new SQLite database file called `riddles.sqlite` in your API directory.

### Table: `riddles`

```sql
CREATE TABLE riddles (
    id TEXT PRIMARY KEY,
    riddle_text TEXT NOT NULL,
    answer TEXT NOT NULL,
    digit INTEGER NOT NULL CHECK (digit >= 0 AND digit <= 9),
    position INTEGER NOT NULL CHECK (position >= 1 AND position <= 3),
    week INTEGER,
    is_active INTEGER DEFAULT 0,
    is_expired INTEGER DEFAULT 0,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    activated_at TEXT,
    expired_at TEXT,
    notes TEXT
);
```

Fields:
- `id`: Unique identifier (UUID)
- `riddle_text`: The riddle question
- `answer`: The correct answer (stored in lowercase for case-insensitive matching)
- `digit`: The digit (0-9) revealed when solved
- `position`: Which position in the 3-digit combination (1, 2, or 3)
- `week`: Optional week number for organization
- `is_active`: Boolean (0 or 1) - whether the riddle is currently active
- `is_expired`: Boolean (0 or 1) - whether the riddle is expired (still visible but doesn't count toward score)
- `created_at`: Timestamp when riddle was created
- `activated_at`: Timestamp when riddle was activated
- `expired_at`: Timestamp when riddle was expired
- `notes`: Optional admin notes

## Required Endpoints

Add these endpoints to your `plumber.R` file:

### 1. GET /riddles/active (Public)

Returns all currently active and expired riddles (without answers).

```r
#* Get active and expired riddles
#* @get /riddles/active
#* @serializer unboxedJSON
#* @tag riddles
get_active_riddles <- function() {
  tryCatch({
    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    if (!dbExistsTable(conn = con, name = "riddles")) {
      return(list(
        status = "success",
        active_riddles = list(),
        expired_riddles = list()
      ))
    }

    # Get active riddles (not expired), excluding the answer field
    active_riddles <- dbGetQuery(
      con,
      "SELECT id, riddle_text, position, week, is_expired FROM riddles WHERE is_active = 1 AND is_expired = 0 ORDER BY position"
    )

    # Get expired riddles, excluding the answer field
    expired_riddles <- dbGetQuery(
      con,
      "SELECT id, riddle_text, position, week, is_expired FROM riddles WHERE is_active = 1 AND is_expired = 1 ORDER BY position"
    )

    return(list(
      status = "success",
      active_riddles = active_riddles,
      expired_riddles = expired_riddles
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Database error:", e$message)
    ))
  })
}
```

### 2. POST /riddles/check (Public)

Checks if an answer is correct and returns the digit if it is.

```r
#* Check riddle answer
#* @param riddle_id The ID of the riddle
#* @param answer The user's answer
#* @post /riddles/check
#* @serializer unboxedJSON
#* @tag riddles
check_riddle_answer <- function(riddle_id, answer) {
  tryCatch({
    # Validate inputs
    if (is.null(riddle_id) || riddle_id == "" || is.null(answer) || answer == "") {
      return(list(
        status = "error",
        message = "Missing riddle_id or answer"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    # Get the riddle
    riddle <- dbGetQuery(
      con,
      "SELECT id, answer, digit, position, is_expired FROM riddles WHERE id = $1 AND is_active = 1",
      params = list(riddle_id)
    )

    if (nrow(riddle) == 0) {
      return(list(
        status = "error",
        correct = FALSE,
        message = "Riddle not found or not active"
      ))
    }

    # Check answer (case-insensitive, trimmed)
    user_answer <- tolower(trimws(answer))
    correct_answer <- tolower(trimws(riddle$answer))

    if (user_answer == correct_answer) {
      return(list(
        status = "success",
        correct = TRUE,
        digit = riddle$digit,
        position = riddle$position,
        is_expired = riddle$is_expired,
        message = "Correct!"
      ))
    } else {
      return(list(
        status = "success",
        correct = FALSE,
        message = "Incorrect answer. Try again!"
      ))
    }

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Error:", e$message)
    ))
  })
}
```

### 3. POST /riddles/submit (Authenticated)

Submit a new riddle (admin only).

```r
#* Submit a new riddle
#* @param riddle_text The riddle question
#* @param answer The correct answer
#* @param digit The digit to reveal (0-9)
#* @param position Position in combination (1-3)
#* @param week Optional week number
#* @param notes Optional admin notes
#* @param auth_code Authentication code
#* @post /riddles/submit
#* @serializer unboxedJSON
#* @tag riddles
submit_riddle <- function(riddle_text, answer, digit, position, auth_code, week = NULL, notes = "") {
  tryCatch({
    # Validate auth code
    source("antitrusties_creds.R")
    if (is.null(auth_code) || auth_code != expected_code) {
      return(list(
        status = "error",
        message = "Invalid authentication code"
      ))
    }

    # Validate inputs
    if (is.null(riddle_text) || riddle_text == "" ||
        is.null(answer) || answer == "") {
      return(list(
        status = "error",
        message = "Missing required fields"
      ))
    }

    digit <- as.integer(digit)
    position <- as.integer(position)

    if (is.na(digit) || digit < 0 || digit > 9) {
      return(list(
        status = "error",
        message = "Digit must be between 0 and 9"
      ))
    }

    if (is.na(position) || position < 1 || position > 3) {
      return(list(
        status = "error",
        message = "Position must be 1, 2, or 3"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    # Create table if it doesn't exist
    if (!dbExistsTable(conn = con, name = "riddles")) {
      dbExecute(con, "
        CREATE TABLE riddles (
            id TEXT PRIMARY KEY,
            riddle_text TEXT NOT NULL,
            answer TEXT NOT NULL,
            digit INTEGER NOT NULL CHECK (digit >= 0 AND digit <= 9),
            position INTEGER NOT NULL CHECK (position >= 1 AND position <= 3),
            week INTEGER,
            is_active INTEGER DEFAULT 0,
            is_expired INTEGER DEFAULT 0,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP,
            activated_at TEXT,
            expired_at TEXT,
            notes TEXT
        )
      ")
    }

    # Insert new riddle
    riddle_id <- UUIDgenerate()

    dbExecute(
      con,
      "INSERT INTO riddles (id, riddle_text, answer, digit, position, week, notes, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8)",
      params = list(
        riddle_id,
        riddle_text,
        tolower(trimws(answer)),
        digit,
        position,
        week,
        notes,
        format(Sys.time(), "%Y-%m-%d %H:%M:%S")
      )
    )

    return(list(
      status = "success",
      message = "Riddle submitted successfully",
      riddle_id = riddle_id
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Error:", e$message)
    ))
  })
}
```

### 4. POST /riddles/activate (Authenticated)

Activate a riddle to make it visible to players.

```r
#* Activate a riddle
#* @param riddle_id The ID of the riddle to activate
#* @param auth_code Authentication code
#* @post /riddles/activate
#* @serializer unboxedJSON
#* @tag riddles
activate_riddle <- function(riddle_id, auth_code) {
  tryCatch({
    # Validate auth code
    source("antitrusties_creds.R")
    if (is.null(auth_code) || auth_code != expected_code) {
      return(list(
        status = "error",
        message = "Invalid authentication code"
      ))
    }

    if (is.null(riddle_id) || riddle_id == "") {
      return(list(
        status = "error",
        message = "Missing riddle_id"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    # Check if riddle exists
    riddle <- dbGetQuery(
      con,
      "SELECT id, is_active FROM riddles WHERE id = $1",
      params = list(riddle_id)
    )

    if (nrow(riddle) == 0) {
      return(list(
        status = "error",
        message = "Riddle not found"
      ))
    }

    if (riddle$is_active == 1) {
      return(list(
        status = "success",
        message = "Riddle is already active"
      ))
    }

    # Activate the riddle
    dbExecute(
      con,
      "UPDATE riddles SET is_active = 1, activated_at = $1 WHERE id = $2",
      params = list(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), riddle_id)
    )

    return(list(
      status = "success",
      message = "Riddle activated successfully"
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Error:", e$message)
    ))
  })
}
```

### 5. POST /riddles/deactivate (Authenticated)

Deactivate a riddle to hide it from players.

```r
#* Deactivate a riddle
#* @param riddle_id The ID of the riddle to deactivate
#* @param auth_code Authentication code
#* @post /riddles/deactivate
#* @serializer unboxedJSON
#* @tag riddles
deactivate_riddle <- function(riddle_id, auth_code) {
  tryCatch({
    # Validate auth code
    source("antitrusties_creds.R")
    if (is.null(auth_code) || auth_code != expected_code) {
      return(list(
        status = "error",
        message = "Invalid authentication code"
      ))
    }

    if (is.null(riddle_id) || riddle_id == "") {
      return(list(
        status = "error",
        message = "Missing riddle_id"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    # Deactivate the riddle
    rows_affected <- dbExecute(
      con,
      "UPDATE riddles SET is_active = 0 WHERE id = $1",
      params = list(riddle_id)
    )

    if (rows_affected == 0) {
      return(list(
        status = "error",
        message = "Riddle not found"
      ))
    }

    return(list(
      status = "success",
      message = "Riddle deactivated successfully"
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Error:", e$message)
    ))
  })
}
```

### 6. POST /riddles/expire (Authenticated)

Mark a riddle as expired. Expired riddles are still visible but don't count toward score.

```r
#* Mark a riddle as expired
#* @param riddle_id The ID of the riddle to expire
#* @param auth_code Authentication code
#* @post /riddles/expire
#* @serializer unboxedJSON
#* @tag riddles
expire_riddle <- function(riddle_id, auth_code) {
  tryCatch({
    # Validate auth code
    source("antitrusties_creds.R")
    if (is.null(auth_code) || auth_code != expected_code) {
      return(list(
        status = "error",
        message = "Invalid authentication code"
      ))
    }

    if (is.null(riddle_id) || riddle_id == "") {
      return(list(
        status = "error",
        message = "Missing riddle_id"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    # Mark the riddle as expired
    rows_affected <- dbExecute(
      con,
      "UPDATE riddles SET is_expired = 1, expired_at = $1 WHERE id = $2",
      params = list(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), riddle_id)
    )

    if (rows_affected == 0) {
      return(list(
        status = "error",
        message = "Riddle not found"
      ))
    }

    return(list(
      status = "success",
      message = "Riddle marked as expired successfully"
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Error:", e$message)
    ))
  })
}
```

### 7. POST /riddles/expire-week (Authenticated)

Mark all riddles from a specific week as expired.

```r
#* Mark all riddles in a week as expired
#* @param week The week number
#* @param auth_code Authentication code
#* @post /riddles/expire-week
#* @serializer unboxedJSON
#* @tag riddles
expire_week <- function(week, auth_code) {
  tryCatch({
    # Validate auth code
    source("antitrusties_creds.R")
    if (is.null(auth_code) || auth_code != expected_code) {
      return(list(
        status = "error",
        message = "Invalid authentication code"
      ))
    }

    if (is.null(week) || week == "") {
      return(list(
        status = "error",
        message = "Missing week parameter"
      ))
    }

    week <- as.integer(week)
    if (is.na(week)) {
      return(list(
        status = "error",
        message = "Week must be a valid number"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    # Mark all riddles in the week as expired
    rows_affected <- dbExecute(
      con,
      "UPDATE riddles SET is_expired = 1, expired_at = $1 WHERE week = $2",
      params = list(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), week)
    )

    return(list(
      status = "success",
      message = paste("Marked", rows_affected, "riddle(s) from week", week, "as expired"),
      riddles_affected = rows_affected
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Error:", e$message)
    ))
  })
}
```

### 8. GET /riddles/all (Authenticated)

Get all riddles including inactive ones (admin only).

```r
#* Get all riddles (admin)
#* @param auth_code Authentication code
#* @get /riddles/all
#* @serializer unboxedJSON
#* @tag riddles
get_all_riddles <- function(auth_code) {
  tryCatch({
    # Validate auth code
    source("antitrusties_creds.R")
    if (is.null(auth_code) || auth_code != expected_code) {
      return(list(
        status = "error",
        message = "Invalid authentication code"
      ))
    }

    con <- dbConnect(RSQLite::SQLite(), "riddles.sqlite")
    on.exit(dbDisconnect(con))

    if (!dbExistsTable(conn = con, name = "riddles")) {
      return(list(
        status = "success",
        riddles = list()
      ))
    }

    # Get all riddles
    riddles <- dbGetQuery(
      con,
      "SELECT * FROM riddles ORDER BY created_at DESC"
    )

    return(list(
      status = "success",
      riddles = riddles
    ))

  }, error = function(e) {
    return(list(
      status = "error",
      message = paste("Database error:", e$message)
    ))
  })
}
```

## CORS Configuration

Update your CORS filter to include the GitHub Pages URL where you'll host the frontend:

```r
#* Enable CORS
#* @filter cors
cors <- function(req, res) {
  allowed_origins <- c(
    "https://ntworthk.github.io",
    "https://mergers.fyi",
    "https://houstonkemp.github.io"  # Add your GitHub Pages URL
  )

  origin <- req$HTTP_ORIGIN

  if (!is.null(origin) && origin %in% allowed_origins) {
    res$setHeader("Access-Control-Allow-Origin", origin)
  } else {
    res$setHeader("Access-Control-Allow-Origin", allowed_origins[1])
  }

  res$setHeader("Access-Control-Allow-Methods", "GET, POST, DELETE, OPTIONS")
  res$setHeader("Access-Control-Allow-Headers", "Content-Type")

  if (req$REQUEST_METHOD == "OPTIONS") {
    res$status <- 200
    return(list())
  } else {
    plumber::forward()
  }
}
```

## Usage Examples

### Submitting a New Riddle

```bash
curl -X POST "https://cardioid.co.nz/api/riddles/submit" \
  -H "Content-Type: application/json" \
  -d '{
    "riddle_text": "I speak without a mouth and hear without ears. I have no body, but I come alive with wind. What am I?",
    "answer": "echo",
    "digit": 7,
    "position": 1,
    "week": 1,
    "notes": "Week 1, Monday",
    "auth_code": "YOUR_AUTH_CODE"
  }'
```

### Activating a Riddle

```bash
curl -X POST "https://cardioid.co.nz/api/riddles/activate" \
  -H "Content-Type: application/json" \
  -d '{
    "riddle_id": "abc-123-def-456",
    "auth_code": "YOUR_AUTH_CODE"
  }'
```

### Viewing All Riddles (Admin)

```bash
curl "https://cardioid.co.nz/api/riddles/all?auth_code=YOUR_AUTH_CODE"
```

## Admin Helper Script (Optional)

You might want to create a simple R script to help manage riddles:

```r
# riddle_manager.R
library(httr)
library(jsonlite)

API_URL <- "https://cardioid.co.nz/api"
source("antitrusties_creds.R")  # Contains expected_code

# Submit a new riddle
submit_riddle <- function(riddle_text, answer, digit, position, week = NULL, notes = "") {
  response <- POST(
    paste0(API_URL, "/riddles/submit"),
    body = list(
      riddle_text = riddle_text,
      answer = answer,
      digit = digit,
      position = position,
      week = week,
      notes = notes,
      auth_code = expected_code
    ),
    encode = "json"
  )

  content(response)
}

# Activate a riddle
activate_riddle <- function(riddle_id) {
  response <- POST(
    paste0(API_URL, "/riddles/activate"),
    body = list(
      riddle_id = riddle_id,
      auth_code = expected_code
    ),
    encode = "json"
  )

  content(response)
}

# Get all riddles
get_all_riddles <- function() {
  response <- GET(
    paste0(API_URL, "/riddles/all"),
    query = list(auth_code = expected_code)
  )

  content(response)
}

# Example usage:
# submit_riddle(
#   "What has keys but no locks?",
#   "keyboard",
#   5,
#   2,
#   week = 1,
#   notes = "Week 1 - Wednesday"
# )
```

## Security Notes

1. **Authentication**: Uses the same `auth_code` system as your existing authenticated endpoints
2. **Answer Storage**: Answers are stored in lowercase to enable case-insensitive matching
3. **Public Endpoints**: Only `/riddles/active` and `/riddles/check` are public - these don't expose answers
4. **CORS**: Make sure to add your GitHub Pages URL to the allowed origins

## Testing

Before going live, test each endpoint:

1. Submit a test riddle
2. Verify it's not visible via `/riddles/active`
3. Activate it
4. Verify it appears via `/riddles/active`
5. Test checking with wrong answer
6. Test checking with correct answer (should return digit)
7. Deactivate it
8. Verify it's no longer visible

## Deployment Checklist

- [ ] Add all endpoint functions to `plumber.R`
- [ ] Update CORS configuration with GitHub Pages URL
- [ ] Create `riddles.sqlite` database (will be created automatically on first use)
- [ ] Test all endpoints with curl or Postman
- [ ] Submit 3 test riddles for positions 1, 2, and 3
- [ ] Deploy to cardioid.co.nz/api
- [ ] Test from the frontend website

## Example Riddles for Testing

Here are some example riddles to get you started:

**Position 1 (Digit: 4):**
"I speak without a mouth and hear without ears. I have no body, but I come alive with wind. What am I?"
Answer: echo

**Position 2 (Digit: 2):**
"The more you take, the more you leave behind. What am I?"
Answer: footsteps

**Position 3 (Digit: 7):**
"I have cities, but no houses. I have mountains, but no trees. I have water, but no fish. What am I?"
Answer: map
