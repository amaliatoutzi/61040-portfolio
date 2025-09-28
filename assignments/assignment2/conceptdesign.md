# Concept design


## Concept specifications

* ### concept MuseumVisit [User, Museum, Exhibit]

```plaintext
purpose
capture a user’s visit to a museum, including an overall museum tag and the list of exhibits seen, each with optional note/photo and a share setting

principle
when a user logs a museum visit and records the exhibits they saw (with optional notes/photos and a visibility setting), the visit becomes an editable diary entry owned by that user.

a set of Logs with
  a owner User
  a museum Museum
  a museumTag of LOVE or LIKE or MEH or NONE
  a visibility of PRIVATE or PUBLIC
  a createdAt DateTime
  an updatedAt DateTime

a set of LogExhibits with
  a log Logs
  an exhibit Exhibit
  a stars of STARS_1 or STARS_2 or STARS_3 or STARS_4 or STARS_5
  an optional note String
  an optional photoUrl String
  a loggedAt DateTime
  a updatedAt DateTime

createLog(owner: User, museum: Museum, visibility: PRIVATE|PUBLIC) : Logs
    effects if a Logs for (owner, museum) exists, do nothing (return it);
    otherwise create a new Logs with owner, museum, visibility := visibility, createdAt := now, updatedAt := now

setMuseumTag(log: Logs, tag: LOVE|LIKE|MEH|NONE, user: User)
    requires log exists and user = log.owner
    effects set log.museumTag := tag; set log.updatedAt := now.

clearMuseumTag(log: Logs, user: User)
    requires log exists and user = log.owner
    effects set log.museumTag := NONE; set log.updatedAt := now.

addExhibit(log: Logs, exhibit: Exhibit, stars: STARS_1..STARS_5, optional note: String, optional photoUrl: String, user: User) : LogExhibits
    requires log exists and user = log.owner and exhibit.museum = log.museum
    effects create a LogExhibits with given fields; set loggedAt := now, updatedAt := now; set log.updatedAt := now.

editExhibit(entry: LogExhibits, stars?: STARS_1..STARS_5, note?: String, photoUrl?: String, user: User)
    requires entry exists and user = entry.log.owner
    effects update provided fields; set entry.updatedAt := now; set entry.log.updatedAt := now.

removeExhibit(entry: LogExhibits, user: User)
    requires entry exists and caller is entry.log.owner
    effects delete the entry; set entry.log.updatedAt := now.

setLogVisibility(log: Logs, visibility: PRIVATE|PUBLIC, user: User)
    requires log exists and user = log.owner
    effects set log.visibility := visibility; set log.updatedAt := now.

deleteLog(log: Logs, user: User)
    requires log exists and user = log.owner
    effects remove the log and all associated LogExhibits.
```
* ### concept Profile [User, Museum, Exhibit, Tag]

```plaintext
purpose
maintain a user’s public identity, favorites, and preset taste tags to power recommendations and social features

principle
users must make a profile to use the app; it acts as authentication but also identification; as users complete their profile (handle, bio, favorites, preset tags), their identity and tastes become visible to friends and usable by the recommendation system. Following links determine whose shared logs appear in a user’s feed; if a profile is PRIVATE, only followers can see that user’s feed.

state
a set of Profiles with
    an owner User
    a username String
    a password String
    an optional bio String
    a set faveMuseums of favorite museums (Museum)
    a set faveExhibits of favorite exhibits (Exhibit)
    a set preferenceTags of Tag
    a visibility of PUBLIC or PRIVATE

a set of PresetTags with
    a tag Tag // e.g., Impressionist, Modern, Photography, Sculpture, Science

a set of Follows with
    a follower User
    a followee User

actions
createProfile (owner: User, username: String, password: String, visibility: PUBLIC|PRIVATE)
    requires no profile exists for owner and username is unique
    effects create profile with profile.visibility := visibility, profile.password := password, and empty sets
    for faveMuseums, preferenceTags, and faveExhibits

updateBio (owner: User, bio: String)
    requires profile exists with owner
    effects set profile.bio := bio

editProfileVisibility (owner: User, visibility: PUBLIC|PRIVATE)
    requires owner is profile owner
    effects profile.visibility := visibility

addPreferenceTag (owner: User, tag: Tag)
    requires owner exists, tag in PresetTags, and tag not in profile.preferenceTags
    effects profile.preferenceTags := profile.preferenceTags ∪ {tag}

removePreferenceTag (owner: User, tag: Tag)
    requires owner exists and tag in profile.preferenceTags
    effects profile.preferenceTags := profile.preferenceTags \ {tag}

addFaveMuseum (owner: User, m: Museum)
    requires owner exists, m not in profile.faveMuseums
    effects profile.faveMuseums := profile.faveMuseums ∪ {m}

removeFaveMuseum (owner: User, m: Museum)
    requires owner exists and m in profile.faveMuseums
    effects profile.faveMuseums := profile.faveMuseums \ {m}

addFaveExhibit (owner: User, e: Exhibit)
    requires owner exists and e not in profile.faveExhibits
    effects profile.faveExhibits := profile.faveExhibits ∪ {e}

removeFaveExhibit (owner: User, e: Exhibit)
    requires owner exists and e in profile.faveExhibits
    effects profile.faveExhibits := profile.faveExhibits \ {e}

follow (follower: User, followee: User)
    requires follower and followee exist, follower ≠ followee and follower does not follow followee
    effects follower adds ("follows") followee

unfollow (follower: User, followee: User)
    requires follower and followee exist, follower follows followee
    effects follower no longer adds ("follows") the followee
```


