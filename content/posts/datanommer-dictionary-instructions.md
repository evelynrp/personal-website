---
date: '2026-04-18'
title: 'How to Add to the Datanommer Dictionary'
---

## Intro

Hatlas is a data lakehouse project built to help the Fedora Project leadership measure community health. Whether people are joining and staying, which contribution types are growing or shrinking, and whether onboarding is effective have been historically very hard to measure.

Hatlas ingests data from Fedora's existing systems and makes it queryable in one place (Apache Iceberg + Polaris). Datanommer is the historical log of everything that has ever happened on Fedora's internal message bus.

Fedora's message bus is like an internal event log that has been running for years. Every time something happens in Fedora - a user account is created, a package is published, a forum post is made, someone is added to a group - a message gets sent on the bus, and Datanommer records it.

However, there are over 28,000 distinct message types, called "topics", in this database. Each one has its own JSON structure. The field that means "username" in one message type might be called `username`, `agent`, `user`, `msg.webhook_body.post.username`, or something else entirely in another.

Nobody has yet created a comprehensive map of what all these messages mean. The Dictionary should systematically map this data in a format that automated pipelines can later use to normalize this data into consistent, analyzable tables.

## Background

Datanommer is a service that listens to **Fedora Messaging** (a RabbitMQ-based message bus) and writes every message to a PostgreSQL database. It's the historical record of all automated and user-initiated activity across Fedora's infrastructure.

The main table you need to know about is `public.messages`, which has these key columns:

| Column | Type | Notes |
|---|---|---|
| `id` | long | Primary key |
| `msg_id` | string | Supposed to be a UUID; early records have year prefix |
| `topic` | string | Identifies the message type (e.g., `org.fedoraproject.prod.fas.user.create`) |
| `timestamp` | timestamptz | When the message was sent |
| `category` | string | Used for Datagrepper queries |
| `agent_name` | string | Sometimes has the acting username; sometimes empty |
| `msg` | string | The actual message payload — topic-specific JSON |
| `headers` | string | JSON metadata including the `fedora_messaging_schema` field |

There are also join tables linking messages to `users` and `packages`.

**Topics** are like message categories. An application is assigned a topic, and it publishes all its messages there. Sub-topics allow further specificity. For example:

```
org.fedoraproject.prod.discourse              ← all Discourse messages
org.fedoraproject.prod.discourse.post.*      ← all Discourse post messages
org.fedoraproject.prod.discourse.post.post_created  ← only post creation events
```

**Schemas** (capital S) are a versioning layer *within* a topic. The schema name lives in `headers.fedora_messaging_schema`. When an application changes its message structure in a breaking way, it bumps the Schema name (e.g., `discourse.event.v1` → `discourse.event.v2`), while the topic may stay the same.

So the combination of **topic + schema name** is what uniquely identifies a particular structure of message. That's exactly how the dictionary is keyed.

> ⚠️ **Important caveat:** Schema versioning discipline has not always been maintained. You may find cases where the same topic+schema combination has different structures across different time ranges. When you discover this, document it in the dictionary and in the topic's example files.

### The Problem(s)

**Problem 1 — Schema diversity.** There are ~1,000 Fedora-relevant topics, each with its own JSON structure. Finding even simple fields like "username" requires reading individual message samples, one topic at a time.

**Problem 2 — The logical layer.** What does "username" actually mean? Is it the person who took the action, or the person the action was taken *on*? For example, when a FAS group sponsor adds a member, there are *two* users in the message: the sponsor (acting user) and the new member (acted-upon user). We need to record both, and distinguish them by role.

**Problem 3 — Activity taxonomy.** Not all contributions are equal, and a flat count of "events" doesn't tell us much. We need to categorize contributions so we can say things like "forum activity is up 20% but code commits are down 10%." This requires a taxonomy, a hierarchy of contribution types, that we're building as we go.

## The Data Dictionary

The dictionary is a single file, `datanommer-dictionary.cue`, that describes, for each topic+schema combination, what the message means and how to extract useful data from it using JSON paths.

It serves three purposes simultaneously:
1. **Human documentation** - a readable index of what each message type is and where the interesting fields live.
2. **Semantic layer** - maps raw field paths to our activity taxonomy, giving meaning to the raw data.
3. **Machine-readable input** - automated pipelines parse this to generate SQL, dbt transformations, Grafana graphs, etc.

### Structure

