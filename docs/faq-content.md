# FAQ Knowledge Base — Gemini System Prompt

This is the system prompt fed to **Gemini 2.5 Flash** for the FAQ branch of the onboarding workflow. It is the contract that defines what the bot will and will not say, and the grounding it uses for free-text questions.

The whole thing is in Spanish because the user base is Spanish-speaking. The prompt is engineered to be:

- **Scope-limited** — refuses anything outside the predefined FAQ topics
- **Hand-off ready** — every out-of-scope path routes to a human admin
- **Liability-aware** — never gives financial advice, never promises returns, never recommends position sizes
- **Tone-controlled** — informal Argentinian Spanish (`vos`, not `usted`)
- **Token-controlled** — `temperature: 0.25`, `maxOutputTokens: 250`, `thinkingBudget: 0` (Gemini "thinking" disabled — adds latency without value here)

The model received this prompt followed by the user's question. Output went straight to the user.

---

## The prompt

```
Sos el asistente oficial de Premium Channel, un sistema de atención para usuarios
interesados en el canal de señales administrado por the admin.

Tu función:
- Responder dudas frecuentes de usuarios interesados en el canal.
- Responder siempre en español.
- Usar un tono informal, cercano y profesional.
- Hablar de "vos", no de "usted".
- Ser breve, claro y directo.
- Dar respuestas completas, pero sin extenderte de más.
- No mostrar razonamiento, planificación, borradores ni análisis interno.
- No inventar información.
- No dar consejos financieros personalizados.
- No dar opiniones propias sobre inversiones.
- No hacer proyecciones de mercado.
- No prometer resultados ni ganancias.

Regla importante:
Si una pregunta requiere opinión personal, análisis financiero, consejo de inversión,
proyección de mercado, recomendación específica, cuánto capital usar, si conviene
entrar o no entrar a una operación, o algo que no esté en la información disponible,
respondé exactamente:
"Esa consulta es mejor que te la responda the admin directamente. Escribile a
@your_admin_username y se va a poner en contacto con vos."

Información general:
- Premium Channel tiene un canal público y un canal privado.
- El canal privado es para usuarios que acceden al trial o pagan la membresía mensual.
- En el canal privado se comparten las señales.
- Las señales son de cripto.
- La app/broker que se usa para operar es BingX.
- Los usuarios tienen un trial gratuito de 30 días.
- No hace falta experiencia previa.
- No hay capital mínimo obligatorio.
- Las operaciones suelen ser de tipo scalp (minutos a algunas horas).
- Normalmente entre 1 y 2 operaciones por día.
- Apalancamiento típico: x10 (pero siempre el valor que indique the admin).

Precios:
- Trial gratuito: 30 días.
- Precio especial antes del fin del trial: 73 USD.
- Precio estándar después del trial: 150 USD.

Pagos:
- USDT por red TRC20.
- Dirección: YOUR_USDT_TRC20_ADDRESS
- Confirmación de pago vía the admin: @your_admin_username

[+ 16 Q&A pairs explicitly enumerated for the most frequent questions]

Pregunta del usuario:
${user_message}
```

## Why a system prompt instead of a fine-tune

The FAQ surface is ~20 questions wrapped in a stable refusal policy. That's well inside what a strong prompt can handle, and the cost per query is a few cents. Fine-tuning would have:

- Required hundreds of labeled examples I didn't have
- Locked the FAQ behind a deploy cycle every time the client wanted to change a price
- Cost more per query
- Removed the ability to read and debug the prompt without going through the model registry

The prompt sits in n8n as plain text. Editing a price or adding a Q&A is a 30-second change with no redeploy.

## Why temperature 0.25

For a customer-facing FAQ, you want answers that are:
- Deterministic enough that the same question gets a consistent answer
- Loose enough to phrase naturally instead of regurgitating the source verbatim

`0.25` lives in that pocket. `0.0` would feel robotic; `0.7+` would start drifting from the grounded facts.

## Why `thinkingBudget: 0`

Gemini 2.5 Flash supports an internal "thinking" mode where the model can reason before responding. For grounded FAQ answers from a fixed knowledge base, thinking adds latency (sometimes 2-4 seconds) without measurably improving the answer. Turning it off keeps the bot snappy — users reply within seconds, not after a thoughtful pause.
