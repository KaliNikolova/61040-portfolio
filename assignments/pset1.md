# Problem Set 1

## Exercise 1: Reading a concept


1. **Invariants.** What are two invariants of the state? (*Hint:* one is about aggregation/counts of items, and one relates requests and purchases). Say which one is more important and why; identify the action whose design is most affected by it, and say how it preserves it.

- For any request or purchase, its count must be greater than or equal to zero.

- For every purchase in a registry, there must exist a corresponding request for the same item, and the total count of purchases for that item must be <= the original requested count.

The second invariant is more important, because it guarantees the whole registry makes sense: purchases always correspond to requested items, and nobody can over-purchase.

Action most affected: purchase. It checks that a request exists and has at least count. Then it decrements the request count so the invariant is preserved.


2. **Fixing an action.** Can you identify an action that potentially breaks this important invariant, and say how this might happen? How might this problem be fixed?

Problematic action: removeItem

If a giver already purchased an item, removing its request would break the consistency invariant — because now a purchase exists without a corresponding request.

Fix: disallow removing items that already have purchases.

Also, for the first invariant (count always being nonnegative):

Problematic action: purchase

A user could call the action with a negative count (e.g., -2). The requires clause Request.count >= -2 would pass, but the action would then incorrectly add 2 to the request count and create a nonsensical purchase record.

Fix: Add a precondition to the purchase action: requires count > 0.


3. **Inferring behavior.** The operational principle describes the typical scenario in which the registry is opened and eventually closed. But a concept specification often allows other scenarios. By reading the specs of the concept actions, say whether a registry can be opened and closed repeatedly. What is a reason to allow this?

Yes, a registry can be opened and closed repeatedly, since nothing in the requires or effects prevents calling open after close.

Reason to allow this: the recipient might want to temporarily close the registry to make updates, then reopen it for friends to continue purchasing.


4. **Registry deletion.** There is no action to delete a registry. Would this matter in practice?

In practice, not having an action to delete a registry could matter. Over time, a user might accumulate multiple registries for different events. Without a deletion feature, their account could become cluttered with old, irrelevant registries. From a data management perspective, it also means the system would store unnecessary data indefinitely. While not critical for the core functionality of a single gift registration event, it impacts the long-term user experience and system maintenance.

5. **Queries.** What are two common queries likely to be executed against the concept state? (Hint: one is executed by a registry owner, and one by a giver of a gift.)

Common registry owner query: "Show me all purchases made (items, counts, who purchased)."

Common giver query: "Show me remaining items available for purchase in this registry."

6. **Hiding purchases.** A common feature of gift registries is to allow the recipient to choose not to see purchases so that an element of surprise is retained. How would you augment the concept specification to support this?

To support hiding purchases, the Registry state could be augmented with a showPurchases flag:


```
state
    a set of Registrys with
        an owner User
        an active Flag
        a showPurchases Flag
        a set of Requests
```

We will also need to add an action for this flag:

```
actions
  togglePurchaseVisibility (registry: Registry)
    requires registry exists
    effects set the showPurchases flag to its opposite value
```

The query that a registry owner uses to view purchased items would then need to respect this flag, only showing the purchases if showPurchases is true.

7. **Generic types.** The User and Item types are specified as generic parameters. The Item type might be populated by SKU codes, for example. Explain why this is preferable to representing items with their names, descriptions, prices, etc.

- Using generic Item makes the concept independent of how items are represented.

- Avoids bloating the spec with fields like name, description, price.

- Lets other concepts handle item details, while GiftRegistration only cares about tracking requests and purchases.


## Exercise 2: Extending a familiar concept

1. Complete the definition of the concept state.

2. Write a requires/effects specification for each of the two actions. (*Hints:* The register action creates and returns a new user. The authenticate action is primarily a guard, and doesn’t mutate the state.)


```
concept PasswordAuthentication
purpose limit access to known users
principle after a user registers with a username and a password,
they can authenticate with that same username and password
and be treated each time as the same user

state
a set of Users with
    a username String
    a password String

actions
register (username: String, password: String): (user: User)
    requires a User with the given username does not already exist
    effects creates a new User with the given username and password, and returns that User 
authenticate (username: String, password: String): (user: User)
    requires a User with the given username exists and their password matches the one provided
    effects returns the matching User
```



