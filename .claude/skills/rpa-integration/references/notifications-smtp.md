# SMTP / Email Notifications

Email is fire-and-forget and slow — never send it inside the request that triggers it, and never let a mail failure fail the user's action.

## Adapter

```python
import aiosmtplib
from email.message import EmailMessage

class EmailSender:
    def __init__(self, s): self.s = s   # host, port, user, password, use_tls

    async def send(self, to: list[str], subject: str, html: str, text: str | None = None):
        msg = EmailMessage()
        msg["From"], msg["To"], msg["Subject"] = self.s.mail_from, ", ".join(to), subject
        msg.set_content(text or "See HTML version.")
        msg.add_alternative(html, subtype="html")
        await aiosmtplib.send(
            msg, hostname=self.s.smtp_host, port=self.s.smtp_port,
            username=self.s.smtp_user, password=self.s.smtp_password,
            start_tls=self.s.smtp_use_tls, timeout=15,
        )
```

## Don't block the request

Sending takes hundreds of ms to seconds and can fail. Decouple it:

- **Simple:** FastAPI `BackgroundTasks` — the response returns, the mail sends after.
  ```python
  @router.post("/requests", status_code=201)
  async def create(body: RequestCreate, bg: BackgroundTasks, ...):
      row = await service.create(body)
      bg.add_task(email_sender.send, [row.requester_email],
                  "Request received", render("received.html", row))
      return row
  ```
- **Robust (match repo):** if the project has a queue/worker, enqueue the email job instead, so a mail outage doesn't lose the notification. Use the outbox pattern when the notification *must* go out.

## Reliability

- **Timeout** every send (above). A hung SMTP socket otherwise ties up a worker.
- **Retry** transient failures (connection, 4xx greylisting, 5xx) with backoff; give up after a cap and record a failed-notification state — don't crash.
- **Idempotency:** guard against double-sends on retry (e.g. a `notified_at` flag on the row, or a sent-log keyed by event id).
- A failed email must **never** roll back the domain action that triggered it. The request succeeded; the notification is best-effort.

## Templating & safety

- Render bodies from templates (Jinja2), not string concatenation; always provide a plain-text alternative alongside HTML.
- Escape user-supplied content in HTML emails.
- Keep credentials in `Settings`; never log full message bodies if they contain personal data.
