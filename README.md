# CineLog

A community film tracking app. Users log films they've watched, rate them, and build collections.

This repository is the starting point for **Project 6: Simulated Code Review**.

---

## Setup

```bash
pip install -r requirements.txt
python app.py
```

The app starts on `http://localhost:5000` and uses a local SQLite database (`cinelog.db`).

---

## Project Structure

```
ai201-project6-cinelog-starter/
├── app.py                     # Flask app factory
├── models.py                  # SQLAlchemy models
├── services/
│   └── collection_service.py  # Business logic for collections
├── routes/
│   ├── films.py               # Film browsing endpoints
│   └── collection.py          # Collection endpoints
├── tests/
│   └── test_collection.py     # Tests for collection service
├── CONTRIBUTING.md            # Commit conventions and PR guidelines
└── requirements.txt
```

<details>
<summary><strong>Architecture diagram</strong> (optional study aid — not required for submission)</summary>

```mermaid
flowchart TD
    Repo["ai201-project6-cinelog-starter/"]
    Repo --> App["app.py<br/>Flask app factory"]
    Repo --> Models["models.py<br/>SQLAlchemy models"]
    Repo --> Routes["routes/"]
    Repo --> Services["services/"]
    Repo --> Tests["tests/"]
    Repo --> Docs["README.md"]
    Repo --> Contrib["CONTRIBUTING.md"]
    Repo --> Req["requirements.txt"]
    Routes --> FilmsRoute["films.py<br/>Film browsing endpoints"]
    Routes --> CollectionRoute["collection.py<br/>Collection endpoints"]
    Services --> CollectionService["collection_service.py<br/>Business logic for collections"]
    Tests --> CollectionTests["test_collection.py<br/>Collection service tests"]
    Models --> User["User"]
    Models --> Film["Film"]
    Models --> Entry["CollectionEntry"]
```

</details>

---

## API Overview

### Films

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/films/` | List all films (supports `?genre=` and `?year=` filters) |
| GET | `/films/<film_id>` | Get a single film by UUID |

### Collection

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/collection/<user_id>` | Get a user's collection (newest first) |
| POST | `/collection/<user_id>/add` | Add a film to the collection |
| DELETE | `/collection/<user_id>/remove` | Remove a film from the collection |

<details>
<summary><strong>Request flow diagram</strong> (optional study aid)</summary>

```mermaid
flowchart LR
    Client["Client / Browser / API Tool"]
    Client --> FilmAPI["GET /films/<br/>GET /films/{film_id}"]
    Client --> CollectionAPI["GET /collection/{user_id}<br/>POST /collection/{user_id}/add<br/>DELETE /collection/{user_id}/remove"]
    FilmAPI --> FilmsRoute["routes/films.py"]
    CollectionAPI --> CollectionRoute["routes/collection.py"]
    FilmsRoute --> FilmModel["Film model"]
    CollectionRoute --> CollectionService["services/collection_service.py"]
    CollectionService --> FilmModel
    CollectionService --> EntryModel["CollectionEntry model"]
    FilmModel --> DB[("SQLite database cinelog.db")]
    EntryModel --> DB
```

Film routes are read-only. Collection routes delegate to `collection_service.py`.

</details>

<details>
<summary><strong>Sequence diagram: add film to collection</strong> (optional study aid)</summary>

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client / API Tool
    participant Route as routes/collection.py
    participant Service as services/collection_service.py
    participant Film as Film model
    participant Entry as CollectionEntry model
    participant DB as SQLite database
    Client->>Route: POST /collection/{user_id}/add
    Route->>Service: add_to_collection(user_id, film_id, rating)
    Service->>DB: Query Film by film_id
    DB-->>Service: Film found or not found
    alt Film does not exist
        Service-->>Route: raise FilmNotFoundError
        Route-->>Client: 404 Not Found
    else Film exists
        Service->>DB: Query CollectionEntry by user_id + film_id
        DB-->>Service: Existing entry or none
        alt Already in collection
            Service-->>Route: raise AlreadyInCollectionError
            Route-->>Client: 409 Conflict
        else Not yet in collection
            Service->>Entry: Create CollectionEntry
            Entry->>DB: Insert entry
            DB-->>Service: Commit success
            Service-->>Route: New collection entry
            Route-->>Client: 201 Created
        end
    end
```

</details>

---

## Data Models

**Film** — A film in the catalog. IDs are UUIDs.

**User** — A registered user. IDs are UUIDs.

**CollectionEntry** — Links a user to a film they've watched. Stores rating and date added. A user can only have one entry per film.

<details>
<summary><strong>Route → service → model diagram</strong> (optional study aid)</summary>

```mermaid
flowchart TD
    subgraph Routes["routes/"]
        ViewCollection["view_collection(user_id)<br/>GET /collection/{user_id}"]
        AddFilm["add_film(user_id)<br/>POST /collection/{user_id}/add"]
        RemoveFilm["remove_film(user_id)<br/>DELETE /collection/{user_id}/remove"]
        ListFilms["list_films()<br/>GET /films/"]
        GetFilm["get_film(film_id)<br/>GET /films/{film_id}"]
    end
    subgraph Services["services/collection_service.py"]
        GetCollection["get_collection(user_id)"]
        AddToCollection["add_to_collection(user_id, film_id, rating)"]
        RemoveFromCollection["remove_from_collection(user_id, film_id)"]
        FilmNotFound["FilmNotFoundError"]
        AlreadyInCollection["AlreadyInCollectionError"]
        NotInCollection["NotInCollectionError"]
    end
    subgraph Models["models.py"]
        User["User<br/>id, username, email"]
        Film["Film<br/>id, title, year, director, genre"]
        CollectionEntry["CollectionEntry<br/>user_id, film_id, rating, date_added"]
    end
    ViewCollection --> GetCollection
    AddFilm --> AddToCollection
    RemoveFilm --> RemoveFromCollection
    ListFilms --> Film
    GetFilm --> Film
    GetCollection --> CollectionEntry
    GetCollection --> Film
    AddToCollection --> Film
    AddToCollection --> CollectionEntry
    RemoveFromCollection --> CollectionEntry
    CollectionEntry --> User
    CollectionEntry --> Film
