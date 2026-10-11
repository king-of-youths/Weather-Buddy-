# Weather Buddy AI (Version 3.1)

An AI-powered Telegram weather assistant built with **n8n**, **Google Gemini**, and **Open-Meteo**.

Users can type a city, ask a natural question ("Will I need an umbrella in Abuja tomorrow?"), tap menu buttons, or share their Telegram location. Gemini only works out *what the user is asking*. Every weather value in a reply comes from the Open-Meteo API, never from the AI.

## What it can do

| Feature | Example |
| --- | --- |
| Current weather | `Weather in Lagos` |
| Forecast | `Tomorrow in Abuja`, `Next 3 days in Kano` (up to 15 days ahead) |
| Rain check and umbrella advice | `Will it rain in Port Harcourt tomorrow?` |
| City comparison | `Is Lagos hotter than Port Harcourt?` |
| Follow-up questions | `Lagos` then `What about tomorrow?` |
| Inline menu buttons | Current Weather, Tomorrow's Forecast, Rain Forecast, Use My Location, Help |
| Shared location | Attach (paperclip), then Location, then send |
| Nigerian states | `Rivers`, `Rivers State`, or `Port Harcourt, Rivers State` |
| Ambiguous names | `Springfield` returns a list and asks which one you meant |

## Files

| File | Purpose |
| --- | --- |
| `weather-buddy-ai-v3-1.json` | Main workflow (22 nodes). Import this into n8n. |
| `weather-buddy-error-handler.json` | Small optional workflow that logs failures to your n8n logs. |
| `README.md` | This file. |

## Requirements

- Self-hosted n8n with a public **HTTPS** address (Telegram will not call plain HTTP webhooks). `WEBHOOK_URL` must be set in n8n's environment.
- A Telegram bot token from [@BotFather](https://t.me/BotFather).
- A Google Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey).
- No Open-Meteo key is needed for the free, non-commercial API.

## Setup

1. **Create credentials in n8n** (Credentials, then Create credential):
   - **Telegram API**: paste the BotFather token.
   - **Google Gemini(PaLM) Api**: paste your AI Studio key. Leave the host at its default.
   - Never put tokens or keys in Code nodes, expressions, screenshots, or chat messages.
2. **Import** `weather-buddy-ai-v3-1.json` (Workflows, then Import from file).
3. **Select credentials** on these nodes:
   - Telegram: `Telegram Trigger`, `Answer Callback Query`, `Send Text`, `Send Text With Menu`
   - Gemini: `Gemini Intent`
4. *(Optional)* Import `weather-buddy-error-handler.json`, then in the main workflow open Settings and set it as the **Error workflow**.
5. **Activate** the workflow (toggle it to Active). Memory only works in active (production) runs, not manual test runs.
6. Send `/start` to your bot.

If you used an earlier version of this workflow, deactivate it first. Two active workflows on the same bot will fight over the Telegram webhook.

## How it works

```
Telegram Trigger
  -> Normalize Update            (message type, chat id, session lookup)
  -> Route Update
       |-- callback    -> Answer Callback Query + Handle Button
       |-- location    -> Build Location Request
       |-- /start,/help-> Command Reply
       |-- bare city after a button -> Build Pending Request
       |-- free text   -> Build Gemini Request -> Gemini Intent -> Validate Intent
       '-- other       -> friendly "unsupported" reply
  -> Route Request
       |-- reply            -> message gate
       |-- has coordinates  -> Weather Request
       '-- needs geocoding  -> Geocode Request -> Geocoding API -> Resolve Locations -> Targets Ready?
  -> Weather Request -> Get Weather -> Format Answer
  -> With Menu?  -> Send Text With Menu  /  Send Text
```

Key design choices:

- **AI only interprets.** Gemini returns JSON (intent, places, date). `Validate Intent` checks it before anything else runs. If the JSON is unusable, a short message is treated as a plain city name; otherwise the user is asked to rephrase.
- **Deterministic routing.** Commands, buttons, and locations never touch the AI.
- **Valid "not found" differs from a failure.** An empty geocoding result gives a "couldn't find that place" reply. An HTTP error or malformed response gives a "couldn't retrieve the weather right now" reply and logs the detail.
- **Shared locations skip geocoding.** Coordinates are validated, then rounded to about 1 km before use.
- **Retries and timeouts.** Each API call retries up to twice with a timeout; nothing loops.

## Node reference