* ### concept Spotlight [User, Museum]
```plaintext
purpose
recommend museums to a user before a visit based on their taste signals

principle
after a user records a few museum tastes (LOVE/LIKE/MEH), the system can return a ranked list of other museums aligned with those tastes; as new signals arrive, future recommendations adapt.

state
a set of TasteSignals with
  a user User
  a museum Museum
  a taste of LOVE or LIKE or MEH
  an updatedAt DateTime

a set of SimilarityLinks with
  a from Museum
  a to Museum
  a similarity Number // higher means more similar
  an updatedAt DateTime

actions
recordMuseumTaste (user: User, museum: Museum, taste: LOVE|LIKE|MEH)
    requires user exists
    effects upsert TasteSignals(user, museum) with taste; set updatedAt := now

clearMuseumTaste (user: User, museum: Museum)
    requires user and TasteSignals(user, museum) exist
    effects delete that TasteSignals

recommendMuseums (user: User, k: Number) : List<Museum>
    requires k ≥ 1
    effects return up to k museums ranked by a black-box recommender that combines the user’s TasteSignals with SimilarityLinks;

system
rebuildSimilarityIndex ()
  effects recompute SimilarityLinks from aggregate community behavior and/or content features;
```

* ### concept Curations [User, Museum, Exhibit]
```plaintext
purpose
when a museum is selected, suggest exhibits within a selected museum based on user preferences

principle
once a user selects a museum to explore, the system returns a ranked list of exhibits in that museum based on aggregate exhibit signals and (when available) the user’s past signals; as new signals are recorded, future suggestions adapt.

state
  a set of ExhibitStars with
    a user User
    a museum Museum
    an exhibit Exhibit
    a stars of STARS_1 | STARS_2 | STARS_3 | STARS_4 | STARS_5
    an updatedAt DateTime

  a set of ExhibitSimilarityLinks with
    a museum Museum
    a from Exhibit
    a to Exhibit
    a similarity Number
    an updatedAt DateTime

actions
recordExhibitStars (user: User, museum: Museum, exhibit: Exhibit, stars: STARS_1..STARS_5)
    requires user exists and exhibit belongs to museum
    effects upsert ExhibitStars(user, museum, exhibit) with stars; set updatedAt := now

clearExhibitStars (user: User, museum: Museum, exhibit: Exhibit)
    requires user exists and ExhibitStars(user, museum, exhibit) exists
    effects delete that ExhibitStars row

recommendExhibits (user: User, museum: Museum, k: Number) : List<Exhibit>
    requires k ≥ 1
    effects return up to k exhibits from museum ranked by a black-box recommender
           that combines community ExhibitStars and (if present) the user’s own ExhibitStars

system
rebuildExhibitSimilarityIndex (museum: Museum)
    effects recompute ExhibitSimilarityLinks for museum from aggregate behavior and/or content features

```

* ### concept MuseumRegistry [Museum, Exhibit]
```plaintext
purpose
  maintain the canonical catalog of museums and exhibits; a museum can be registered only with a valid invite code issued outside the app, and thereafter managed by a single admin key

principle
  an out-of-band invite code authorizes the one-time creation (or claim) of a museum; once registered, presenting the museum’s admin key authorizes edits. each exhibit belongs to exactly one museum; only PUBLIC exhibits are visible to users.

state
a set of InviteCodes with
    a codeHash Hash                 // hash(inviteCode) issued externally
    a name String                   // intended museum name
    a location String               // intended location
    an expiresAt DateTime
    an optional usedAt DateTime     // null if unused

a set of Museums with
    an id MuseumId
    a name String
    a location String              // e.g., city / region
    an optional shortDescription String
    a keyHash Hash
    a createdAt DateTime
    an updatedAt DateTime

a set of Exhibits with
    an id ExhibitId
    a museum MuseumId              // exhibits belong to exactly one museum
    a name String
    an optional shortDescription String
    a visibility of PUBLIC or HIDDEN
    a createdAt DateTime
    a updatedAt DateTime

actions
registerMuseum (inviteCode: String, adminKey: String, name: String, location: String, shortDescription?: String) : MuseumId
    requires InviteCodes[hash(inviteCode)] exists, usedAt is null, now < expiresAt and no existing Museum with (name, location)
    effects create museum with name, location, shortDescription,keyHash := hash(adminKey), createdAt := now, updatedAt := now; also set InviteCodes[hash(inviteCode)].usedAt := now

editMuseum (museum: MuseumId, adminKey: String, name?: String, location?: String, shortDescription?: String)
    requires Museums[museum] exists and hash(adminKey) = Museums[museum].keyHash
    effects update provided fields; set updatedAt := now

addExhibit (museum: MuseumId, adminKey: String, name: String, shortDescription?: String, visibility?: PUBLIC|HIDDEN) : ExhibitId
    requires Museums[museum] exists and hash(adminKey) = Museums[museum].keyHash
    effects create exhibit with museum := museum, name, shortDescription, visibility := visibility or HIDDEN, createdAt := now, updatedAt := now

editExhibit (exhibit: ExhibitId, adminKey: String, name?: String, shortDescription?: String, visibility?: PUBLIC|HIDDEN)
    requires Exhibits[exhibit] exists and hash(adminKey) = Museums[Exhibits[exhibit].museum].keyHash
    effects update provided fields; set updatedAt := now
```
## Essential synchronizations

