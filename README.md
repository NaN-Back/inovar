import os
import zipfile

# Nome do projeto e ZIP
project_name = "plpe-mvp-docker"
zip_filename = f"{project_name}.zip"

# Função para criar arquivos e pastas
def create_file(path, content=""):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(content)

# -----------------------------
# Backend
# -----------------------------
create_file(f"{project_name}/backend/requirements.txt",
"""fastapi
uvicorn
pydantic
numpy
psycopg2-binary
reportlab
""")

create_file(f"{project_name}/backend/simulation.py",
"""import numpy as np

def generate_life_vector(user):
    career_score = min(user['years_experience']/20, 1)
    risk_tolerance = user['risk_score']
    financial_resilience = min(user['financial_reserve']/50000, 1)
    mobility_index = 1 if user['country_target'] != user['country_current'] else 0.5
    entrepreneurial_fit = risk_tolerance*0.7 + financial_resilience*0.3
    return {
        "career_score": career_score,
        "risk_tolerance": risk_tolerance,
        "financial_resilience": financial_resilience,
        "mobility_index": mobility_index,
        "entrepreneurial_fit": entrepreneurial_fit
    }

def simulate_decision(vector, decision_type):
    n_sim = 5000
    results = []
    for _ in range(n_sim):
        if decision_type == 'career':
            growth = np.random.normal(vector['career_score'], 0.1)
            financial = growth*10000 + vector['financial_resilience']*5000
            emotional = growth*0.8
        elif decision_type == 'country_change':
            growth = vector['mobility_index']*np.random.normal(1,0.2)
            financial = growth*8000 + vector['financial_resilience']*4000
            emotional = growth*0.9
        elif decision_type == 'entrepreneurship':
            growth = vector['entrepreneurial_fit']*np.random.normal(1,0.3)
            financial = growth*15000
            emotional = growth*0.7
        else:
            financial = 0
            emotional = 0
        results.append({"financial": financial, "emotional": emotional})
    return results

def calculate_eai(simulations):
    avg_financial = np.mean([s['financial'] for s in simulations])
    avg_emotional = np.mean([s['emotional'] for s in simulations])
    eai = 0.6*(avg_financial/10000) + 0.4*avg_emotional
    return round(eai,2)
""")

create_file(f"{project_name}/backend/pdf_report.py",
"""from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

def generate_pdf_report(user, decision_type, vector, eai, sims, filename="report.pdf"):
    c = canvas.Canvas(filename, pagesize=letter)
    c.setFont("Helvetica", 12)
    c.drawString(50, 750, f"PLPE Report - Decision Type: {decision_type}")
    c.drawString(50, 730, f"Name: {user['name']}, Age: {user['age']}")
    c.drawString(50, 710, f"EAI (Existential Alignment Index): {eai}")
    c.drawString(50, 690, "Life Vector:")
    y = 670
    for k,v in vector.items():
        c.drawString(60, y, f"{k}: {v}")
        y -= 20
    c.drawString(50, y-10, "Sample Simulations (10):")
    for s in sims[:10]:
        c.drawString(60, y-30, f"Financial: {s['financial']:.2f}, Emotional: {s['emotional']:.2f}")
        y -= 20
    c.save()
    return filename
""")

create_file(f"{project_name}/backend/main.py",
"""from fastapi import FastAPI
from pydantic import BaseModel
from simulation import generate_life_vector, simulate_decision, calculate_eai
from pdf_report import generate_pdf_report

app = FastAPI()

class UserInput(BaseModel):
    name: str
    age: int
    country_current: str
    country_target: str
    profession: str
    years_experience: int
    salary_current: float
    education_level: str
    risk_score: float
    financial_reserve: float
    dependents: int

@app.post("/simulate")
def simulate(user: UserInput, decision_type: str):
    vector = generate_life_vector(user.dict())
    sims = simulate_decision(vector, decision_type)
    eai = calculate_eai(sims)
    pdf_file = generate_pdf_report(user.dict(), decision_type, vector, eai, sims)
    return {"vector": vector, "eai": eai, "simulations": sims[:10], "pdf": pdf_file}
""")

create_file(f"{project_name}/backend/Dockerfile",
"""FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
""")

# -----------------------------
# Frontend
# -----------------------------
create_file(f"{project_name}/frontend/package.json",
"""{
  "name": "plpe-frontend",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "latest",
    "react": "latest",
    "react-dom": "latest"
  }
}
""")

