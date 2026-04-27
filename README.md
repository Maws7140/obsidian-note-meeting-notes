# Meeting notes

Obsidian Meeting Notes dashboard

## Description

A productivity system for managing meeting recordings and tracking upcoming tasks in Obsidian.

## Features

### Upcoming Events Widget
- Displays tasks from specified sources in a calendar-style view
- Groups tasks by date (Today, Overdue, Future)
- Color-coded by priority (pink=high, orange=medium, blue=low)
- Project/school tags displayed on each task
- Click to open task source

### Audio Recording Manager
- **Record** - Capture audio directly (requires Audio Recorder plugin)
- **Search** - Find recordings by name
- **Sort** - By date, name, or size (ascending/descending)
- **Filter** - By audio format (MP3, WAV, M4A, WEBM, OGG)
- **One-click Notes** - Create meeting notes linked to recordings
- **Month grouping** - Collapsible month sections

## Prerequisites

- **Obsidian** (Desktop recommended)
- **Dataview** plugin
- **Datacore** plugin (for JSX support)
- **Audio Recorder** plugin (optional, for in-app recording)

## Setting task source
```javascript
const SOURCES = ['"TaskNotes/Tasks"', '"School"'];
```

## Task Properties

For tasks to appear in the upcoming events widget, use these frontmatter fields:

```yaml
---
title: Task title
status: pending    # Use "done", "completed", or "cancelled" to hide
priority: high    # high, medium, or low
due: 2024-01-20  # or: scheduled, start
projects: ProjectName
class: ClassName
---
```

## Folder Structure

```
/
├── Meeting notes.md    # Copy this to your vault
├── Meeting notes/      # Auto-created for meeting note files
└── [recordings]       # Audio files go here
```
## Configuration

### Upcoming Events Widget

| Variable | Description | Default |
|----------|-------------|---------|
| `DAYS_AHEAD` | Days to look ahead | `14` |
| `SOURCES` | Task folder paths | `['"TaskNotes/Tasks"', '"School"']` |
| `PRIORITY_COLORS` | Priority → color map | high: pink, medium: orange, low: blue |

### Audio Manager

Search, sort, and filter controls are accessible via toolbar buttons. All audio formats in your vault are detected automatically.

## Required Plugins

| Plugin | Version | Link |
|--------|---------|------|
| Datacore | 0.1.29 | [GitHub](https://github.com/search?q=Datacore%20obsidian) |
| Dataview | 0.5.68 | [GitHub](https://github.com/search?q=Dataview%20obsidian) |

## Attached Snippets

- `snippets/audio-manager.css` — audio-manager
- `snippets/upcoming-events.css` — upcoming-events

## Screenshots

![Screenshot 1](<img width="2880" height="1920" alt="image" src="https://github.com/user-attachments/assets/f8117bf1-696b-4fee-a190-9c0b2cb770d1" />
)
## Installation

1. Download the `.md` file(s) from this repo
2. Place them in your vault
3. Copy the attached CSS snippet files into `.obsidian/snippets/`
4. Enable them in Settings > Appearance > CSS Snippets
5. Install the required plugins listed above

---
*Published via [Vault Hub](https://obsidianvaulthub.com)*
