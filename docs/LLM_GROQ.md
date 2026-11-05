# Своя нейросеть вместо ответов Алисы (голос Алисы остаётся)

Проверено на Яндекс Станции Мини 2 с часами (YNDX-00020), root, quasar 0.369.1.46.

## Как это устроено

```
Алиса (голос, распознавание) → quasar-bridge на колонке → свой прокси → Groq API (OpenAI-совместимый)
```

- **quasar-bridge 0.7.1** (skrashevich/quasar-bridge) перехватывает фразы «Алиса, спроси нейросеть: …» и отправляет их в OpenAI-совместимый API. Ответ озвучивается голосом Алисы. Остальное (музыка, погода, таймеры) остаётся у Яндекса.
- **Groq** бесплатный и быстрый, но из РФ отвечает `403`. Поэтому запросы идут через свой маленький прокси на сервере в другой стране.
- **Прокси** заодно чинит несовместимости между тем, что шлёт bridge, и тем, что принимает Groq.

## 1. Прокси на сервере

Нужен любой Linux-сервер вне РФ с Python 3. Сервер в домашней сети за NAT подойдёт, если открыть его наружу через Tailscale Funnel (HTTPS из коробки).

```sh
sudo mkdir -p /opt/groq-proxy
# случайный секрет — это будет часть URL, чтобы прокси не нашёл кто угодно
openssl rand -hex 12 | sudo tee /opt/groq-proxy/secret
sudo nano /opt/groq-proxy/proxy.py   # код из приложения ниже
```

systemd-юнит `/etc/systemd/system/groq-proxy.service`:

```ini
[Unit]
Description=Groq proxy for quasar-bridge
After=network-online.target

[Service]
ExecStart=/usr/bin/python3 /opt/groq-proxy/proxy.py
Restart=always
DynamicUser=yes
ReadOnlyPaths=/opt/groq-proxy

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload && sudo systemctl enable --now groq-proxy
sudo tailscale funnel --bg 8787       # прокси слушает только 127.0.0.1:8787
```

Адрес для колонки: `https://<имя-сервера>.<tailnet>.ts.net/<секрет>/openai/v1`.

Логи: `journalctl -u groq-proxy -f`.

## 2. quasar-bridge на колонке

Установщик ставится по инструкции hackalice.me («Своя LLM вместо Алисы»). На вопросы про бэкенд:

- **Base URL**: адрес прокси из шага 1 (`…/<секрет>/openai/v1`)
- **API key**: свой ключ Groq (console.groq.com). Ключ хранится только на колонке, прокси его просто передаёт дальше.
- **Модель**: `openai/gpt-oss-20b` или `openai/gpt-oss-120b` (прокси всё равно подменит 20b на 120b)

Программа ставится в `/data/opt/quasar-bridge`, управление: `sh /data/opt/quasar-bridge/app/control.sh status|logs|restart`.

Использование: «Алиса, спроси нейросеть: почему небо голубое?», «Алиса, начни новый разговор».

## 3. Что прокси исправляет и почему

| Проблема | Что видно | Что делает прокси |
|---|---|---|
| Groq отклоняет OpenAI-поля | `400` на `chat_template_kwargs`, `store`, `metadata` и т.п. | Удаляет поля из списка `DROP`, `developer` → `system`, убирает `name` у сообщений |
| Модель с рассуждениями | Пустой ответ или озвучивается «размышление» | `reasoning_effort=low`, `include_reasoning=false`, `max_tokens` не меньше 2048 |
| Фраза-триггер уходит в модель | Модель отвечает «у меня нет доступа к нейросети» | Срезает «спроси нейросеть» из начала вопроса |
| Модель называет себя Алисой | Системный промпт bridge говорит «ты Алиса» | Подставляет своё имя в системные сообщения и добавляет подсказку перед последним вопросом |
| Слабая 20b | Ответы хуже | Подменяет `gpt-oss-20b` на `gpt-oss-120b` |