3. What essential invariant must hold on the state? How is it preserved?

Invariant: Usernames must be unique across all Users.

Preservation: The register action preserves this invariant through its requires clause, which prevents the creation of a new User if a user with that username already exists.

4. One widely used extension of this concept requires that registration be confirmed by email. Extend the concept to include this functionality. (*Hints:* you should add (1) an extra result variable to the register action that returns a secret token that (via a sync) will be emailed to the user; (2) a new confirm action that takes a username and a secret token and completes the registration; (3) whatever additional state is needed to support this behavior.)

```
  concept PasswordAuthentication
  purpose limit access to known and verified users
  principle after a user registers with a username and a password, they receive a secret token. They must use this token to confirm their registration before they can authenticate. Once confirmed, they can authenticate with their username and password and be treated as the same user each time.
    
  state
    a set of Users with
        a username String
        a password String
        a status of PENDING or REGISTERED

    a set of SecretTokens with
        a User
        a token String

  actions
    register (username: String, password: String): (token: String)
        requires a User with the given username does not already exist
        effects creates a new User with the given username, password, and a status of PENDING. Generates a new, unique SecretToken associated with this User and returns the token string.

    confirm (username: String, token: String): (user: User)
        requires a User with the given username exists and has a PENDING status.
        Also, requires a SecretToken exists for that User with a matching token string.
        effects changes the User's status to CONFIRMED, deletes the SecretToken, and returns the confirmed User.

    authenticate (username: String, password: String): (user: User)
        requires a User with the given username exists, has a CONFIRMED status, and their password matches the one provided
        effects returns the matching User
```

## Exercise 3: Comparing concepts

Concept specification for PersonalAccessToken:

```
concept PersonalAccessToken [User, Scope]
purpose provide revocable, scoped, programmatic access on behalf of a user.
principle 
  A user generates a token, selecting a set of permissions (scopes). 
  The system provides a secret token string which is displayed only once and must be saved by the user. 
  An application can then use this string to perform actions limited by the token's scopes. 
  The user can revoke the token at any time, immediately invalidating it for all future use.

state
  a set of PersonalAccessTokens with
    an owner User
    a tokenHash String
    a set of Scopes

actions
  generate (owner: User, scopes: set of Scope): (token: String)
    requires the owner User exists
    effects creates a new PersonalAccessToken with the given owner and scopes. Generates a unique, secret token string, stores a cryptographic hash of that string in the tokenHash field, and returns the original plaintext token string to the user.

  authenticate (token: String): (user: User, scopes: set of Scope)
    requires a PersonalAccessToken exists whose tokenHash matches the cryptographic hash of the provided token string.
    effects returns the owner User and the set of Scopes associated with the matching token.

  revoke (token: PersonalAccessToken)
    requires the token exists
    effects deletes the PersonalAccessToken.
```

How the PersonalAccessToken and PasswordAuthentication concepts differ:

The difference between the PasswordAuthentication and PersonalAccessToken concepts lies in their purpose and trust model. PasswordAuthentication is designed for interactive use, granting a user full access to their account with a single, memorized secret. In contrast, PersonalAccessToken is designed for programmatic use by applications, granting limited and specific access on a user's behalf.

This core difference in purpose leads to several key distinctions:
- Multiplicity and Revocation: A user has only one password, which must be reset entirely. They can, however, generate many distinct Personal Access Tokens, each of which can be individually revoked without affecting the others. This allows for granular control over which applications have access.
- Scope: A password typically grants the full permissions of the user account. A token is defined by a set of scopes, limiting its capabilities to only what is necessary.
- Form and Storage: Passwords are designed to be memorable for humans. Tokens are long, random strings not meant for memorization, and must be stored securely by the application or script using them.


Could the GitHub page be improved, and if so how?

Yes, the documentation could be significantly improved by clarifying the core purpose of tokens at the very beginning. The current phrase "an alternative to using passwords" is misleading because it suggests they are interchangeable. The key confusion is between identity and capability.

