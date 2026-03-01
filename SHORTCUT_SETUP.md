# iOS Notes → Website Sync: Shortcut Setup Guide

This guide walks you through setting up the iOS Shortcut that syncs notes from your Apple Notes "Guides" folder to your website.

## Prerequisites

- iPhone or iPad running iOS 16+
- The Shortcuts app (pre-installed)
- A "Guides" folder created in Apple Notes
- A GitHub account with access to the `tomnagengast.com` repo

## Step 1: Create a GitHub Personal Access Token

1. Go to **github.com/settings/tokens** → **Fine-grained tokens** → **Generate new token**
2. **Token name:** `ios-notes-sync`
3. **Expiration:** 1 year (set a calendar reminder to rotate)
4. **Repository access:** "Only select repositories" → select `tomnagengast/tomnagengast.com`
5. **Permissions:** Contents → **Read and write** (leave everything else as "No access")
6. Click **Generate token** and copy it

### Store the token on your device

Save the token to a text file on iCloud Drive so the Shortcut can read it without hardcoding:

1. Open the **Files** app on your iPhone
2. Navigate to **iCloud Drive** → **Shortcuts** (create this folder if it doesn't exist)
3. Create a new text file called `github-token.txt` containing only the token
4. Save it

This keeps the token out of the Shortcut itself, making it safe to share the Shortcut and easy to rotate the token later.

## Step 2: Create the "Guides" Folder in Apple Notes

1. Open **Notes** app
2. Tap **New Folder** → name it **Guides**
3. Write some test notes in this folder (these will become your guides)

## Step 3: Build the iOS Shortcut

Open the **Shortcuts** app and create a new Shortcut called **"Sync Guides"**.

Add the following actions in order:

---

### 3.1 — Load the GitHub token

| # | Action | Configuration |
|---|--------|---------------|
| 1 | **Text** | `tomnagengast` — tap the result and **Set Name** to `Owner` |
| 2 | **Text** | `tomnagengast.com` — set name to `Repo` |
| 3 | **Text** | `guides/guides.json` — set name to `FilePath` |
| 4 | **Get File** | File Path: `Shortcuts/github-token.txt`, Service: iCloud Drive. Set name to `GitHubToken` |

---

### 3.2 — Find all notes in the Guides folder

| # | Action | Configuration |
|---|--------|---------------|
| 5 | **Find Notes** | Add filter: **Folder is Guides**. Sort by: **Modification Date**, Order: **Latest First** |

Set the result name to `AllNotes`.

---

### 3.3 — Build the guides data

| # | Action | Configuration |
|---|--------|---------------|
| 6 | **Set Variable** | Variable Name: `GuidesList`, Value: (leave empty — this initializes the list) |
| 7 | **Repeat with Each** | Input: `AllNotes` |

**Inside the Repeat loop**, add these actions for each `Repeat Item`:

| # | Action | Configuration |
|---|--------|---------------|
| 8 | **Get Name of** | Input: `Repeat Item`. Set result name to `NoteTitle` |
| 9 | **Replace Text** | Input: `NoteTitle`. Find: `[^a-zA-Z0-9 -]` (enable **Regular Expression**). Replace with: (empty). Set result name to `CleanTitle` |
| 10 | **Replace Text** | Input: `CleanTitle`. Find: ` +` (enable **Regular Expression**). Replace with: `-`. Set result name to `SlugDraft` |
| 11 | **Replace Text** | Input: `SlugDraft`. Find: `--+` (enable **Regular Expression**). Replace with: `-`. Set result name to `SlugClean` |
| 12 | **Change Case** | Input: `SlugClean`. To: **lowercase**. Set result name to `Slug` |
| 13 | **Get Details of** | Input: `Repeat Item`. Get: **Body**. Set result name to `NoteBody` |
| 14 | **Make HTML from Rich Text** | Input: `NoteBody`. **Make Full Document**: OFF. Set result name to `BodyHTML` |
| 15 | **Get Details of** | Input: `Repeat Item`. Get: **Creation Date**. Set result name to `RawCreated` |
| 16 | **Format Date** | Input: `RawCreated`. Date Format: **Custom** → `yyyy-MM-dd'T'HH:mm:ssZZZZZ`. Set result name to `CreatedAt` |
| 17 | **Get Details of** | Input: `Repeat Item`. Get: **Modification Date**. Set result name to `RawUpdated` |
| 18 | **Format Date** | Input: `RawUpdated`. Date Format: **Custom** → `yyyy-MM-dd'T'HH:mm:ssZZZZZ`. Set result name to `UpdatedAt` |
| 19 | **Dictionary** | Create with these key-value pairs: |

Dictionary keys for action 19:
- `slug` → `Slug`
- `title` → `NoteTitle`
- `body` → `BodyHTML`
- `createdAt` → `CreatedAt`
- `updatedAt` → `UpdatedAt`

| # | Action | Configuration |
|---|--------|---------------|
| 20 | **Add to Variable** | Variable: `GuidesList`. Input: the Dictionary from action 19 |

**End Repeat**

---

### 3.4 — Build the final JSON

| # | Action | Configuration |
|---|--------|---------------|
| 21 | **Format Date** | Input: **Current Date**. Date Format: **Custom** → `yyyy-MM-dd'T'HH:mm:ssZZZZZ`. Set result name to `SyncTimestamp` |
| 22 | **Dictionary** | Create with: `lastSynced` → `SyncTimestamp`, `guides` → `GuidesList` |
| 23 | **Set Variable** | Variable: `PayloadDict`. Input: the Dictionary from action 22 |

---

### 3.5 — Get the current file SHA from GitHub

This is needed so GitHub knows which version of the file you're replacing.

| # | Action | Configuration |
|---|--------|---------------|
| 24 | **URL** | `https://api.github.com/repos/tomnagengast/tomnagengast.com/contents/guides/guides.json` |
| 25 | **Get Contents of URL** | Method: **GET**. Headers: `Authorization` → `Bearer [GitHubToken]`, `Accept` → `application/vnd.github+json` |
| 26 | **Get Dictionary Value** | Key: `sha`. Set result name to `CurrentSHA` |

> **Note:** On the very first sync, this will return a 404 error (the file doesn't exist on GitHub yet). That's OK — the Shortcut will still work because we handle this in the next step.

---

### 3.6 — Push to GitHub

| # | Action | Configuration |
|---|--------|---------------|
| 27 | **Get text from** | Input: `PayloadDict`. This converts the dictionary to a JSON string. Set result name to `JSONText` |
| 28 | **Base64 Encode** | Input: `JSONText`. Set result name to `EncodedContent` |
| 29 | **Replace Text** | Input: `EncodedContent`. Find: `\n` (enable Regular Expression). Replace with: (empty). Set result name to `CleanBase64` |
| 30 | **Dictionary** | Create with: `message` → `Sync guides from Apple Notes`, `content` → `CleanBase64` |
| 31 | **If** | Input: `CurrentSHA`. Condition: **has any value** |
| 32 | **Set Dictionary Value** | Dictionary: from action 30. Key: `sha`. Value: `CurrentSHA` |
| 33 | **End If** |  |
| 34 | **URL** | `https://api.github.com/repos/tomnagengast/tomnagengast.com/contents/guides/guides.json` |
| 35 | **Get Contents of URL** | URL: from action 34. Method: **PUT**. Headers: `Authorization` → `Bearer [GitHubToken]`, `Accept` → `application/vnd.github+json`. Request Body: **JSON** → the Dictionary from action 30 (after the If block) |

---

### 3.7 — Confirm success

| # | Action | Configuration |
|---|--------|---------------|
| 36 | **Count** | Input: `AllNotes`. Set result name to `GuideCount` |
| 37 | **Show Notification** | Title: "Guides Synced". Body: `Synced [GuideCount] guides to your website!` |

---

### 3.8 — Shortcut settings

Tap the **settings icon** (top right of the Shortcut editor):

1. Enable **"Allow Sharing Large Amounts of Data"** (under Privacy) — this prevents failures when syncing many notes
2. Optionally add the Shortcut to your Home Screen or set it as a Back Tap action for quick access

## Step 4: Test It

1. Make sure you have at least one note in your **Guides** folder in Apple Notes
2. Run the **Sync Guides** Shortcut
3. Check your GitHub repo — you should see `guides/guides.json` updated with your notes
4. Visit your website at `/guides/` — your guides should appear

## Troubleshooting

**Shortcut fails with an error on the GET request (step 25):**
This is normal on the first sync. The file doesn't exist yet, so GitHub returns 404. The If/Else in step 31-33 handles this by omitting the SHA.

**Guides appear but formatting looks wrong:**
Apple Notes HTML can include some unusual formatting. The site's CSS handles common elements (headings, paragraphs, lists, bold, italic, links, tables, blockquotes). If you find elements that don't render well, the `.guide-content` styles in `guides/index.html` can be extended.

**Title appears duplicated in the guide body:**
Apple Notes sometimes includes the title as the first line of the body. If this happens, add a **Replace Text** action after step 14 to strip the first `<h1>` tag: Find `^<h1[^>]*>.*?</h1>` (regex), Replace with empty.

**Token expired:**
GitHub fine-grained tokens expire after the duration you set. Generate a new one and update `iCloud Drive/Shortcuts/github-token.txt`.

## How It All Works

```
You write in Apple Notes "Guides" folder
                    |
                    v
Tap "Sync Guides" Shortcut (or say "Hey Siri, Sync Guides")
                    |
                    v
Shortcut reads all notes, builds JSON, pushes to GitHub
                    |
                    v
Netlify detects the push and auto-deploys
                    |
                    v
Your website at /guides/ shows the updated content
```
