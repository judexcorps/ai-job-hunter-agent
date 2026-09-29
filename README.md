# AI Job Hunter Agent

An automated workflow that finds new job openings every day, scores each one against my CV using AI, and emails me the best matches with a draft cover letter.

## The problem
Searching job boards by hand every day is slow and repetitive. This workflow does it automatically.

## How it works
1. A daily schedule triggers the workflow
2. It fetches new jobs from free job feeds
3. Google Gemini AI compares each job to my CV and gives a match score
4. Results are saved to Google Sheets
5. The top matches are emailed to me through Gmail, with a draft cover letter

## Tools used (all free)
- n8n (Community Edition)
- Google Gemini API (free tier)
- Google Sheets
- Gmail
- Free public job feeds

## Setup
1. Install Node.js from nodejs.org
2. Run `npx n8n` in your terminal and open the link it shows
3. Import the file `workflow.json` from this repository
4. Add your own Gemini API key, and connect your Google account
5. Paste your CV text into the CV node
6. Activate the workflow

## Screenshots
(Add screenshots of the workflow here)

## Author
Jude Ughakpoteni
Email: runotenijude@gmail.com
LinkedIn: https://linkedin.com/in/jude-ughakpoteni-337393413
