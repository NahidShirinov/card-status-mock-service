# Card Status Mock (WireMock)

Xarici "kart status dəyişmə" servisini simulyasiya edən WireMock mock-u. Fərz olunan API kontraktı:

**Request:** `POST /api/cards/status`
```json
{ "cardId": "4111111111111111", "requestedStatus": "BLOCKED" }
```

**Response (uğurlu):**
```json
{ "cardId": "4111111111111111", "status": "BLOCKED", "result": "SUCCESS", "processedAt": "2026-09-21T12:00:00.000Z" }
```

> Real API-nizin field adları fərqlidirsə, `mappings/*.json` içindəki `jsonBody` və `urlPath` dəyərlərini uyğunlaşdırın.

## İşə salmaq

```bash
docker compose up
```

Mock `http://localhost:8080/api/cards/status` ünvanında ayağa qalxır.

Docker istəmirsinizsə, standalone JAR ilə də olar:
```bash
java -jar wiremock-standalone.jar --port 8080 --global-response-templating --root-dir .
```

## Davranış rejimləri

### 1. Default (yükləmə/stress test üçün) — ~20% sabit xəta faizi
Heç bir xüsusi header göndərmirsinizsə, sorğunun nəticəsi **`cardId`-nin son rəqəminə** görə təyin olunur: son rəqəm `0` və ya `5` olan sorğular 500 (xəta) qaytarır (10-dan 2-si, yəni ~20%), qalanları uğurlu olur.

Bu yanaşma **tamamilə stateless**-dir (heç bir paylaşılan vəziyyət saxlanmır), ona görə **paralel/konkurrent sorğularda təhlükəsizdir** — əvvəlki versiyada istifadə olunan WireMock Scenario (dövr) mexanizmi paylaşılan mutable state saxladığı üçün çoxlu paralel sorğu göndəriləndə (məs. batch endpoint 20 thread ilə işlədəndə) race condition yaradıb gözlənilməz `404 Scenario does not match` xətalarına səbəb olurdu.

Xəta nisbətini dəyişmək üçün `mappings/11-random-error.json`-dakı regex-i (`.*[05]$`) uyğunlaşdırın (məs. yalnız `0`-la bitənlər üçün `.*0$` — bu, ~10% edər).

Gecikmə hər cavabda 50–300ms arasında təsadüfi seçilir (`delayDistribution`).

### 2. Funksional test üçün — məcburi nəticə (header ilə)
Konkret sorğuda nəticəni məcburi təyin etmək üçün `X-Simulate` header-i göndərin:

| Header dəyəri | Nəticə |
|---|---|
| `X-Simulate: success` | Həmişə 200 + uğurlu cavab |
| `X-Simulate: error` | Həmişə 500 + xəta cavabı |
| `X-Simulate: timeout` | 30 saniyə gözlədikdən sonra bağlantı kəsilir (client timeout simulyasiyası) |

```bash
curl -X POST http://localhost:8080/api/cards/status \
  -H "Content-Type: application/json" \
  -H "X-Simulate: timeout" \
  -d '{"cardId":"4111111111111111","requestedStatus":"BLOCKED"}'
```

Bu, sizin servisin **timeout/retry məntiqini** deterministik şəkildə test etməyə imkan verir (dövri ssenaridən asılı olmadan).

## Qeydlər / növbəti addımlar

- **Konkurrensiya limiti test etmək** üçün: yük testi zamanı eyni anda neçə paralel sorğu göndərdiyinizi (k6/Gatling VU sayı) sizin servisin xarici çağırışlar üçün istifadə etdiyi connection pool/thread pool ölçüsü ilə tutuşdurun.
- **Kafka/DB tərəfini** bu mock əhatə etmir — o, yalnız xarici status-dəyişmə servisini simulyasiya edir. Kafka consumer lag və DB yazı performansını ayrıca test etmək lazımdır (məs. testcontainers ilə lokal Kafka/DB qaldırıb bulk yazı ölçmək).
- Faylın böyük olduğu ssenarini simulyasiya etmək üçün sintetik CSV/Excel test faylı generasiyası lazımdırsa, deyin — ayrıca skript hazırlaya bilərəm.