create_file(f"{project_name}/frontend/pages/index.js",
"""import { useState } from "react";

export default function Home() {
  const [form, setForm] = useState({name:"", age:25, country_current:"", country_target:"", profession:"", years_experience:0, salary_current:0, education_level:"", risk_score:0.5, financial_reserve:0, dependents:0});
  const [result, setResult] = useState(null);
  const [decision, setDecision] = useState("career");

  const handleSubmit = async e => {
    e.preventDefault();
    const res = await fetch(`http://localhost:8000/simulate?decision_type=${decision}`, {
      method:"POST",
      headers:{"Content-Type":"application/json"},
      body:JSON.stringify(form)
    });
    const data = await res.json();
    setResult(data);
  };

  return (
    <div style={{padding:"20px"}}>
      <h1>PLPE MVP – Life Simulation Intelligence</h1>
      <select onChange={e=>setDecision(e.target.value)}>
        <option value="career">Career</option>
        <option value="country_change">Country Change</option>
        <option value="entrepreneurship">Entrepreneurship</option>
      </select>
      <form onSubmit={handleSubmit} style={{marginTop:"10px"}}>
        <input placeholder="Name" onChange={e=>setForm({...form,name:e.target.value})} />
        <input placeholder="Age" type="number" onChange={e=>setForm({...form,age:e.target.value})} />
        <input placeholder="Current Country" onChange={e=>setForm({...form,country_current:e.target.value})} />
        <input placeholder="Target Country" onChange={e=>setForm({...form,country_target:e.target.value})} />
        <input placeholder="Profession" onChange={e=>setForm({...form,profession:e.target.value})} />
        <button type="submit" style={{marginTop:"10px"}}>Simular</button>
      </form>

      {result && (
        <div style={{marginTop:"20px"}}>
          <h2>Resultado</h2>
          <pre>{JSON.stringify(result.vector,null,2)}</pre>
          <p>EAI: {result.eai}</p>
          <a href={result.pdf} target="_blank" rel="noreferrer">Baixar PDF</a>
        </div>
      )}
    </div>
  );
}
""")

create_file(f"{project_name}/frontend/styles/globals.css", "/* globals.css placeholder */\n")
create_file(f"{project_name}/frontend/Dockerfile",
"""FROM node:20

WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]
""")

# -----------------------------
# DB
# -----------------------------
create_file(f"{project_name}/db/create_tables.sql",
"""CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT,
    email TEXT UNIQUE,
    age INT,
    country_current TEXT,
    country_target TEXT,
    profession TEXT,
    years_experience INT,
    salary_current NUMERIC,
    education_level TEXT,
    risk_score NUMERIC,
    financial_reserve NUMERIC,
    dependents INT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE questionnaire (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    question TEXT,
    answer NUMERIC,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE simulations (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    decision_type TEXT,
    score_alignment NUMERIC,
    score_risk NUMERIC,
    projected_financial NUMERIC,
    projected_emotional NUMERIC,
    created_at TIMESTAMP DEFAULT NOW()
);
""")

# -----------------------------
# Docker Compose
# -----------------------------
create_file(f"{project_name}/docker-compose.yml",
"""version: "3.9"

services:
  db:
    image: postgres:15
    container_name: plpe_db
    environment:
      POSTGRES_USER: plpe_user
      POSTGRES_PASSWORD: plpe_pass
      POSTGRES_DB: plpe_db
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./db/create_tables.sql:/docker-entrypoint-initdb.d/create_tables.sql

  backend:
    build: ./backend
    container_name: plpe_backend
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_USER: plpe_user
      DB_PASS: plpe_pass
      DB_NAME: plpe_db

  frontend:
    build: ./frontend
    container_name: plpe_frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  db_data:
""")

# -----------------------------
# README
# -----------------------------
create_file(f"{project_name}/README.md",
"""# PLPE MVP Dockerizado

## Como rodar

1. Instale Docker e Docker Compose
2. No terminal, rode:
   docker-compose up --build
3. Acesse:
   Frontend: http://localhost:3000
   Backend API: http://localhost:8000/simulate
""")

# -----------------------------
# Criar ZIP
# -----------------------------
with zipfile.ZipFile(zip_filename, 'w', zipfile.ZIP_DEFLATED) as zipf:
    for root, dirs, files in os.walk(project_name):
        for file in files:
            filepath = os.path.join(root, file)
            zipf.write(filepath, os.path.relpath(filepath, project_name))

print(f"✅ ZIP do PLPE MVP Dockerizado criado com sucesso: {zip_filename}")
