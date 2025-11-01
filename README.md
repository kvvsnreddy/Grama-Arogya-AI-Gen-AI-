Grama Arogya+ — Repo Skeleton
This document contains a ready-to-use repository skeleton (README + code files) for the Grama Arogya+ AI Buildathon project. Copy files into your repository and run the app locally using the instructions below.
________________________________________
File: README.md
# 🌿 Grama Arogya+ AI
AI-Powered Rural Health & Wellness Companion — Healthcare in every villager’s pocket.

## Quick start
1. Clone repository
```bash
git clone <your-repo-url>
cd grama-arogya-plus
2.	Create virtual env and install
python -m venv venv
source venv/bin/activate  # mac/linux
venv\Scripts\activate     # windows
pip install -r requirements.txt
3.	Set OpenAI key (needed for optional enhanced responses)
export OPENAI_API_KEY="your_key"   # mac/linux
setx OPENAI_API_KEY "your_key"     # windows
4.	Run Streamlit UI
streamlit run app.py
What is included
•	app.py — Streamlit frontend & integration
•	utils.py — health scoring + helper functions
•	ml_model.py — placeholder for TFJS/Py model or rule-based risk predictor
•	requirements.txt
•	README.md — this file
Notes
•	This repo uses OpenAI APIs optionally for richer conversational guidance. The app works offline with rule-based scoring if no key is provided.
•	Replace placeholders with your deployment URLs and dataset paths.
Team
Veera Venkata Satyanarayana Reddy Kovvuri — Project Owner

---

## File: requirements.txt

```text
streamlit
openai
numpy
pandas
python-dotenv
pydantic
langchain
pyngrok
(Install these with pip install -r requirements.txt)
________________________________________
File: utils.py
import math
from typing import Dict

def compute_bmi(weight_kg: float, height_cm: float) -> float:
    if not weight_kg or not height_cm:
        return None
    h = height_cm / 100.0
    return round(weight_kg / (h*h), 1)


def simple_health_score(age: int, bp_sys: int, bp_dia: int, sugar: float, pulse: int, bmi: float) -> Dict:
    """Return a dict with score (0-100) and risk_level (green/yellow/red) using simple heuristics.
    This is intentionally conservative and meant as an MVP rule-based predictor.
    """
    score = 100

    # Age penalty
    if age and age > 50:
        score -= min(10, (age-50) * 0.5)

    # Blood pressure
    if bp_sys and bp_dia:
        if bp_sys >= 180 or bp_dia >= 120:
            score -= 40
        elif bp_sys >= 140 or bp_dia >= 90:
            score -= 20
        elif bp_sys >= 130 or bp_dia >= 85:
            score -= 10

    # Sugar (fasting approx)
    if sugar:
        if sugar >= 300:
            score -= 35
        elif sugar >= 200:
            score -= 25
        elif sugar >= 140:
            score -= 10

    # Pulse
    if pulse:
        if pulse < 40 or pulse > 120:
            score -= 20
        elif pulse > 100:
            score -= 10

    # BMI
    if bmi:
        if bmi >= 35 or bmi < 16:
            score -= 20
        elif bmi >= 30 or bmi < 18.5:
            score -= 10

    # clamp
    score = max(0, min(100, int(score)))

    # risk level
    if score >= 75:
        level = 'green'
    elif score >= 45:
        level = 'yellow'
    else:
        level = 'red'

    return {'score': score, 'level': level}


def triage_message(level: str) -> str:
    if level == 'green':
        return 'Low risk. Advise home care, lifestyle changes, and routine follow-up.'
    if level == 'yellow':
        return 'Moderate risk. Recommend monitoring vitals closely and schedule a primary care visit.'
    return 'High risk. Seek urgent medical attention — contact nearest PHC or call emergency services.'
________________________________________
File: app.py (Streamlit)
import streamlit as st
import os
from utils import compute_bmi, simple_health_score, triage_message
import openai
from dotenv import load_dotenv

load_dotenv()
OPENAI_KEY = os.getenv('OPENAI_API_KEY')
if OPENAI_KEY:
    openai.api_key = OPENAI_KEY

st.set_page_config(page_title='Grama Arogya+', layout='wide')

st.markdown('# 🌿 Grama Arogya+ AI')
st.markdown('AI-Powered Rural Health Companion — voice-first & offline-capable MVP')

with st.sidebar:
    st.header('User / Device')
    name = st.text_input('Full name')
    age = st.number_input('Age', min_value=1, max_value=120, value=30)
    sex = st.selectbox('Sex', ['Male', 'Female', 'Other'])

