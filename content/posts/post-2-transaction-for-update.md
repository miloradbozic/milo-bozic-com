+++
date = '2025-04-03T01:41:41+02:00'
draft = true
title = 'Transaction for Update'
tags = ["SQL", "Transaction", "FOR UPDATE", "Database Optimization"]
+++

## Introduction

When working with transactions in SQL, one of the challenges you might encounter is ensuring that operations on a shared resource are handled correctly without causing 
multiple event emissions or race conditions. A common issue arises when multiple processes or users are updating the same record concurrently, leading to duplicated event 
emissions or conflicting operations.

In this post, I'll walk through how I initially used the `FOR UPDATE` clause in a transaction to avoid such problems and later transitioned to a simpler solution using `FOR 
UPDATE RETURNING`.

## Initial Approach: Using `FOR UPDATE`

To prevent multiple processes from modifying the same data concurrently, I used `FOR UPDATE` in my SQL transaction. The `FOR UPDATE` clause locks the selected rows so that 
no other transactions can modify them until the current transaction is complete. This ensures that the update operation is atomic and that other processes wait for the 
current transaction to finish before proceeding.

Here’s an example query I used initially:

```sql
BEGIN;

SELECT * FROM my_table
WHERE some_condition = true
FOR UPDATE;

-- Perform updates on the selected rows
UPDATE my_table
SET column_name = 'new_value'
WHERE some_condition = true;

COMMIT;
```

In this setup, the FOR UPDATE effectively prevented multiple processes from making conflicting updates to the same row. However, this solution still had its downsides, such 
as complex handling of multiple rows and a higher likelihood of contention in high-traffic environments.

Refining the Approach: Using FOR UPDATE RETURNING
After reflecting on the limitations of the initial approach, I decided to simplify the process. Instead of explicitly checking for event emissions and manually managing the 
lock, I switched to using FOR UPDATE RETURNING in my SQL query. The FOR UPDATE RETURNING clause not only locks the rows but also returns the count of the rows that were 
actually updated. This provides immediate feedback on the operation's result, which allows for easier decision-making and more efficient handling of the transaction.

Here’s the updated query I used:

```
sql
Copy
Edit
BEGIN;

WITH updated AS (
  UPDATE my_table
  SET column_name = 'new_value'
  WHERE some_condition = true
  RETURNING *
)
SELECT COUNT(*) FROM updated;

COMMIT;
```

This method returns the count of updated rows directly, which simplifies the logic in the application and eliminates the need for manually handling multiple row updates or 
checking for locks.

Conclusion: Which Method Is Better?
After comparing both methods, I found that using FOR UPDATE RETURNING is a more efficient and simplified approach. Here's why:

Simplicity: The FOR UPDATE RETURNING method reduces the complexity by combining the row lock and the update count in one query. There's no need for additional logic to track 
locks manually.

Performance: With FOR UPDATE RETURNING, you get immediate feedback on how many rows were actually updated, allowing for better performance in high-concurrency environments.

Readability: The simpler query is easier to understand and maintain in the long term, reducing the chance for errors.

While FOR UPDATE works well for locking and updating rows, FOR UPDATE RETURNING offers a more concise and efficient solution, especially when you need to know the exact 
count of rows affected by the update. In the end, FOR UPDATE RETURNING is the better choice for simplifying event emission handling and ensuring that transactions are 
executed more efficiently.