```plaintext
sync enforceExhibitBelongsToMuseum
    when MuseumVisit.addExhibit (log, exhibit, stars, note?, photoUrl?, user)
    where MuseumRegistry: Exhibits[exhibit].museum = MuseumVisit:Logs[log].museum
    then MuseumVisit.addExhibit (log, exhibit, stars, note?, photoUrl?, user)

sync clampLogsOnProfilePrivate
    when Profile.editProfileVisibility (owner, PRIVATE)
    then MuseumVisit.setLogVisibility (each log by owner, PRIVATE)

sync forcePrivateIfProfilePrivate
    when MuseumVisit.createLog (owner, museum, visibility)
    where Profile: profile(owner).visibility = PRIVATE
    then MuseumVisit.createLog (owner, museum, PRIVATE)

sync tagFeedsSpotlight
    when MuseumVisit.setMuseumTag (log, tag, user)
    where tag in {LOVE, LIKE, MEH}
    then Spotlight.recordMuseumTaste (user, MuseumVisit:Logs[log].museum, tag)

sync starsOnAdd
    when MuseumVisit.addExhibit (log, exhibit, stars, note?, photoUrl?, user)
    then Curations.recordExhibitStars (user, MuseumVisit:Logs[log].museum, exhibit, stars)

sync starsOnEdit
    when MuseumVisit.editExhibit (entry, stars?, note?, photoUrl?, user)
    where MuseumVisit: stars? is provided
    then Curations.recordExhibitStars (user, entry.log.museum, entry.exhibit, stars?)

sync refreshSpotlightIndex
    when Spotlight.recordMuseumTaste (user, museum, taste)
    then Spotlight.rebuildSimilarityIndex ()

sync refreshCurationsIndex
    when Curations.recordExhibitStars (user, museum, exhibit, stars)
    then Curations.rebuildExhibitSimilarityIndex (museum)

sync requireProfileForVisit
    when MuseumVisit.createLog (owner, museum, visibility)
    where Profile: Profiles exists for owner
    then MuseumVisit.createLog (owner, museum, visibility)
```

## Notes

I have design 5 concepts:
1. Profile [User, Museum, Exhibit, Tag] is the identity and access layer. A user must have a Profile to use the app. Profile carries visibility (PUBLIC/PRIVATE), favorites, and preset taste tags, and defines follow relationships. Profile visibility governs who may view a user’s public artifacts (e.g., visits surfaced elsewhere). Authentication is simplified by storing username/password inside Profile; no other concept stores credentials.

2. MuseumRegistry [Museum, Exhibit] is the source of truth for museums and exhibits. Registration requires an out-of-band inviteCod (so for example, an email is sent to museums inviting them to join the app); subsequent edits require the museum’s adminKey. This acts as authorization for catalog management only (it does not grant user-level privileges). Registry also controls exhibit visibility (PUBLIC/HIDDEN), which downstream concepts respect.

3. MuseumVisit [User, Museum, Exhibit] records a user’s visit (one log per user-museum) and per-exhibit stars/notes/photos with a per-log visibility (PUBLIC/PRIVATE). It is the only place users create “experience data.”

4. Spotlight [User, Museum] recommends museums pre-visit from users’ museum-level tastes; it treats the algorithm as a black box over TasteSignals and SimilarityLinks.

5. Curations [User, Museum, Exhibit] recommends exhibits within a chosen museum from ExhibitStars and museum-scoped ExhibitSimilarityLinks.

- Extra things to note: Profile visibility can clamp newly created visits to PRIVATE (via a sync). MuseumRegistry’s adminKey gates all catalog edits (create/edit museum & exhibits, set exhibit visibility). Viewing of user content is governed by the visit’s visibility.

- As for generic type instantiations:
In Spotlight, the target type is Museum (recommendations return List<Museum>).
In Curations, recommendations target Exhibit, but are scoped by Museum (recommendExhibits(user, museum, k)).
In MuseumVisit, LogExhibits.exhibit must satisfy the invariant exhibit.museum = log.museum.
Tag in Profile is instantiated from the global PresetTags catalog (e.g., Impressionist, Photography).
