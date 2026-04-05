# WhatsApp Telugu-English Translator Bot

## Overview
A WhatsApp bot account that sits in a group chat and automatically translates messages — English to Telugu and Telugu to English — posting translations back to the group.

## Goals
- Bridge the language gap in mixed Telugu/English WhatsApp groups
- Make conversations inclusive for both Telugu and English speakers

## Key Features
- Monitors a WhatsApp group as a bot account
- Detects language of each incoming message (Telugu or English)
- Translates and posts the result back to the group
- Minimal noise — only posts when a translation is needed

## Pages

## Ideas & Notes
- Consider formatting: e.g. reply to the original message vs. standalone post
- Handle Tanglish (Telugu written in English script / transliteration)
- Option to suppress translation for messages that are already bilingual

## Resources

## Status
- [ ] Research WhatsApp bot options (WhatsApp Business API, Baileys, etc.)
- [ ] Choose translation backend (Google Translate API, DeepL, custom model)
- [ ] Handle Telugu script detection vs. transliteration
- [ ] Design message formatting / threading behavior
- [ ] Build MVP
