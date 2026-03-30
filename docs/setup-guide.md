# HireIQ n8n Workflow Setup Guide

## Prerequisites

- n8n cloud account or self-hosted instance
- OpenAI API key
- Google Sheets API credentials

## Import the Workflow

1. Go to n8n → Workflows → Import
2. Upload workflows/hireiq-resume-screener.json
3. Add your credentials (OpenAI, Google Sheets)
4. Activate the workflow
5. Copy the webhook URL to your frontend .env

## Workflow Overview

Webhook → Rate Limiter → PDF Extractor
→ Loop → OpenAI Scorer → Rank & Sort
→ Google Sheets → Respond to Webhook
