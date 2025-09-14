# Exercise 2

## 1.
```plaintext
concept PasswordAuthentication
purpose limit access to known users
principle after a user registers with a username and a password,
  they can authenticate with that same username and password
  and be treated each time as the same user

state
  a set of Users with
    a username String
    a password String
```

## 2.
```plaintext
actions

register (username: String, password: String): (user: User)
  requires no user with this username exists
  effects create a new User with this username and password

authenticate (username: String, password: String): (user: User)
  requires a User exists with this username and corresponding password
  effects none (authentication is successful)
```

## 3.
- Invariant: No two users may have the same username.
- Preservation: This is preserved by register’s **requires**, which prevents duplicates. Since authenticate does not mutate the state, it cannot violate the invariant.

## 4.
```plaintext
concept PasswordAuthentication
purpose limit access to known users

state
  a set of Users with
    a username String
    a password String
    a status of PENDING or ACTIVE
    a confirmToken String

actions

register (username: String, password: String): (user: User, token: String)
  requires no user with this username exists
  effects
    create a new User with username, password, status = PENDING
    generate a new token and set user.confirmToken := token

confirm (username: String, token: String): (user: User)
  requires a User exists with this username
           and user.status = PENDING
           and user.confirmToken = token
  effects set user.status := ACTIVE

authenticate (username: String, password: String): (user: User)
  requires a User exists with this username and corresponding password
           and user.status = ACTIVE
  effects none
```