Имя ассистента и стиль ответа задаются в `HINT` в начале кода.

## Совместимость

Работает одновременно с AirPlay (classic), Luna и soundya — конфликтов не замечено. Прокси пишет в журнал вопрос и начало ответа; если это не нужно, удалите строки `[?] question` и `[<] reply start`.

## Приложение: код прокси (`/opt/groq-proxy/proxy.py`)

```python
import http.server, socketserver, ssl, urllib.request, urllib.error, json, re
UPSTREAM = "https://api.groq.com"
PORT = 8787
SECRET = open("/opt/groq-proxy/secret").read().strip()
PREFIX = "/" + SECRET
HOP = {"host", "connection", "keep-alive", "proxy-connection", "transfer-encoding",
       "te", "trailer", "upgrade", "content-length", "accept-encoding"}
# fields OpenAI clients send that Groq rejects with 400
DROP = {"store", "metadata", "service_tier", "prompt_cache_key", "safety_identifier",
        "logprobs", "top_logprobs", "logit_bias", "modalities", "audio", "prediction",
        "web_search_options", "verbosity", "chat_template_kwargs",
        "top_k", "min_p", "repetition_penalty", "enable_thinking"}

MODEL_MAP = {"openai/gpt-oss-20b": "openai/gpt-oss-120b"}
TRIGGER = re.compile(r"^\s*(алиса[\s,]*)?(спроси|спросить|узнай|скажи)?\s*(у\s+)?(нейросет\w*|нейронк\w*|ии|chatgpt|чат\s*gpt|gpt)[\s,:.\-]*", re.I)
HINT = ("Тебя зовут Александра, ты умная голосовая помощница внутри колонки. Ты сама и есть нейросеть: "
        "отвечай на вопрос сама и никогда не говори, что у тебя нет доступа к нейросети. "
        "Если спрашивают, кто ты или как тебя зовут, отвечай: я Александра. Говори о себе в женском роде. "
        "Отвечай по-русски, коротко (1-3 предложения), простым текстом без списков, markdown и эмодзи - ответ будет озвучен голосом.")

def clean(body):
    try:
        d = json.loads(body)
    except Exception:
        return body, []
    if not isinstance(d, dict):
        return body, []
    removed = [k for k in list(d) if k in DROP]
    for k in removed:
        d.pop(k, None)
    if d.get("n") not in (None, 1):
        d.pop("n"); removed.append("n")
    for m in d.get("messages") or []:
        if isinstance(m, dict) and "name" in m and m.get("role") != "tool":
            m.pop("name"); removed.append("messages.name")
        if isinstance(m, dict) and m.get("role") == "developer":
            m["role"] = "system"; removed.append("developer->system")
    so = d.get("stream_options")
    if isinstance(so, dict):
        for k in list(so):
            if k != "include_usage":
                so.pop(k); removed.append("stream_options." + k)
    if str(d.get("model", "")) in MODEL_MAP:
        removed.append("model %s->%s" % (d["model"], MODEL_MAP[d["model"]]))
        d["model"] = MODEL_MAP[d["model"]]
    msgs = d.get("messages")
    if isinstance(msgs, list) and msgs:
        for m in reversed(msgs):
            if isinstance(m, dict) and m.get("role") == "user" and isinstance(m.get("content"), str):
                new = TRIGGER.sub("", m["content"], count=1).strip()
                if new and new != m["content"]:
                    m["content"] = new[0].upper() + new[1:]
                    removed.append("trigger-phrase")
                break
        for m in msgs:
            if isinstance(m, dict) and m.get("role") == "system" and isinstance(m.get("content"), str):
                m["content"] = re.sub(r"Алис(а|ы|е|у|ой)", "Александра", m["content"])
        last_user = max((i for i, m in enumerate(msgs) if isinstance(m, dict) and m.get("role") == "user"), default=len(msgs))
        msgs.insert(last_user, {"role": "system", "content": HINT})
    model = str(d.get("model", ""))
    if "gpt-oss" in model:
        d["reasoning_effort"] = "low"
        d["include_reasoning"] = False
        removed.append("+reasoning low/hidden")
    elif "qwen3" in model:
        d["reasoning_effort"] = "none"
        removed.append("+reasoning none")
    for k in ("max_tokens", "max_completion_tokens"):
        v = d.get(k)
        if isinstance(v, int) and v < 2048:
            removed.append("%s %d->2048" % (k, v))
            d[k] = 2048
    return json.dumps(d).encode(), removed

class H(http.server.BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"
    def _plain(self, code, text):
        b = text.encode()
        self.send_response(code)
        self.send_header("Content-Length", str(len(b)))
        self.end_headers()
        self.wfile.write(b)
    def _go(self):
        n = int(self.headers.get("Content-Length") or 0)
        body = self.rfile.read(n) if n else None
        if not (self.path == PREFIX or self.path.startswith(PREFIX + "/")):
            return self._plain(404, "not found")
        path = self.path[len(PREFIX):] or "/"
        removed = []
        if body and self.command == "POST":
            body, removed = clean(body)
            try:
                keys = sorted(json.loads(body).keys())
            except Exception:
                keys = "?"
            print("[>] %s keys=%s removed=%s" % (path, keys, removed), flush=True)
            try:
                print("[?] question: %s" % [m.get("content") for m in json.loads(body)["messages"] if m.get("role") == "user"][-1][:200], flush=True)
            except Exception:
                pass
        req = urllib.request.Request(UPSTREAM + path, data=body, method=self.command)
        for k, v in self.headers.items():
            if k.lower() not in HOP and not k.lower().startswith(("x-forwarded", "tailscale")):
                req.add_header(k, v)
        try:
            r = urllib.request.urlopen(req, timeout=120, context=ssl.create_default_context())
        except urllib.error.HTTPError as e:
            r = e
        except Exception as e:
            print("[!] %s %s -> %s" % (self.command, path, e), flush=True)
            return self._plain(502, "proxy error: %s" % e)
        code = r.status if hasattr(r, "status") else r.code
        print("[*] %s %s -> %s" % (self.command, path, code), flush=True)
        if code >= 400:
            err = r.read()
            print("[!] error body: %s" % err[:800].decode("utf-8", "replace"), flush=True)
            self.send_response(code)
            for k, v in r.headers.items():
                if k.lower() not in HOP:
                    self.send_header(k, v)
            self.send_header("Content-Length", str(len(err)))
            self.end_headers()
            self.wfile.write(err)
            return
        self.send_response(code)
        for k, v in r.headers.items():
            if k.lower() not in HOP:
                self.send_header(k, v)
        self.send_header("Transfer-Encoding", "chunked")
        self.end_headers()
        seen = b""
        while True:
            chunk = r.read1(8192) if hasattr(r, "read1") else r.read(8192)
            if not chunk:
                break
            if self.command == "POST" and len(seen) < 3000:
                seen += chunk[:3000 - len(seen)]
            self.wfile.write(b"%x\r\n%s\r\n" % (len(chunk), chunk))
            self.wfile.flush()
        self.wfile.write(b"0\r\n\r\n")
        self.wfile.flush()
        if seen:
            print("[<] reply start: %s" % seen[:700].decode("utf-8", "replace").replace("\n", " "), flush=True)
    do_GET = do_POST = do_PUT = do_DELETE = do_PATCH = _go
    def log_message(self, *a):
        pass

class S(socketserver.ThreadingMixIn, http.server.HTTPServer):
    daemon_threads = True
    allow_reuse_address = True

if __name__ == "__main__":
    print("[+] groq proxy v8 on 127.0.0.1:%d" % PORT, flush=True)
    S(("127.0.0.1", PORT), H).serve_forever()
```