| Node | Responsibility |
| --- | --- |
| Normalize Update | Detects message, button, or location; extracts chat id and first name; loads the chat's session |
| Route Update | Switch on message type |
| Handle Button | Maps button data to a reply and stores a pending request |
| Command Reply | Welcome and help text |
| Build Location Request | Validates shared coordinates |
| Build Pending Request | Turns a bare city name into the request a button started |
| Build Gemini Request | Builds the Gemini prompt (today's date in Africa/Lagos, last place, user text) |
| Gemini Intent | Calls the Gemini `generateContent` endpoint |
| Validate Intent | Validates intent, dates, locations, and horizon; stores a pending request if the city is missing |
| Route Request | Splits replies, coordinate requests, and geocoding requests |
| Geocoding API / Resolve Locations | Open-Meteo geocoding, country/state matching, ambiguity detection |
| Get Weather | Open-Meteo forecast: current weather plus daily data |
| Format Answer | Builds the reply from API values only, then saves the session |
| With Menu? | Chooses whether the reply gets the inline button menu |
| Send Text / Send Text With Menu | Telegram replies |

## Memory

- Each chat has its own session, keyed by Telegram chat id, so users never see each other's context.
- It stores the **last place, last intent, and any pending request** in n8n workflow static data.
- Sessions expire after **24 hours** without activity and are capped in number.
- Static data lives in n8n's database, so it survives restarts, but it is **only saved in production (active) executions**, not when you press Execute workflow.
- Because it uses workflow static data, it is basic memory. It is not a full conversation history.

## Gemini model

The workflow calls `gemini-3.5-flash-lite` (set in the URL of `Gemini Intent`). Google lists `gemini-3.6-flash` and `gemini-3.8-flash` as alternatives. `gemini-2.5-flash` is scheduled to shut down on 16 October 2026, so do not use it. Do not add `temperature`, `top_p`, or `top_k` for Gemini 3 models. Always check Google's current model list before changing it, because names change.

## Privacy and data

- **Sent to Google Gemini:** the user's message text, today's date, and the name of the last place they asked about.
- **Sent to Open-Meteo:** city names, or coordinates (rounded to about 1 km for shared locations).
- **Stored in n8n:** the last place (including rounded coordinates for shared locations), intent, and pending request, for up to 24 hours.
- Tell your users this before using the bot with real customers. Review Google's and Open-Meteo's current terms first.

## Open-Meteo terms

The free API is for non-commercial use and requires attribution (the bot adds "Weather data provided by Open-Meteo.com"). Before offering this to clients or charging for it, check the current terms, limits, and whether a paid plan is needed.

## Test checklist

Mark each one yourself after you see the result in Telegram and the n8n execution.

- [ ] `/start` shows the welcome message and five buttons
- [ ] `/help` shows the help text
- [ ] `Lagos, Nigeria` returns current weather with real values
- [ ] `Port Harcourt, Rivers State` and `Rivers` resolve to Port Harcourt
- [ ] `London, United Kingdom` returns London, not Lagos
- [ ] `XYZABC123` returns the "couldn't find that place" reply
- [ ] `Springfield` asks which one you meant
- [ ] `Will I need an umbrella in Lagos tomorrow?` gives a qualified rain answer
- [ ] `Lagos`, then `What about tomorrow?` uses Lagos (workflow must be active)
- [ ] `Compare Lagos and Port Harcourt` shows both temperatures
- [ ] Each button works, including a city typed after a button
- [ ] A shared Telegram location returns weather
- [ ] A date more than 15 days ahead gets a friendly refusal
- [ ] A forced API failure returns the friendly error and logs the cause
- [ ] Two different Telegram accounts get separate answers and separate memory

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| Bot does not reply at all | Workflow not active, or another workflow is using the same bot. Check Executions. |
| "What about tomorrow?" asks for a city | You are running test executions, or the session expired. Activate the workflow. |
| Buttons missing | Only the replies that set the menu show buttons. Send `/start` first. If still missing, re-add the buttons in `Send Text With Menu`. |
| Every free-text message gets "trouble understanding" | Check `Gemini Intent`: credential selected, model name valid, key not restricted. |
| A state name gives the wrong place | Send the capital city instead, or add the country. |
| "Couldn't retrieve the weather" | Open `Get Weather` in the failed execution and read its output. The detail is also in the n8n logs. |

## Limitations

- Forecasts only go up to Open-Meteo's supported horizon (about 15 days); past dates are not supported.
- Comparisons support up to three places.
- Mixed requests such as "compare my location with Lagos" are not supported.
- Reverse geocoding is not used, so shared locations show as "Your shared location".
- Memory is basic and per-chat, and is lost after 24 hours of silence.

## Roadmap ideas

Multi-day weather for several cities, preferred units and language, user-saved favourite cities, scheduled daily reports, severe-weather alerts where reliable data exists, usage limits and monitoring, and persistent preferences in a database.

## Status

Generated for n8n and logic-checked with mock data. Items in the test checklist above are the real acceptance tests and should be confirmed in your own n8n installation.