st.header('Vitals & Symptoms')
col1, col2, col3 = st.columns(3)
with col1:
    weight = st.number_input('Weight (kg)', min_value=1.0, max_value=300.0, value=60.0)
    bp_sys = st.number_input('BP Systolic', min_value=50, max_value=300, value=120)
with col2:
    height = st.number_input('Height (cm)', min_value=50, max_value=250, value=165)
    bp_dia = st.number_input('BP Diastolic', min_value=30, max_value=200, value=80)
with col3:
    sugar = st.number_input('Sugar (mg/dL)', min_value=20.0, max_value=1000.0, value=90.0)
    pulse = st.number_input('Pulse (bpm)', min_value=20, max_value=200, value=72)

symptoms = st.text_area('Symptoms (describe in local language or English)', height=120)

if st.button('Analyze Health'):
    bmi = compute_bmi(weight, height)
    st.metric('BMI', bmi or '—')

    result = simple_health_score(age, bp_sys, bp_dia, sugar, pulse, bmi)
    st.subheader(f"Health Score: {result['score']} — {result['level'].upper()}")
    st.info(triage_message(result['level']))

    # Suggest simple actions
    if result['level'] == 'green':
        st.success('Suggested: Daily yoga, balanced millet-rich diet, hydration, monitor weekly')
    elif result['level'] == 'yellow':
        st.warning('Suggested: Reduce salt, schedule follow-up, record BP twice daily, consult local doctor')
    else:
        st.error('Suggested: Urgent consult. If breathing difficulty or chest pain — seek emergency help')

    # Optional: Use OpenAI for personalised advice if key is available
    if OPENAI_KEY:
        prompt = (
            f"You are a compassionate rural health assistant.\nAge:{age}, Sex:{sex}, BMI:{bmi}, "
            f"BP:{bp_sys}/{bp_dia}, Sugar:{sugar}, Pulse:{pulse}. Symptoms: {symptoms}. "
            f"Give a short, clear recommendation in 2-3 lines in simple language and include Do's and Don'ts."
        )
        try:
            resp = openai.ChatCompletion.create(
                model='gpt-4o-mini',
                messages=[{'role':'user','content': prompt}],
                max_tokens=200,
                temperature=0.2,
            )
            text = resp['choices'][0]['message']['content']
            st.markdown('**AI Guidance (OpenAI)**')
            st.write(text)
        except Exception as e:
            st.write('OpenAI call failed:', e)

    # Store result locally (simple csv log)
    import pandas as pd
    from datetime import datetime
    log_row = {
        'timestamp': datetime.utcnow().isoformat(),
        'name': name,
        'age': age,
        'sex': sex,
        'bmi': bmi,
        'bp_sys': bp_sys,
        'bp_dia': bp_dia,
        'sugar': sugar,
        'pulse': pulse,
        'score': result['score'],
        'level': result['level'],
        'symptoms': symptoms
    }
    logs_path = 'health_logs.csv'
    if os.path.exists(logs_path):
        df = pd.read_csv(logs_path)
        df = df.append(log_row, ignore_index=True)
    else:
        df = pd.DataFrame([log_row])
    df.to_csv(logs_path, index=False)
    st.success('Saved health log locally')

st.markdown('---')
st.markdown('## Quick Tools')
if st.button('Show last 5 logs'):
    import pandas as pd
    if os.path.exists('health_logs.csv'):
        df = pd.read_csv('health_logs.csv')
        st.dataframe(df.tail(5))
    else:
        st.info('No logs yet')
________________________________________
File: ml_model.py (placeholder)
# Placeholder for model training or TFJS integration.
# For Buildathon MVP we rely on rule-based `simple_health_score` in utils.py.

# Example: export a model with sample features and labels for later TFJS conversion.

import numpy as np
import pandas as pd

def train_dummy_model(df: pd.DataFrame):
    # implement later: train a small sklearn/tf model on curated dataset
    pass
________________________________________
Usage & Deployment
•	Local: streamlit run app.py
•	To deploy: Create a small VM on Render/Vercel/Heroku or deploy Streamlit Cloud.
•	For offline: Streamlit app works without OpenAI key; remove OpenAI call to ensure offline capability.
________________________________________
License
MIT
________________________________________
If you want, I can now: - Produce README.md as a separate file ready to commit - Generate a git commit message & git commands to push - Create a short 4-slide PPT (assets + speaker notes)
