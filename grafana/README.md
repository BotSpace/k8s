# Grafana dashboards

BotSpace biznes ko'rsatkichlari uchun Grafana dashboard'lari.

## `botspace-business.json` — Biznes ko'rsatkichlari

Manba metrikalar (Prometheus orqali, mavjud ServiceMonitor'lar bilan scrape qilinadi):

| Metrika | Manba | Tavsif |
|---|---|---|
| `botspace_user_registrations_total` | backend (`web`) | Yangi ro'yxatdan o'tishlar |
| `botspace_user_logins_total{method}` | backend (`web`) | Login'lar (Telegram) — aktivlik proksi |
| `botspace_ai_runs_total` | backend (`web`) | AI run'lari (token hisobot kelganda) |
| `botspace_ai_tokens_total{kind}` | backend (`web`) | AI sarflagan tokenlar (input/output) |
| `agent_runs_total{outcome}` | agent | Agent run'lari (completed/failed/cancelled) |
| `agent_run_duration_seconds` | agent | Run davomiyligi (p50/p95) |
| `agent_llm_input_tokens_total{category}` | agent | LLM token sarfi kategoriya bo'yicha |
| `django_http_requests_total_by_view_*` | backend | Platforma yuklamasi |

Panellar "qaysi vaqtda ko'p" savoliga `increase(...[1h])` / `rate(...[5m])`
orqali javob beradi — cho'qqilar vaqt o'qida ko'rinadi.

## O'rnatish (avtomatik provisioning)

Grafana sidecar (`kiwigrid/k8s-sidecar`) barcha namespace'larda
`grafana_dashboard: "1"` yorlig'li ConfigMap'larni kuzatadi va ichidagi JSON'ni
avtomatik import qiladi. `kustomization.yaml` shu ConfigMap'ni JSON fayldan
hosil qiladi (JSON yagona manba bo'lib qoladi):

```bash
kubectl apply -k k8s/grafana
```

Dashboard `grafana.botspace.uz` da **"BotSpace — Biznes ko'rsatkichlari"**
nomi bilan (uid: `botspace-business`) bir necha soniyada paydo bo'ladi.

## Qo'lda import

Yoki Grafana UI → Dashboards → Import → `botspace-business.json` yuklang,
datasource sifatida Prometheus'ni tanlang.
