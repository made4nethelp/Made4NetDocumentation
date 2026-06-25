# Force User Logout Enhancement

## Overview

A new force logout mechanism was implemented to allow the Scheduler Service to remotely log out users from the SCExpert web application.

Since the Scheduler Service runs as a separate class library and does not have access to ASP.NET Session objects, direct session termination is not possible. To solve this, a database-driven force logout approach was introduced.

---

## Problem Statement

The existing scheduler logout process attempted to terminate user sessions by:

- Removing records from `WHACTIVITY`
- Removing records from `SYS_LICDT`
- Disconnecting licenses

Although these actions removed tracking and licensing information, users remained logged into the web application because their ASP.NET sessions were still active.

The Scheduler Service does not have access to:

- `HttpContext.Current`
- `Session`
- `FormsAuthentication`
- User browser sessions

Therefore, it cannot directly invoke:

```vb
Session.Abandon()
FormsAuthentication.SignOut()
```

---

## Solution Design

A new database flag named `FORCELOGOUT` was added to the `SYS_USERS` table.

### Database Change

```sql
ALTER TABLE SYS_USERS
ADD FORCELOGOUT BIT NOT NULL DEFAULT(0)
```

### Scheduler Behavior

When the scheduler determines that users should be logged out, it updates the user record:

```sql
UPDATE SYS_USERS
SET FORCELOGOUT = 1
WHERE USERNAME = @UserName
```

This marks the user for forced logout.

---

## Scheduler Implementation

The logout scheduler was updated to:

1. Retrieve users from USERLOGINAUDIT
2. Remove user activity records
3. Set the FORCELOGOUT flag

Example:

```csharp
string sqlUpdate =
    $"UPDATE SYS_USERS " +
    $"SET FORCELOGOUT = 1 " +
    $"WHERE USERNAME = '{userName.Replace(\"'\", \"''\")}'";

Made4Net.DataAccess.DataInterface.RunSQL(
    sqlUpdate,
    "Made4NetSchema");
```

---

## Global.asax Enhancement

The ASP.NET application was updated to check the FORCELOGOUT flag during each request.

### Event Used

```vb
Application_AcquireRequestState
```

This event is automatically executed by ASP.NET during every request after session state has been loaded.

### Logout Flow

1. User makes any request.
2. Application_AcquireRequestState() executes.
3. Application checks SYS_USERS.FORCELOGOUT.
4. If the flag is set:
   - Reset flag to 0
   - Disconnect license
   - Close database connections
   - Sign out authentication
   - Clear session
   - Abandon session

---

## Active User View

```sql
CREATE VIEW vActiveUsersLast24Hours
AS
SELECT
    USERNAME,
    MAX(LOGINTIME) AS LASTLOGINTIME
FROM USERLOGINAUDIT
WHERE EVENTTYPE = 'SUCCESS'
  AND LOGINTIME >= DATEADD(HOUR, -24, GETDATE())
GROUP BY USERNAME
```

Usage:

```sql
SELECT *
FROM vActiveUsersLast24Hours
ORDER BY LASTLOGINTIME DESC
```

---

## Testing

### Test Case 1 – Force Logout

```sql
UPDATE SYS_USERS
SET FORCELOGOUT = 1
WHERE USERNAME = 'Admin'
```

Refresh the browser and verify the user is redirected to login.

### Test Case 2 – License Cleanup

Verify:
- Session terminated
- License disconnected
- Database connections released

### Test Case 3 – Flag Reset

Verify:

```sql
SELECT FORCELOGOUT
FROM SYS_USERS
WHERE USERNAME = 'Admin'
```

returns 0 after logout processing.

---

## Benefits

- No dependency on ASP.NET Session from Scheduler Service
- Uses standard ASP.NET request lifecycle
- Ensures proper license cleanup
- Supports remote logout of active users
- Minimal impact on existing application architecture
