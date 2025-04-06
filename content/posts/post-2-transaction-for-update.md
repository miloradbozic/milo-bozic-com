+++
date = '2025-04-03T01:41:41+02:00'
draft = false
title = 'Transaction for Update'
tags = ["SQL", "Transaction", "FOR UPDATE", "Database Optimization"]
+++

## Introduction

(This is for Mysql. Postgresql has different mechanism like RETURNING)

```go
func (s *Service) UnenrollMembership(
	ctx  context.Context,
	accountID  string,
) error {
	s.membershipRepo.UnenrollMembership(
		ctx,
		accountID,
	)
	if err != nil {
		return err
	}
	
	s.produceUnenrollmentEvent(ctx, accountID)
}

func (s *ChannelMembershipRepo) UnenrollChannelMembership(ctx context.Context, accountID string) error {
	params  :=  db.UpdateMembershipEndTimeParams{
		AccountID: accountID,
		EndAt: sql.NullTime{
			Time: time.Now(),
			Valid: true,
		},
	}
	err  :=  s.queries.UpdateMembershipEndTime(ctx, params)
	if  err  !=  nil {
		return  err
	}
	return nil
}

/* name: UpdateEnrollmentChannelMembershipEndTime :exec */ 
-- UpdateEnrollmentChannelMembershipEndTime for any row of a given account_id with NULL end_time, updates the end_time 
-- to the given time 
UPDATE membership SET end_at = sqlc.arg(end_at) WHERE account_id = ? AND (end_at IS NULL OR end_at > sqlc.arg(end_at));

```
SQL code generated from sqlc query:
```sql
const updateEnrollmentChannelMembershipEndTime = -- name: UpdateEnrollmentChannelMembershipEndTime :exec
UPDATE membership SET end_at = ? WHERE account_id = ? AND (end_at IS NULL OR end_at > ?)
```
 
## Initial Approach: Using `ROW_COUNT()`

:exec replaced  :execrows
 Does execrows use ```sql SELECT ROW_COUNT();``` internally?
```go
/* name: UpdateEnrollmentChannelMembershipEndTime :execrows */ 
-- UpdateEnrollmentChannelMembershipEndTime for any row of a given account_id with NULL end_time, updates the end_time 
-- to the given time 
UPDATE membership SET end_at = sqlc.arg(end_at) WHERE account_id = ? AND (end_at IS NULL OR end_at > sqlc.arg(end_at));
```

SQL code generated from sqlc query:
```sql
const updateEnrollmentChannelMembershipEndTime = -- name: UpdateEnrollmentChannelMembershipEndTime :execrows
UPDATE membership SET end_at = ? WHERE account_id = ? AND (end_at IS NULL OR end_at > ?)
```

```go
func (s *MembershipRepo) UnenrollMembership(ctx context.Context, accountID string) (bool, error) {
	params  :=  db.UpdateMembershipEndTimeParams{
		AccountID: accountID,
		EndAt: sql.NullTime{
			Time: time.Now(),
			Valid: true,
		},
	}
	rowsAffected, err  :=  s.queries.UpdateMembershipEndTime(ctx, params)
	if  err  !=  nil {
		return  false, err
	}
	return rowsAffected > 0, nil
}

func (s *Service) UnenrollMembership(
	ctx  context.Context,
	accountID  string,
) error {
	updated, err := s.membershipRepo.UnenrollMembership(
		ctx,
		accountID,
	)
	if err != nil {
		return err
	}

	if updated {
		s.produceUnenrollmentEvent(ctx, accountID)
	}
}

```
 
  ## The test

```go
func (s *DBTestSuiteMapPDID) TestMultipleUpdateMembership() {
	accountID  :=  "account_id3"
	endAt  :=  time.Now()

	params  :=  UpdateEnrollmentChannelMembershipEndTimeParams{
		AccountID: accountID,
		EndAt: commonsql.TimeToNullTime(endAt),
	}

	numGoroutines  :=  100
	var  wg  sync.WaitGroup
	wg.Add(numGoroutines)

	results := make(chan bool, numGoroutines)
	for  i  :=  0; i  <  numGoroutines; i++ {
		go  func() {
		defer  wg.Done()
			rowsAffected, err  :=  s.q.UpdateEnrollmentChannelMembershipEndTime(
				context.Background(),
				params,
			)

			updated  :=  rowsAffected  >  0
			require.NoError(s.T(), err)
			results  <-  updated
		}()
}

	wg.Wait()
	close(results)

	var  updateCount  int
	for  updated  :=  range  results {
	if  updated {
		updateCount++
	}
}

require.Equal(s.T(), 1, updateCount, "only one transaction should update the row")
var  result  bool
err  :=  s.DB.QueryRow(
"SELECT COUNT(*) > 0 FROM enrollment_channel_membership WHERE account_id = ? AND end_at = ?",
accountID, endAt,
).Scan(&result)

require.NoError(s.T(), err)
require.True(s.T(), result)
}
```
  
 ## Problems with the approach

 ## Second solution - using  `FOR UPDATE` inside of transaction

## Deep Dive 

Why for update works and update doesnt event though UPDATE locks the rows? Whats different really? Does inside transaction FOR UPDATE checks if someone already ASKED for a lock?