```

Routes handle HTTP, services handle business logic, models represent database tables.

</details>

---

## Naming Conventions

Service functions follow a `verb_to_noun` pattern. See `CONTRIBUTING.md` for full details.

---

## Running Tests

```bash
pytest tests/
```

<details>
<summary><strong>Test coverage map</strong> (optional study aid)</summary>

```mermaid
flowchart TD
    Tests["tests/test_collection.py"]
    Tests --> AddHappy["test_add_to_collection_creates_entry"]
    Tests --> AddDuplicate["test_add_to_collection_duplicate_raises"]
    Tests --> AddMissingFilm["test_add_to_collection_nonexistent_film_raises"]
    Tests --> SortOrder["test_get_collection_returns_newest_first"]
    AddHappy --> AddFunc["add_to_collection()"]
    AddDuplicate --> AddFunc
    AddMissingFilm --> AddFunc
    SortOrder --> GetFunc["get_collection()"]
    AddFunc --> FilmExists["Checks film exists"]
    AddFunc --> DuplicateCheck["Checks duplicate entry"]
    AddFunc --> CreateEntry["Creates CollectionEntry"]
    GetFunc --> QueryEntries["Queries CollectionEntry"]
    GetFunc --> NewestFirst["Sorts by date_added descending"]
    FilmExists --> FilmNotFound["FilmNotFoundError"]
    DuplicateCheck --> AlreadyInCollection["AlreadyInCollectionError"]
    Tests --> Fixtures["pytest fixtures"]
    Fixtures --> MemoryDB["In-memory SQLite database"]
    Fixtures --> SampleUser["sample_user"]
    Fixtures --> SampleFilm["sample_film"]
```

</details>

---

## Your Task

You're working on the `feature/watchlist` branch, which adds a watchlist feature to CineLog. A maintainer (`@dev-lead`) has reviewed your PR and left six comments. Your job is to address all six.

Read `CONTRIBUTING.md` before touching any code. Then check out the `feature/watchlist` branch:

```bash
git checkout feature/watchlist
```

The open PR and the maintainer's review comments are filed on GitHub. Work through each comment and document your responses in your **PR Response Doc**.

<details>
<summary><strong>Homework workflow</strong> (optional study aid)</summary>

```mermaid
stateDiagram-v2
    [*] --> ReadDocs
    ReadDocs --> CheckoutBranch
    CheckoutBranch --> ReadReviewComments
    ReadReviewComments --> ChooseComment
    ChooseComment --> ReproduceOrInspect
    ReproduceOrInspect --> TraceCode
    TraceCode --> IdentifyRootCause
    IdentifyRootCause --> ImplementFix
    ImplementFix --> AddOrUpdateTests
    AddOrUpdateTests --> RunTests
    RunTests --> CheckConventions
    CheckConventions --> CommitChange
    CommitChange --> UpdatePRResponse
    UpdatePRResponse --> MoreComments
    MoreComments --> ChooseComment: More review comments remain
    MoreComments --> FinalReview: All six addressed
    FinalReview --> Submit
    Submit --> [*]
    ReadDocs: Read README.md and CONTRIBUTING.md
    CheckoutBranch: git checkout feature/watchlist
    ReadReviewComments: Understand maintainer feedback
    TraceCode: route → service → model → test
    CheckConventions: Conventional commits, no merge commits
    CommitChange: One logical change per commit
    UpdatePRResponse: Explain what changed and why
```

</details>

<details>
<summary><strong>Bug-investigation workflow</strong> (optional study aid)</summary>

```mermaid
flowchart TD
    A["Start with maintainer review comment"] --> B["Identify affected feature"]
    B --> C{"Which layer is involved?"}
    C -->|Endpoint behavior| D["Inspect routes/*.py"]
    C -->|Business rule| E["Inspect services/collection_service.py"]
    C -->|Data shape / IDs / relationships| F["Inspect models.py"]
    C -->|Regression expectation| G["Inspect tests/test_collection.py"]
    C -->|Convention issue| H["Inspect CONTRIBUTING.md"]
    D --> I["Trace route inputs and response status"]
    E --> J["Trace service function behavior"]
    F --> K["Check model fields and UUID assumptions"]
    G --> L["Compare against existing test pattern"]
    H --> M["Check commit, naming, PR, and test rules"]
    I --> N["Reproduce the issue"]
    J --> N
    K --> N
    L --> N
    M --> N
    N --> O["Write minimal fix"]
    O --> P["Run pytest tests/"]
    P --> Q{"Tests pass and review comment addressed?"}
    Q -->|No| B
    Q -->|Yes| R["Commit one logical change"]
    R --> S["Document response in PR Response Doc"]
```

</details>