The file uses [Cuelang](https://cuelang.org/), which is like JSON with schema validation and templating built in. Think of it as "JSON that enforces its own rules and has variables." It renders to plain JSON via:

```bash
cue export datanommer-dictionary.cue > datanommer-dictionary.json
```

The schema structures at the top of the file define what's valid, and the entries at the bottom are where new topics are added.

### Anatomy of a Dictionary Entry

Here is a complete, annotated example entry:

```cue
{
  // The full topic string from `public.messages.topic`
  "topic": "org.fedoraproject.prod.discourse.post.post_created",

  "header_schemas": [
    {
      // The schema name from `headers.fedora_messaging_schema`
      "fedora_schema_name": "discourse.event.v1",

      // Does datanommer link this message to users in users_messages?
      "users_messages": true,

      // "out" = what we can extract from this message
      "out": {
        "user_activities": [
          {
            // Which taxonomy node does this map to?
            "taxonomy": "ua.community.forum.post_created",

            // Where in the message JSON is the acting user's identifier?
            "userid_ref": {
              "jsonpath": "$.msg.webhook_body.post.username"
              // users_field defaults to "fas_username"
            },

            // Non-PII data worth keeping alongside this activity record
            "extras": {
              "category_id": {"jsonpath": "$.msg.webhook_body.post.category_id"}
            },

            // Useful but potentially identifying data — restricted access
            "priv_extras": {
              "topic_id":  {"jsonpath": "$.msg.webhook_body.post.topic_id"},
              "post_url":  {"jsonpath": "$.msg.webhook_body.post.post_url"}
            }
          }
        ]
      }
    }
  ]
}
```

Key things to understand about this structure:

- A single topic can have **multiple schema versions** in `header_schemas`.
- A single schema can produce **multiple user activity records** from one message (e.g., the sponsor action AND the access-granted action are both recorded from the same FAS group sponsor message).
- `userid_ref.jsonpath` must point to a scalar value (the username/email/etc.), not an object.
- `userid_ref.users_field` tells us what *kind* of identifier it is. It defaults to `fas_username`. Other valid values: `email`, `github_username`, `gitlab_username`, `matrix_username`, `irc_username`.
- JSONPath uses `$.msg...` to reference into the `msg` column, or you can override `column` to point to another column (like `headers`).

### Activity Taxonomy

The taxonomy is a dot-separated hierarchy (using Postgres `ltree` style). Current top-level categories:

| Category | Meaning |
|---|---|
| `ua.code` | Code-based contributions (commits, PRs, reviews, etc.) |
| `ua.community` | People-to-people activity (forum posts, events, meetings) |
| `ua.admin` | Management/organizational tasks (group membership, account management) |

**Currently defined taxonomy nodes:**

| ID | Description |
|---|---|
| `ua.community.forum.post_created` | Created a forum post |
| `ua.admin.access_added` | Was granted privileges |
| `ua.admin.access_removed` | Had privileges revoked |
| `ua.admin.joined` | Created their account |
| `ua.admin.managed_group` | Managed group membership |
| `ua.admin.updated_profile` | Updated their user profile |

**Taxonomy design principles:**

- Be abstract, not too specific. Prefer `ua.code.commit` over `ua.code.commit.github.bugfix`.
- Use `extras` for the specifics that don't belong in the taxonomy node itself.
- Think in terms of: "How would we query this at a summary level?" and "How would we drill down?"

If you encounter an activity type that doesn't fit any existing taxonomy node, propose a new one.

### Extras vs. Privileged Extras (PII)

When you document a topic, you'll identify fields beyond the username that might be useful. These go into `extras` or `priv_extras`:

**`extras`** - data that is useful for analysis and is *not* PII. This should be safe to expose in aggregated or public reports. Example: the forum category ID of a post.

**`priv_extras`** - data that is useful but *could* be used to identify an individual or link activity back to them. This goes into a restricted-access column. Example: a post URL or topic ID (which could be cross-referenced to find who wrote what).

**The rule of thumb:** If storing this field in the analytics table would make it possible to trace the record back to a specific person's real-world action, it belongs in `priv_extras`, not `extras`.

Do **not** store anything in `extras` that ties the record back to a user. When in doubt, use `priv_extras` or omit it and leave a comment for the team.

## Documenting Topics

Use `fedora-topics.csv` to pick an undocumented topic. Good starting criteria:

- Topics under `org.fedoraproject.prod.*` (not `org.centos.*` — CentOS topics are intentionally excluded for now)
- Topics that appear frequently in the data (higher volume = more impactful to document)
- Topics that look like clear user actions based on the topic name (e.g., anything with `.user.`, `.group.`, `.post.`, `.comment.`)
- Avoid webhook topics and anything under `org.fedoraproject.prod.github.*` or `org.fedoraproject.prod.gitlab.*` until the basic/clean cases are covered

**Find a sample message for your chosen topic.** Use this query against Datanommer (or the Hatlas dataset):

```sql
SELECT
  m.id, m.msg, m.headers, m.agent_name, m.category,
  um.user_id,
  u.name AS user_name
FROM messages m
LEFT JOIN users_messages um ON um.msg_id = m.id
LEFT JOIN users u ON u.id = um.user_id
WHERE m.timestamp >= '2026-01-01'::date
  AND m.topic = 'org.fedoraproject.prod.YOUR_TOPIC_HERE'
LIMIT 3;
```

Save the sample message JSON to the `topics/<topic-name>/` directory with the filename matching the `public.messages.id` value. Save both the `msg` column and the `headers` column as separate files:

```
topics/org.fedoraproject.prod.foo.bar/
  12345678-msg.json
  12345678-headers.json
```

Check the `headers` JSON for the `fedora_messaging_schema` field. This is your `fedora_schema_name`.

### Writing a Dictionary Entry

Add the new entry to the `"topics"` array in `datanommer-dictionary.cue`.

**Use this template:**

```cue
{
  "topic": "org.fedoraproject.prod.YOUR.TOPIC.HERE",
  "header_schemas": [
    {
      "fedora_schema_name": "YOUR_SCHEMA_NAME_FROM_HEADERS",
      "users_messages": true,   // set to false if this message has no user association
      "out": {
        "user_activities": [
          {
            "taxonomy": _taxonomy.ua.YOUR.TAXONOMY.NODE.id,
            "userid_ref": {
              "jsonpath": "$.msg.PATH.TO.USERNAME.FIELD"
              // add users_field here if it's not a FAS username
            },
            "extras": {
              // non-PII fields worth keeping
              "field_name": {"jsonpath": "$.msg.path.to.field"},
              // or hardcoded values:
              "action": {"static": "some_action_label"},
            },
            "priv_extras": {
              // potentially identifying fields
            }
          }
        ]
      }
    }
  ]
}
```

**Checklist before you consider an entry complete:**

- [ ] `topic` matches the exact string from `messages.topic`
- [ ] `fedora_schema_name` matches the exact string from `headers.fedora_messaging_schema`
- [ ] `users_messages` is set correctly (true if the message has a user association in datanommer's join table)
- [ ] `userid_ref.jsonpath` points to a real field in the sample message that contains a username or email
- [ ] `userid_ref.users_field` is set if the identifier is not a FAS username
- [ ] Each distinct user role in the message has its own `user_activities` entry
- [ ] `extras` contains only non-PII fields
- [ ] `priv_extras` contains any potentially identifying fields
- [ ] `cue export datanommer-dictionary.cue` runs without errors
- [ ] At least one sample message is saved in the `topics/` directory

### Extend the Taxonomy if Needed

If you encounter an activity type that doesn't fit any current taxonomy node, propose a new one. Add it to the `_taxonomy` object near the top of `datanommer-dictionary.cue`.

The format follows this pattern:

```cue
_taxonomy: {
  ua: {
    code: {
      commit: {"id": "ua.code.commit", "desc": "committed code"}
    }
  }
}
```

The `desc` field should complete the sentence: *"[username] [desc]"*.  
Example: `"mwinters committed code"` → desc is `"committed code"`.

When proposing a new taxonomy node, include a brief note in your PR explaining:
- Why none of the existing nodes fit
- What the new node represents
- Whether it should be a new top-level category or a sub-node of an existing one

The taxonomy is still very much in flux and intentionally kept abstract. When in doubt, go broader rather than more specific. Use `extras` for the specifics.

### Submit Your Work

1. Render and commit the updated JSON:
   ```bash
   cue export datanommer-dictionary.cue > datanommer-dictionary.json
   ```

2. Commit both `datanommer-dictionary.cue` and `datanommer-dictionary.json`, plus any files in `topics/`.

3. Update `fedora-topics.csv` to mark your topic(s) as documented.

4. Open a pull request or share your branch in the `#data:fedoraproject.org` Matrix channel for review.

## Reference

### Key Files and Links

| Resource | Location |
|---|---|
| Data dictionary (source) | `datanommer-dictionary.cue` |
| Data dictionary (rendered JSON) | `datanommer-dictionary.json` |
| Topic status tracker | `fedora-topics.csv` |
| Full topic list (text) | https://codeberg.org/fedora-mwinters/datanommer-ops/src/branch/main/docs/facts/fedora-topics.txt |
| Datanommer local setup | https://codeberg.org/fedora-mwinters/datanommer-ops |
| DB dumps (upstream) | https://infrastructure.fedoraproject.org/infra/db-dumps/ |
| Fedora Messaging schema docs | https://fedora-messaging.readthedocs.io/en/stable/user-guide/schemas.html |
| Legacy fedmsg data dictionary | https://github.com/fedora-infra/fedmsg_meta_fedora_infrastructure |
| Cuelang docs | https://cuelang.org/ |
| Hatlas project homepage | (internal — ask in Matrix) |
| Matrix channel | `#data:fedoraproject.org` |
| Fedora Discussions | https://discussion.fedoraproject.org/ (tag: `#commops`) |

### Schema Quick Reference

#### Top-level structure

```
#Dict
└── topics[]
    └── #Topic
        ├── topic (string)
        └── header_schemas[]
            └── #HeaderSchema
                ├── fedora_schema_name (string)
                ├── users_messages (bool, default false)
                ├── packages_messages (bool, default false)
                └── out?
                    └── user_activities[]
                        └── #UserActivity
                            ├── taxonomy (string)
                            ├── userid_ref (#UserIdentifierFieldRef)
                            ├── extras? (map of #MessageFieldRef | #HardcodedScalar)
                            └── priv_extras? (map of #MessageFieldRef | #HardcodedScalar)
```

#### UserIdentifierFieldRef — valid `users_field` values

| Value | Meaning |
|---|---|
| `fas_username` | FAS (Fedora Account System) username — **this is the default** |
| `email` | Email address |
| `github_username` | GitHub username |
| `gitlab_username` | GitLab username |
| `matrix_username` | Matrix username |
| `irc_username` | IRC username |

#### MessageFieldRef — structure

```cue
{
  column:   *"msg" | string  // which DB column; defaults to "msg"
  jsonpath: string           // JSONPath expression, e.g. "$.msg.user"
}
```

#### HardcodedScalar — structure

```cue
{
  static: string | int | bool  // a literal value to store
}
```

### Worked Examples

#### Example A: Single user, simple action

**Topic:** `org.fedoraproject.prod.fas.user.create`  
**What happens:** A new Fedora account is created.  
**Message structure (simplified):** `{"msg": {"user": "newusername"}}`

```cue
{
  "topic": "org.fedoraproject.prod.fas.user.create",
  "header_schemas": [
    {
      "fedora_schema_name": "noggin.user.create.v1",
      "users_messages": true,
      "out": {
        "user_activities": [
          {
            "taxonomy": _taxonomy.ua.admin.joined.id,
            "userid_ref": {
              "jsonpath": "$.msg.user"
            }
          }
        ]
      }
    }
  ]
}
```

---

#### Example B: Two users in one message

**Topic:** `org.fedoraproject.prod.fas.group.member.sponsor`  
**What happens:** A FAS group sponsor adds a member to a group. Two things happen: the sponsor acted, and the new member was granted access.

```cue
{
  "topic": "org.fedoraproject.prod.fas.group.member.sponsor",
  "header_schemas": [
    {
      "fedora_schema_name": "noggin.group.member.sponsor.v1",
      "users_messages": true,
      "out": {
        "user_activities": [
          // Activity 1: the sponsor managed the group
          {
            "taxonomy": _taxonomy.ua.admin.managed_group.id,
            "userid_ref": {
              "jsonpath": "$.msg.agent"   // ← the acting user
            },
            "extras": {
              "action":     {"static": "add_user"},
              "group_type": {"static": "fas"},
              "group_name": {"jsonpath": "$.msg.group"}
            }
          },
          // Activity 2: the new member was granted access
          {
            "taxonomy": _taxonomy.ua.admin.access_added.id,
            "userid_ref": {
              "jsonpath": "$.msg.user"    // ← the acted-upon user
            },
            "extras": {
              "group_type": {"static": "fas"},
              "group_name": {"jsonpath": "$.msg.group"}
            }
          }
        ]
      }
    }
  ]
}
```

---

#### Example C: Message with PII in priv_extras

**Topic:** `org.fedoraproject.prod.discourse.post.post_created`  
**What happens:** A user creates a post on Fedora Discussion (Discourse).  
The post's URL and topic ID could be used to identify the user's content, so they go in `priv_extras`.

```cue
{
  "extras": {
    "category_id": {"jsonpath": "$.msg.webhook_body.post.category_id"}
    // Safe: category ID is an aggregate grouping, not linked to a specific user
  },
  "priv_extras": {
    "topic_id": {"jsonpath": "$.msg.webhook_body.post.topic_id"},
    "post_url": {"jsonpath": "$.msg.webhook_body.post.post_url"}
    // Sensitive: these can be cross-referenced to find a specific person's post
  }
}
```

---

### 9. Edge Cases and Gotchas

#### CentOS topics — skip them
Topics beginning with `org.centos.*` are intentionally excluded from this project. CentOS has misused the topic/schema design pattern so severely (encoding package names directly into topic strings, creating ~29,000 topics) that analyzing their data would require a separate effort. Don't document them.

#### Webhook topics — defer for now
Some topics are "generic webhooks" where external services like GitHub or GitLab send unstructured data. These are problematic because:
- The message structure can change without any change to the topic or schema name
- Multiple different external services may share the same webhook topic
- We have no control over the schema

For now, **focus on "first-party" Fedora Messaging topics** (those from services Fedora controls: noggin/FAS, Discourse, etc.). Webhook topics will be handled later once we have the clean cases covered.

#### Schema versions that drift
The schema versioning system has not always been used correctly. You may find that messages on the same topic+schema combination have *different structures* depending on when they were sent. If you encounter this:
- Document both structures in the dictionary with a note
- Save samples of both variants in the `topics/` directory
- Add a `README.md` in that directory describing when each structure was used (if you can determine it)

#### The `agent_name` column
The `messages.agent_name` column was supposed to always contain the acting username, but it's unreliable — sometimes it's empty when the username only exists in the message body. Don't rely on it. Use `userid_ref.jsonpath` to point directly into `msg` instead.

#### msg_id UUIDs with year prefix
Early Datanommer messages have a year prepended to the UUID in `msg_id` (e.g., `2014-xxxxxxxx-xxxx-...`). This is a known data quality issue being addressed in the Silver tier upgrade. It should not affect your dictionary work, but be aware that sample messages from early years may look different.

#### `users_messages` vs. documenting the userid
These are two separate concerns:
- `users_messages: true` means Datanommer itself has already populated the `users_messages` join table for this topic (this was done at ingest time).
- `userid_ref` in your dictionary entry tells the *analytics pipeline* where to find the username in the raw message, so it can build the `analytics.user_activity` table.

Both are needed. Set `users_messages` based on whether the join table is actually populated, not based on whether the message has a username field.

### Data Usage Guidelines

By contributing to Hatlas and working with this data, you agree to the Fedora Data Usage Guidelines. The core principles:

**Community, Not Individuals.** Metrics focus on community health, not evaluating individual contributors. The analytics pipeline is deliberately designed to make aggregate analysis easy and individual surveillance hard.

**Privacy and Respect.** PII (personally identifiable information) is used only when required for aggregation or deduplication, and is never displayed directly. This is why we have `extras` vs. `priv_extras`.

**Ethical Use of Data.** This data must never be used to measure individual productivity, enforce policy, or make participation decisions about specific people.

**Transparency.** All transformations and methods are openly documented. The data dictionary *is* that documentation.

These are not just soft guidelines — they are the reason the project is structured the way it is. The separation of `extras` and `priv_extras`, the anonymization design of the analytics tables, and the avoidance of individual-level reporting are all direct implementations of these principles. Keep them in mind when deciding what to put in `extras` vs. `priv_extras`, and when proposing new taxonomy nodes or output fields.

### Getting Help

**Primary channel:** Matrix — `#data:fedoraproject.org`  
If you've never used Matrix, you'll need a Fedora Accounts login. Setup guide: https://fedoraproject.org/wiki/Matrix

**Secondary:** Fedora Discussions (Discourse) — tag your posts with `#commops`

**Primary contact:** mwinters (`@mwinters:fedora.im`)

**When to ask for help:**
- You've found a topic whose message structure doesn't match any of the existing patterns
- You're unsure whether a field should be `extras` or `priv_extras`
- You want to propose a new taxonomy node but aren't sure where it fits
- The `cue export` command is giving you errors you can't decipher
- You've found evidence that a schema version has drifted over time

Don't spin your wheels for more than an hour on any of these — just ask. The whole point of the Matrix channel is to coordinate exactly this kind of knowledge-sharing.

---

*This document was compiled from the Hatlas project documentation, the Datanommer Data Dictionary README, the Datanommer dataset page, Matrix working group discussion logs, and the Hatlas project blog post. If anything here conflicts with the source files, the source files take precedence — and please flag the discrepancy so this document can be updated.*

## Revision History

- 2026-04-18: Original posting.