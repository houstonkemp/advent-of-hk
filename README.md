# 🔐 Riddle Lock Challenge

A fun advent-style riddle challenge where participants solve word riddles to collect digits and unlock a physical combination lock!

## Overview

This system allows you to run a 3-week riddle challenge where:
- Participants solve riddles 3 times per week
- Each correct answer reveals one digit (0-9)
- After collecting 3 digits, they have the combination to unlock a physical lock
- The riddles and answers are stored securely in a backend database

## Components

### Frontend (This Repo)
- **index.html** - The main webpage hosted on GitHub Pages
- Beautiful, responsive design
- Shows active riddles
- Allows users to submit answers
- Displays collected digits in a combination lock display
- Stores progress locally in browser

### Backend (Separate Repo)
- R Plumber API hosted at `cardioid.co.nz/api`
- SQLite database for riddles
- Secure endpoints for submitting and activating riddles
- Public endpoints for fetching riddles and checking answers
- See `API_BACKEND_GUIDE.md` for full implementation details

## Setup Instructions

### 1. Frontend Setup (GitHub Pages)

1. Push this repository to GitHub
2. Enable GitHub Pages in repository settings:
   - Go to Settings > Pages
   - Source: Deploy from a branch
   - Branch: main (or your default branch)
   - Folder: / (root)
3. Your site will be available at: `https://[username].github.io/[repo-name]/`

### 2. Backend Setup

Follow the instructions in `API_BACKEND_GUIDE.md` to:
1. Add the required endpoints to your Plumber API
2. Update CORS configuration
3. Test the endpoints
4. Deploy to your server

### 3. Update API URL (if needed)

If your API is not at `cardioid.co.nz/api`, update line 138 in `index.html`:

```javascript
const API_BASE_URL = 'https://your-api-domain.com/api';
```

## Usage

### For Administrators

#### 1. Submit a New Riddle

```bash
curl -X POST "https://cardioid.co.nz/api/riddles/submit" \
  -H "Content-Type: application/json" \
  -d '{
    "riddle_text": "I speak without a mouth and hear without ears. What am I?",
    "answer": "echo",
    "digit": 7,
    "position": 1,
    "week": 1,
    "notes": "Week 1, Monday",
    "auth_code": "YOUR_AUTH_CODE"
  }'
```

This creates the riddle but keeps it inactive (not visible to users).

#### 2. Activate a Riddle

When you're ready to make it available to participants:

```bash
curl -X POST "https://cardioid.co.nz/api/riddles/activate" \
  -H "Content-Type: application/json" \
  -d '{
    "riddle_id": "the-riddle-id-from-submit",
    "auth_code": "YOUR_AUTH_CODE"
  }'
```

#### 3. View All Riddles

```bash
curl "https://cardioid.co.nz/api/riddles/all?auth_code=YOUR_AUTH_CODE"
```

### For Participants

1. Visit the website
2. Read the active riddles
3. Submit your answer
4. If correct, a digit is revealed
5. Collect all 3 digits to unlock the physical lock!

## Riddle Planning Guide

For a 3-week challenge with 3 riddles per week:

| Week | Day | Position | Digit | Notes |
|------|-----|----------|-------|-------|
| 1 | Mon | 1 | ? | First digit |
| 1 | Wed | 2 | ? | Second digit |
| 1 | Fri | 3 | ? | Third digit |
| 2 | Mon | 1 | ? | First digit (new set) |
| 2 | Wed | 2 | ? | Second digit (new set) |
| 2 | Fri | 3 | ? | Third digit (new set) |
| 3 | Mon | 1 | ? | First digit (final set) |
| 3 | Wed | 2 | ? | Second digit (final set) |
| 3 | Fri | 3 | ? | Third digit (final set) |

**Important:** Set your physical lock to match the digits for week 1 first. After week 1 is complete, you can:
- Reset the physical lock for week 2
- OR use a different lock for week 2
- OR keep adding to the same combination if you have a different reward system

## Example Riddles

### Easy
- "What has keys but no locks?" → **keyboard**
- "I have cities but no houses, mountains but no trees. What am I?" → **map**
- "The more you take, the more you leave behind. What am I?" → **footsteps**

### Medium
- "I speak without a mouth and hear without ears. What am I?" → **echo**
- "What can travel around the world while staying in a corner?" → **stamp**
- "I have a face and hands but no arms or legs. What am I?" → **clock**

### Hard
- "What word becomes shorter when you add two letters to it?" → **short**
- "Forward I am heavy, backward I am not. What am I?" → **ton**
- "I am always hungry and must always be fed. What am I?" → **fire**

## Features

### Frontend Features
- ✅ Responsive design (mobile-friendly)
- ✅ Beautiful gradient UI with lock theme
- ✅ Real-time combination display
- ✅ Progress saved in browser
- ✅ Clear success/error feedback
- ✅ Marks solved riddles
- ✅ Celebration when all digits collected

### Backend Features
- ✅ Secure authentication for admin endpoints
- ✅ SQLite database for persistence
- ✅ Case-insensitive answer matching
- ✅ Riddle activation system
- ✅ CORS support for cross-origin requests
- ✅ Error handling and validation

## Technical Details

- **Frontend:** Pure HTML/CSS/JavaScript (no build process needed)
- **Backend:** R Plumber API with SQLite
- **Storage:** LocalStorage for user progress, SQLite for riddles
- **Security:** Auth codes for admin endpoints, no answer exposure on public endpoints

## Troubleshooting

### Riddles not loading
- Check browser console for errors
- Verify API URL is correct in index.html
- Check CORS configuration in backend
- Test API endpoint directly: `curl https://cardioid.co.nz/api/riddles/active`

### Answers not working
- Verify riddle is activated in the backend
- Check answer spelling (system is case-insensitive)
- Check browser console for API errors

### Progress lost
- Progress is stored in browser's localStorage
- If user clears browser data, progress is lost
- Users can call `resetProgress()` from browser console to reset manually

## Future Enhancements

Possible additions:
- Leaderboard to track who completes first
- Hints system (costs points)
- Difficulty ratings
- Timer for each riddle
- Email notifications when new riddles are active
- Admin dashboard UI

## License

Personal project - feel free to adapt for your own use!

## Credits

Created for a fun workplace advent challenge featuring riddles and a physical combination lock with treats! 🎉🍬
