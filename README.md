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

### 1. Default (yükləmə/stress test üçün) — dövri xəta
Heç bir xüsusi header göndərmirsinizsə, sorğular `card-status-flow` ssenarisi üzrə dövr edir:
4 uğurlu cavab + 1 xəta (500), sonra təkrar başdan — yəni ~20% xəta faizi, sabit və təkrarlana bilən şəkildə (təsadüfi deyil, amma yük testində davamlı xəta axını yaradır).

Xəta nisbətini dəyişmək üçün `mappings/1X-cycle-*.json` fayllarına state əlavə edin/silin (məs. 10 uğurludan 1 xəta üçün 9 success state + 1 error state).

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

## Scenario state-i sıfırlamaq

Testlər arası dövr sayğacını sıfırlamaq üçün (WireMock admin API):
```bash
curl -X POST http://localhost:8080/__admin/scenarios/reset
```

## Qeydlər / növbəti addımlar

- **Konkurrensiya limiti test etmək** üçün: yük testi zamanı eyni anda neçə paralel sorğu göndərdiyinizi (k6/Gatling VU sayı) sizin servisin xarici çağırışlar üçün istifadə etdiyi connection pool/thread pool ölçüsü ilə tutuşdurun.
- **Kafka/DB tərəfini** bu mock əhatə etmir — o, yalnız xarici status-dəyişmə servisini simulyasiya edir. Kafka consumer lag və DB yazı performansını ayrıca test etmək lazımdır (məs. testcontainers ilə lokal Kafka/DB qaldırıb bulk yazı ölçmək).
- Faylın böyük olduğu ssenarini simulyasiya etmək üçün sintetik CSV/Excel test faylı generasiyası lazımdırsa, deyin — ayrıca skript hazırlaya bilərəm.
