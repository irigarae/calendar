# Calendar

Small util to ease the creation of kitchen calendars for my mum.

# Moon phases

To retrieve the moon phases from [Instituto Geográfico Nacional](https://www.ign.es) we need to scrape the dates from the pdfs published there (didn't find a better source).
We download the pdfs:

```shell
for year in {2023..2035}; do
  curl -O "https://astronomia.ign.es/rknowsys-theme/images/webAstro/paginas/documentos/Agendas_Astronomicas/Agenda_astronomica_$year.pdf"
done
```

Convert them to plain text files, for example using https://pdftotext.com.
And then extract the json. Here's an example code in Clojure.

```shell
clj -Sdeps '{:deps {org.clojure/data.json {:mvn/version "2.4.0"}}}'
```

```clojure
(require '[clojure.data.json :as json]
         '[clojure.string :as str])

(defn nested-group-by [[f & fs] xs]
  (if f
    (into (sorted-map) (update-vals (group-by f xs) #(nested-group-by fs %)))
    xs))

(let [month-names ["" "Ene" "Feb" "Mar" "Abr" "May" "Jun" "Jul" "Ago" "Set" "Oct" "Nov" "Dic"]
      moons (for [year (range 2023 (inc 2035))
                  :let [content (slurp (str "~/Downloads/pdftotext/Agenda_astronomica_" year ".txt"))
                        content (re-find #"(?<=3. Fases de la Luna\nFase\n\nmes día h min)[\s\S]*(?=Todas las fechas anteriores)" content)
                        content (str/replace-first content #"\d+\n\n" "")
                        phases (re-seq #"Luna nueva|Cuarto creciente|Luna llena|Cuarto menguante" content)
                        months (re-seq #"Ene|Feb|Mar|Abr|May|Jun|Jul|Ago|Set|Oct|Nov|Dic" content)
                        days (re-seq #"(?<=\D |\n)\d+(?=\n)" content)
                        hours (re-seq #"\d{2} \d{2}" content)
                        _ (assert (apply = (map count [months days hours phases]))
                                  (mapv count [months days hours phases]))]
                  [month day hour phase] (map vector months days hours phases)
                  :let [month (.indexOf month-names month)
                        day (parse-long day)
                        [hour minute] (map parse-long (re-seq #"\d+" hour))
                        ;; some moons have minute 60 which I assume is minute 0 of the next hour
                        hour (+ hour (quot minute 60))
                        minute (rem minute 60)
                        phase (case phase
                                "Luna nueva" "new"
                                "Cuarto creciente" "crescent"
                                "Luna llena" "full"
                                "Cuarto menguante" "waning")
                        local-date (format "%04d-%02d-%02d" year month day)
                        local-date-time (format "%04d-%02d-%02dT%02d:%02d" year month day hour minute)
                        instant (-> (java.time.LocalDateTime/parse local-date-time)
                                    (.atZone (java.time.ZoneId/of "Europe/Madrid"))
                                    (.toInstant))]]
              (array-map
                :year year, :month month, :day day,
                :hour hour, :minute minute,
                :phase phase,
                :date local-date
                :datetime local-date-time
                :epoch-milli (.toEpochMilli instant)
                :instant (str instant)))
      _ (assert (= (map (juxt :year :month :day) moons)
                   (sort (map (juxt :year :month :day) moons))))]
  (->> moons
       (nested-group-by [:year :month])
       (json/pprint)))
```