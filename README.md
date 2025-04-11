# Travel-Agent-AI
# travel_agent_ai/main.py

from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import openai
import requests
import os

app = FastAPI()

# Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Environment variables
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")
SERPAPI_KEY = os.getenv("SERPAPI_KEY")

# Models
class TravelQuery(BaseModel):
    user_message: str
    user_id: str

# Handlers
@app.post("/chat")
async def chat(query: TravelQuery):
    user_message = query.user_message
    user_id = query.user_id

    # 1. Get Gemini response (idea generator)
    gemini_response = get_gemini_response(user_message)

    # 2. Get SerpAPI travel data
    serp_results = get_serp_data(user_message)

    # 3. Merge & format with ChatGPT
    final_response = format_itinerary_with_chatgpt(user_message, gemini_response, serp_results)

    return {"reply": final_response}


def get_gemini_response(prompt: str) -> str:
    url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key={GEMINI_API_KEY}"
    payload = {"contents": [{"parts": [{"text": prompt}]}]}
    response = requests.post(url, json=payload)
    data = response.json()
    return data['candidates'][0]['content']['parts'][0]['text']


def get_serp_data(query: str) -> str:
    params = {
        "q": query,
        "location": "India",
        "api_key": SERPAPI_KEY,
        "engine": "google"
    }
    serp_url = "https://serpapi.com/search"
    response = requests.get(serp_url, params=params)
    results = response.json()
    top_results = results.get("organic_results", [])[:3]
    return "\n".join([r.get("title", "") + ": " + r.get("link", "") for r in top_results])


def format_itinerary_with_chatgpt(prompt: str, gemini_output: str, serp_data: str) -> str:
    openai.api_key = OPENAI_API_KEY
    system_prompt = f"You are a friendly travel planner. Use the following Gemini insights and web data to help the user:\n\nGemini Output:\n{gemini_output}\n\nSERPAPI Results:\n{serp_data}"
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": prompt}
    ]
    response = openai.ChatCompletion.create(model="gpt-4", messages=messages)
    return response.choices[0].message['content']
