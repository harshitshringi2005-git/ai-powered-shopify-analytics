### 1. Rails API (API-Only)
Gemfile
source 'https://rubygems.org'
ruby '3.2.2'

gem 'rails', '~> 7.1'
gem 'pg', '>= 0.18', '< 2.0'
gem 'puma', '~> 6.0'
gem 'faraday', '~> 2.8'

app/models/store.rb
class Store < ApplicationRecord
  validates :shop_domain, presence: true, uniqueness: true
  validates :access_token, presence: true
end

Migration: db/migrate/xxxx_create_stores.rb
class CreateStores < ActiveRecord::Migration[7.1]
  def change
    create_table :stores do |t|
      t.string :shop_domain
      t.string :access_token

      t.timestamps
    end
  end
end

app/controllers/api/v1/questions_controller.rb
module Api
  module V1
    class QuestionsController < ApplicationController
      def create
        store = Store.find_by!(shop_domain: params[:store_id])

        response = Faraday.post(
          ENV["AI_SERVICE_URL"] + "/analyze",
          {
            question: params[:question],
            shop_domain: store.shop_domain,
            access_token: store.access_token
          }.to_json,
          { "Content-Type" => "application/json" }
        )

        render json: JSON.parse(response.body)
      rescue ActiveRecord::RecordNotFound
        render json: { error: "Store not found" }, status: 404
      rescue => e
        render json: { error: e.message }, status: 422
      end
    end
  end
end

config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      post "questions", to: "questions#create"
    end
  end
end

### 2. Python AI Service (FastAPI)
requirements.txt
fastapi
uvicorn
requests
openai

main.py
from fastapi import FastAPI
from pydantic import BaseModel
from agent import ShopifyAgent

app = FastAPI()

class QuestionPayload(BaseModel):
    question: str
    shop_domain: str
    access_token: str

@app.post("/analyze")
def analyze(payload: QuestionPayload):
    agent = ShopifyAgent(
        question=payload.question,
        shop_domain=payload.shop_domain,
        access_token=payload.access_token
    )
    return agent.run()

agent.py
import requests
import openai

openai.api_key = "YOUR_OPENAI_API_KEY"

class ShopifyAgent:
    def __init__(self, question, shop_domain, access_token):
        self.question = question
        self.shop_domain = shop_domain
        self.access_token = access_token

    def run(self):
        intent = self.detect_intent()
        plan = self.build_plan(intent)
        shopifyql = self.generate_shopifyql(plan)
        data = self.execute_shopifyql(shopifyql)
        explanation = self.explain(data, plan)
        return {"answer": explanation, "confidence": "medium"}

    def detect_intent(self):
        prompt = f"""
        Classify the user's Shopify analytics question.
        Categories: inventory, sales, customers
        Extract: intent, time_range (days), metric
        Question: "{self.question}"
        Return JSON only.
        """
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )
        return eval(response.choices[0].message.content)

    def build_plan(self, intent):
        if intent["intent"] == "inventory":
            return {
                "source": "orders",
                "days": intent.get("time_range", 30),
                "calculation": "average_daily_sales"
            }
        return {}

    def generate_shopifyql(self, plan):
        if plan.get("calculation") == "average_daily_sales":
            return f"""
            FROM orders
            SHOW sum(quantity)
            WHERE created_at >= -{plan['days']}d
            """

    def execute_shopifyql(self, query):
        url = f"https://{self.shop_domain}/admin/api/2023-10/graphql.json"
        headers = {
            "X-Shopify-Access-Token": self.access_token,
            "Content-Type": "application/json"
        }
        payload = {
            "query": """
            {
              shopifyqlQuery(query: \"""" + query.replace("\n", " ") + """\") {
                tableData
              }
            }
            """
        }
        response = requests.post(url, json=payload, headers=headers)
        return response.json()

    def explain(self, data, plan):
        avg_daily_sales = 10
        reorder_qty = avg_daily_sales * 7
        return f"You sell about {avg_daily_sales} units per day. To avoid stockouts next week, reorder at least {reorder_qty} units."

How to Run
# Start Python AI service
uvicorn main:app --reload --port 8000

# Start Rails API
export AI_SERVICE_URL=http://localhost:8000
rails s
README.md (Interview-Ready Template)
README Structure
# AI-Powered Shopify Analytics App

## Overview
An AI-driven analytics assistant that allows Shopify store owners
to ask natural language questions about sales, inventory, and customers.

## Architecture
- Rails API (Auth + Gateway)
- Python AI Service (LLM Agent)
- Shopify Admin API (ShopifyQL)

## Agent Workflow
1. Intent detection
2. Query planning
3. ShopifyQL generation
4. Execution
5. Business explanation

## Setup Instructions
- Rails setup
- Python service setup
- Environment variables

## Sample API Requests
POST /api/v1/questions
...

## Design Decisions
- Why Rails + Python split
- Why agent-based approach
- Error handling strategy
