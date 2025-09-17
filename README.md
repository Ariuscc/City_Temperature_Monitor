# City_Temperature_Monitor

City Temperature Monitoring (Kafka → Spark Structured Streaming)

Monitoring temperatur miast w czasie rzeczywistym z detekcją anomalii i zapisem wyników do Parquet.
Projekt działa lokalnie na Docker Compose: generator zdarzeń → Kafka → Spark Structured Streaming → pliki wyjściowe.

Funkcje

Producer: generuje zdarzenia JSON (city, temperature, ts_iso) i publikuje do Kafka.

Spark Structured Streaming:

Anomalie OUT_OF_RANGE (poza zakresem temperatur).

Anomalie SUDDEN_SPIKE (nagły skok/spadek względem poprzedniego pomiaru w mieście).

Metryki 1-minutowe (avg/std/count per miasto) z watermarkiem (tolerancja spóźnionych rekordów).

Output: Parquet + checkpointy (wznowienia, exactly-once na sinku).

Architektura (skrót)

Producer ──► Kafka topic "city-temperatures" ──► Spark Structured Streaming
   ├─ OUT_OF_RANGE  ─► /data/output/anomalies_out_of_range_parquet/
   ├─ SUDDEN_SPIKE  ─► /data/output/anomalies_spikes_parquet/     (foreachBatch)
   └─ Metrics 1m    ─► /data/output/metrics_1m_parquet/

(checkpointy: /data/output/chk_*/ , _spark_metadata/ itp.)


Dane wyjściowe (host)

W repozytorium, katalog montowany na hosta (np. ./data/output/) zawiera:

data/output/
├─ anomalies_out_of_range_parquet/
├─ anomalies_spikes_parquet/
├─ metrics_1m_parquet/
├─ chk_* , _spark_metadata/ , .crc  (metadane / checkpointy)

Eksport danych z kontenera na dysk

CID=$(docker compose ps -q spark)
mkdir -p export
docker cp ${CID}:/data/output ./export/


Jak to działa (w skrócie)

OUT_OF_RANGE: filtr temperature < ANOMALY_MIN OR temperature > ANOMALY_MAX.

SUDDEN_SPIKE: w foreachBatch liczony jest lag(temperature) po event_time w obrębie miasta, a następnie filtr abs(curr - prev) ≥ SPIKE_DELTA.

Okna 1-minutowe: tumbling window(event_time, WINDOW_DURATION) + withWatermark(WATERMARK_DELAY); agregacje avg/stddev_samp/count.

Zapis Parquet: tryb append, checkpointy w chk_* zapewniają wznawianie i spójność przetwarzania.