A more effective explanation would be: "Passwords are for you. Tokens are for your apps." A password proves your identity for full, interactive account access. A Personal Access Token is a secure key you give to an application or script to perform specific, limited tasks on your behalf - a key you can revoke at any time without affecting your password.



## Exercise 4: Defining familiar Concepts

1. URL Shortener

```
concept URLShortener [User]
purpose to create short, memorable, and shareable aliases for long URLs.
principle 
  A user provides a long URL and can optionally suggest a custom short alias. 
  The service generates a unique short URL that, when accessed, redirects the user to the original long URL.

state
  a set of ShortURLs with
    an alias String
    an originalURL String
    an optional owner User

actions
  create (originalURL: String, customAlias: optional String, owner: optional User): (shortURL: ShortURL)
    requires if a customAlias is provided, that alias is not already in use.
    effects creates a new ShortURL. If a customAlias is provided, it is used as the alias. Otherwise, a new, unique, short alias is generated. The new ShortURL is returned.

  delete (alias: String, owner: User)
    requires a ShortURL with the given alias exists and its owner matches the provided owner.
    effects deletes the ShortURL.
```

Notes:

- The core invariant is the uniqueness of the alias. The create action must guarantee this.

- The owner is marked as optional to allow for both anonymous and registered user-created URLs. The delete action, however, requires an owner, making it a feature for registered users.

2. Billable Hours Tracking

```
concept BillableHoursTracking [Employee, Project]
purpose to allow employees to accurately record time spent on specific client projects for billing purposes.
principle
  An employee starts a work session by selecting a project and describing the task. 
  When they finish, they end the session, and the system records the duration. 
  If an employee forgets to end a session, they or a manager can manually edit the session later to add the correct end time.

state
  a set of WorkSessions with
    an employee Employee
    a project Project
    a description String
    a startTime DateTime
    an optional endTime DateTime

actions
  startSession (employee: Employee, project: Project, description: String): (session: WorkSession)
    requires the employee does not already have an active session (one where endTime is not set).
    effects creates a new WorkSession with the given details, using the current time as startTime, and returns it. The endTime is left unset.

  endSession (employee: Employee)
    requires the employee has an active session.
    effects finds the active session for the employee and sets its endTime to the current time.

  editSession (session: WorkSession, newStartTime: DateTime, newEndTime: DateTime)
    requires newStartTime is before newEndTime.
    effects updates the startTime and endTime for the specified session.
```

Notes:
- The key design decision is using an optional endTime to model both active (no end time) and completed (has an end time) sessions in a single data structure.
- The invariant that an employee can only have one active session at a time is crucial for the logic of startSession and endSession.
- The editSession action explicitly addresses the problem of an employee forgetting to clock out. It provides a direct mechanism to correct the data after the fact.


3. Conference Room Booking

```
concept ConferenceRoomBooking [User]
purpose to enable users to find and reserve available conference rooms for specific time slots.
principle
  A user queries the system to find rooms that are available for a desired time period.
  From the list of available rooms, the user selects one and books it for their meeting.
  The user who created the booking can also cancel it later.

state
  a set of ConferenceRooms with
    a name String
    a capacity Number

  a set of Bookings with
    a bookedBy User
    a room ConferenceRoom
    a startTime DateTime
    a endTime DateTime
    a title String

actions
  findAvailableRooms (startTime: DateTime, endTime: DateTime, requiredCapacity: Number): (rooms: set of ConferenceRoom)
    requires startTime is before endTime.
    effects returns the set of ConferenceRooms that meet the capacity requirement and have no conflicting Bookings in the given time frame.

  bookRoom (user: User, room: ConferenceRoom, startTime: DateTime, endTime: DateTime, title: String): (booking: Booking)
    requires startTime is before endTime.
    requires the specified room has no bookings that conflict with the given time frame.
    effects creates a new Booking with the provided details and returns it.

  cancelBooking (user: User, booking: Booking)
    requires the booking exists and the 'bookedBy' user of the booking matches the provided user.
    effects deletes the booking.
```

Notes:
- The central complexity, which is abstracted in the requires clauses, is the logic for detecting time overlaps between bookings for the same room. It ensures that there are no booking conflicts.
