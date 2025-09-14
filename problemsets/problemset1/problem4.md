# Exercise 4

## Concept: URLShortener [User]

**purpose**
map long URLs to short suffixes that redirect them

**principle**
a user creates a short link by supplying a (long) URL and optionally a custom suffix;
if no suffix is supplied, the system generates a unique random suffix;
the short link can be shared, and anyone who visits it is redirected to the original long URL;
the system does not allow two short links to use the same suffix.

**state**
```plaintext
a set of Links with
  a originalURL String
  an optional suffix String
  an owner User
```

**actions**
```plaintext
create (owner: User, originalURL: String, optional suffix: String): (link: Link)
  requires if suffix is provided, no existing link uses that suffix
  effects if suffix is provided, create new link with this suffix;
          otherwise generate a new suffix and create link

resolve (suffix: String): (originalURL: String)
  requires a Link with this suffix exists
  effects none

delete (owner: User, link: Link)
  requires link exists and link.owner = owner
  effects remove link from Links
```

**Notes**
Because suffix is optional in the create action, this concept supports both auto-generated and user-provided suffixes. When the suffix is absent, one will automatically be generated (at random). The **create** requires ensures suffix uniqueness, preserving the invariant that no two links share the same suffix. The owner field is not necessary, but it allows users to manage their own links.


## Concept: BillableHours [User, Project]

**purpose**
track employee billable hours by recording time spent per project

**principle**
an employee starts a timed session; they select a project and entering a description for it;
the employee ends the session to record when it ended;
an employee cannot have more than one session active at a time;
if a new session is started while one is active, the previous session is ended automatically.

**state**
```plaintext
a set of Sessions with
  an employee User
  a Project
  a description String
  a start DateTime
  an optional end DateTime
```

**actions**
```plaintext
start (employee: User, project: Project, description: String, now: DateTime): (session: Session)
  requires employee exists and project exists
  effects if a session sess exists with sess.employee = employee and sess.end is not set
            then set sess.end := now
          create a new session with employee, project, description, start = now

end (employee: User, session: Session, now: DateTime)
  requires session exists
           and session.employee = employee
           and session.end is not set
           and now >= session.start
  effects set session.end := now

duration (session: Session): (hours: Number)
  requires session exists and session.end is set
  effects none (returns session.end − session.start)
```

**Notes**
Each employee may have at most one active session at a time. This invariant is maintained by ensuring that start ends any ongoing session for that employee. This addresses the case where a worker forgets to end the session. An alternative would be to automatically end sessions at a fixed cutoff (e.g., 5pm or end of shift), but keeping sessions indefinite is more general since a lot of factories work 24/7.

## Concept: ConferenceRoomBooking [User, Room]

**purpose**
allow users to book conference rooms for specific time intervals

**principle**
a user can book an available room for a given start and end time;
a user may cancel their booking at any time;
the system does not allow bookings with conflicting times in the same room;
others can view which rooms are available.

**state**
```plaintext
a set of Bookings with
  a room Room
  a user User
  a start DateTime
  an end DateTime
```

**actions**
```plaintext
book (user: User, room: Room, start: DateTime, end: DateTime): (booking: Booking)
  requires start < end
           and no existing booking for this room overlaps
  effects create a new booking with these fields

cancel (user: User, booking: Booking)
  requires booking exists and booking.user = user
  effects remove booking from Bookings

query (room: Room, start: DateTime, end: DateTime): (available: Flag)
  requires start < end
  effects none (flag is true if no booking for this room overlaps with the interval, else false)
```

**Notes**
The main invariant is that no room can be booked by more than one user during the same time interval. This is preserved through the book action's requires. The query action is not strictly essential, but it is a useful feature for checking availability that many booking services offer.
