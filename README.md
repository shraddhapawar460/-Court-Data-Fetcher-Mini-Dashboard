# -Court-Data-Fetcher-Mini-Dashboard
Task 1: Court-Data Fetcher & Mini-Dashboard, using Flask + Playwright + SQLite. You can pick-and-choose bits, or use the whole thing as a starting point
1. Project structure
2. court-data-fetcher/
├── app.py
├── scraper.py
├── models.py
├── requirements.txt
├── templates/
│   └── index.html
└── README.md.
requirements.txt
txt
Copy code
Flask==2.3.2
playwright==1.35.0
SQLAlchemy==2.0.19
flask_sqlalchemy==3.0.4
After installing, run:

bash
Copy code
pip install -r requirements.txt
playwright install chromium
Database models (models.py)
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class QueryLog(db.Model):
    __tablename__ = 'query_log'
    id = db.Column(db.Integer, primary_key=True)
    court       = db.Column(db.String, nullable=False)
    case_type   = db.Column(db.String, nullable=False)
    case_number = db.Column(db.String, nullable=False)
    filing_year = db.Column(db.String, nullable=False)
    timestamp   = db.Column(db.DateTime, server_default=db.func.now())

class CaseData(db.Model):
    __tablename__ = 'case_data'
    id           = db.Column(db.Integer, primary_key=True)
    query_id     = db.Column(db.Integer, db.ForeignKey('query_log.id'))
    parties      = db.Column(db.String)
    next_hearing = db.Column(db.String)
    pdf_url      = db.Column(db.String)
    Scraper module (scraper.py)from playwright.sync_api import sync_playwright

def fetch_case(court_url, case_type, case_number, filing_year):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        # 1. Navigate
        page.goto(court_url)
        # 2. Fill form (example selectors—adjust per site)
        page.fill('select#caseType', case_type)
        page.fill('input#caseNo', case_number)
        page.fill('input#year', filing_year)
        page.click('button#search')
        page.wait_for_selector('.result-row')

        # 3. Parse
        parties = page.text_content('.result-row .parties')
        next_hearing = page.text_content('.result-row .next-hearing')
        pdf_link = page.get_attribute('.result-row a.last-order', 'href')

        browser.close()

    return {
        'parties': parties,
        'next_hearing': next_hearing,
        'pdf_url': pdf_link
    }
    Flask app (app.py)
from flask import Flask, render_template, request, flash, redirect, url_for
from models import db, QueryLog, CaseData
from scraper import fetch_case
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///cases.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.secret_key = os.environ.get('FLASK_SECRET', 'dev')

db.init_app(app)

@app.before_first_request
def init_db():
    db.create_all()

@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        court_url   = request.form['court_url']
        case_type   = request.form['case_type']
        case_number = request.form['case_number']
        filing_year = request.form['filing_year']

        try:
            data = fetch_case(court_url, case_type, case_number, filing_year)
        except Exception as e:
            flash(f"Error fetching case: {e}", 'danger')
            return redirect(url_for('index'))

        # log query
        q = QueryLog(court=court_url, case_type=case_type,
                     case_number=case_number, filing_year=filing_year)
        db.session.add(q); db.session.commit()

        # store data
        cd = CaseData(query_id=q.id, parties=data['parties'],
                      next_hearing=data['next_hearing'], pdf_url=data['pdf_url'])
        db.session.add(cd); db.session.commit()

        return render_template('index.html', result=data)

    return render_template('index.html')

if __name__ == '__main__':
    app.run(debug=True)
    Simple template (templates/index.html)
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Court Data Fetcher</title>
  <style>
    body { font-family: sans-serif; padding:2rem; }
    .result { margin-top:1.5rem; border:1px solid #ccc; padding:1rem; }
  </style>
</head>
<body>
  <h1>Court Data Fetcher</h1>
  {% with messages = get_flashed_messages(with_categories=true) %}
    {% for category, msg in messages %}
      <div class="flash {{ category }}">{{ msg }}</div>
    {% endfor %}
  {% endwith %}

  <form method="post">
    <label>Court URL:     <input name="court_url"   required></label><br>
    <label>Case Type:     <input name="case_type"   required></label><br>
    <label>Case Number:   <input name="case_number" required></label><br>
    <label>Filing Year:   <input name="filing_year" required></label><br>
    <button type="submit">Fetch Case</button>
  </form>

  {% if result %}
    <div class="result">
      <h2>Result</h2>
      <p><strong>Parties:</strong> {{ result.parties }}</p>
      <p><strong>Next Hearing:</strong> {{ result.next_hearing }}</p>
      <p><a href="{{ result.pdf_url }}" target="_blank">Download Latest PDF</a></p>
    </div>
  {% endif %}
</body>
</html>
README.md (highlights)

# Court-Data Fetcher & Mini-Dashboard

## Overview
Allows users to enter court URL, case type/number/year, scrapes the public court site via Playwright, and displays parties, next-hearing date, and PDF link.

## Setup
1. `git clone …`
2. `pip install -r requirements.txt`
3. `playwright install`
4. `export FLASK_SECRET=…`
5. `flask run`

## CAPTCHA strategy
If you hit a CAPTCHA, you can:
- Integrate a remote CAPTCHA-solving API
- Fallback to manual token entry in a hidden form field

## Extras (optional)
- Dockerfile
- Pagination if multiple orders
- Unit tests with pytest + GitHub Actions
-
