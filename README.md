# Analiza zarobków w branży DataScience
- Source: plik źródłowy csv z kaggle.com 'Data science job salaries', strefakursow.pl.
- Techstack: SQLite, Power BI, DataGrip
- Raport polazuje zarobki branży Data science w zależności od różnych czynników np. doświadczenie, stanowisko, lokalizacja firmy, wielkość, data ankiety.
## Pytania badawcze i sprwadzenie zmiennych
- przyjrzymy się bardziej szczegółowo wybranym zmiennym, sprawdzimy ich unikatowe poziomy i liczbę obserwacji: work_year, exprience_leveel, employment_type, remote_ration, compatny_size
- pytania badawcze:
- jak kształtują się zarobki dla wybranych zmiennych w zleżności od roku, poziomu doświadczenia, rodzaju zatrudnienia, pracy zdalnej, wielkości firmy.
- dodatkowo sprawdzimy czy istnieje związek pomiedzy danym rokiem a pracą zdalną /2020 a 2021 czyli czas pandemii i rokiem 2022 po pandemii/
- czy istnieje związek pomiedzy pracą zdalną a wielkością firmy
- czy istnieje związek pomiędzy pracą zdalną a poziomem doświadczenia pracownika
- dzieki temu sprawdzimy jak zmieniała się płaca na przełomach lat, w zależności od wielkości firmy i poziomu doświadczenia pracownika

```
/*Wyświetlenie poziomów wybranych zmiennych*/
SELECT 'work_year' AS variable, work_year AS levels, COUNT(*) AS observations
FROM data_science_salaries
GROUP BY work_year
UNION
SELECT 'experience_level' AS variable, experience_level AS levels, COUNT(*) AS observations
FROM data_science_salaries
GROUP BY experience_level
UNION
SELECT 'employment_type' AS variable, employment_type AS levels, COUNT(*) AS observations
FROM data_science_salaries
GROUP BY employment_type
UNION
SELECT 'remote_ratio' AS variable, remote_ratio AS levels, COUNT(*) AS observations
FROM data_science_salaries
GROUP BY remote_ratio
UNION
SELECT 'company_size' AS variable, company_size AS levels, COUNT(*) AS observations
FROM data_science_salaries
GROUP BY company_size;
```
```
variable,levels,observations
company_size,Large,198
company_size,Medium,326
company_size,Small,83
employment_type,Contract,5
employment_type,Freelance,4
employment_type,Full-Time,588
employment_type,Part-Time,10
experience_level,Junior,88
experience_level,Lead,26
experience_level,Mid,213
experience_level,Senior,280
remote_ratio,0,127
remote_ratio,50,99
remote_ratio,100,381
work_year,2020,72
work_year,2021,217
work_year,2022,318
```
- sprwadźmy liczbę obserwacji dla stanowisk
```
SELECT job_title, COUNT(*) AS n
FROM data_science_salaries
GROUP BY job_title
ORDER BY n DESC;
```
```
Data Scientist,143
Data Engineer,132
Data Analyst,97
Machine Learning Engineer,41
Research Scientist,16
Data Science Manager,12
Data Architect,11
Machine Learning Scientist,8
Big Data Engineer,8
Principal Data Scientist,7
Director of Data Science,7
Data Science Consultant,7
Data Analytics Manager,7
AI Scientist,7
ML Engineer,6
Lead Data Engineer,6
Computer Vision Engineer,6
BI Data Analyst,6
Head of Data,5
Data Engineering Manager,5
Business Data Analyst,5
Applied Data Scientist,5
Head of Data Science,4
Data Analytics Engineer,4
Applied Machine Learning Scientist,4
Analytics Engineer,4
Principal Data Engineer,3
Machine Learning Infrastructure Engineer,3
Machine Learning Developer,3
Lead Data Scientist,3
Lead Data Analyst,3
Data Science Engineer,3
Computer Vision Software Engineer,3
Product Data Analyst,2
Principal Data Analyst,2
Financial Data Analyst,2
ETL Developer,2
Director of Data Engineering,2
Cloud Data Engineer,2
Staff Data Scientist,1
NLP Engineer,1
Marketing Data Analyst,1
Machine Learning Manager,1
Lead Machine Learning Engineer,1
Head of Machine Learning,1
Finance Data Analyst,1
Data Specialist,1
Data Analytics Lead,1
Big Data Architect,1
3D Computer Vision Researcher,1
```
- przeanalizujemy trzy pierwsze stanowiska
## Odpowiemy na podstaowe pytania:
- na którym stanowisku zarabia się najwięcej
- w którym kraju zarabia się najwięcej
- czy istnieje związek pomiędzy wielkością firmy a zrobkami
- czy poziom zarobków zmienia się z biegiem lat, jeżeli tak to w jaki sposób
- następnie zrobimy analizę eksploracyjną i wnioskowanie statystyczne
# Przetworzenie danych
## Oczyszczamy dane
- sprawdzamy duplikaty
```
SELECT COUNT(*) AS n_of_duplicates
FROM (SELECT work_year,
             experience_level,
             employment_type,
             job_title,
             salary,
             salary_currency,
             salary_in_usd,
             employee_residence,
             remote_ratio,
             company_location,
             company_size,
             COUNT(*) AS records
      FROM data_science_salaries
      GROUP BY work_year, experience_level, employment_type, job_title, salary, salary_currency, salary_in_usd,
               employee_residence, remote_ratio, company_location, company_size)
WHERE records > 1;
```
- listujemy duplikaty
```
SELECT *
FROM (SELECT work_year,
             experience_level,
             employment_type,
             job_title,
             salary,
             salary_currency,
             salary_in_usd,
             employee_residence,
             remote_ratio,
             company_location,
             company_size,
             COUNT(*) AS records
      FROM data_science_salaries
      GROUP BY work_year, experience_level, employment_type, job_title, salary, salary_currency, salary_in_usd,
               employee_residence, remote_ratio, company_location, company_size)
WHERE records > 1;
```
- sprawdzamy braki w danych
- błędne dane pod kątem wartości null, pustych znaków, białych znaków
- zmienimy typy danych jeżeli będzie to konieczne
- zmieniamy kodowanie dla trzech zmiennych, opiszemy pełnymi nazwami:
```
UPDATE data_science_salaries
SET experience_level = CASE
                           WHEN experience_level = 'EN' THEN 'Junior'
                           WHEN experience_level = 'MI' THEN 'Mid'
                           WHEN experience_level = 'SE' THEN 'Senior'
                           WHEN experience_level = 'EX' THEN 'Lead' END,
    employment_type  = CASE
                           WHEN employment_type = 'FT' THEN 'Full-Time'
                           WHEN employment_type = 'PT' THEN 'Part-Time'
                           WHEN employment_type = 'CT' THEN 'Contract'
                           WHEN employment_type = 'FL' THEN 'Freelance' END,
    company_size     = CASE
                           WHEN company_size = 'L' THEN 'Large'
                           WHEN company_size = 'M' THEN 'Medium'
                           WHEN company_size = 'S' THEN 'Small' END;
```
- tworzymy dodatkowo przedziały zarobków w nowej kolumnie
```
ALTER TABLE data_science_salaries
    ADD COLUMN salary_in_usd_range TEXT;
```
- i importujemy dane:
```
UPDATE data_science_salaries
SET salary_in_usd_range = CASE
                              WHEN salary_in_usd < 50000 THEN '< 50 k'
                              WHEN salary_in_usd >= 50000 AND salary_in_usd < 100000 THEN '50-100 k'
                              WHEN salary_in_usd >= 100000 AND salary_in_usd < 150000 THEN '100-150 k'
                              WHEN salary_in_usd >= 150000 AND salary_in_usd < 200000 THEN '150-200 k'
                              WHEN salary_in_usd >= 200000 AND salary_in_usd < 250000 THEN '200-250 k'
                              WHEN salary_in_usd >= 250000 AND salary_in_usd < 300000 THEN '250-300 k'
                              WHEN salary_in_usd >= 300000 AND salary_in_usd < 350000 THEN '300-350 k'
                              WHEN salary_in_usd >= 350000 AND salary_in_usd < 400000 THEN '350-400 k'
                              WHEN salary_in_usd >= 400000 AND salary_in_usd < 450000 THEN '400-450 k'
                              WHEN salary_in_usd >= 450000 AND salary_in_usd < 500000 THEN '450-500 k'
                              WHEN salary_in_usd >= 500000 THEN '>= 500 k' END;
```
# Eksploracja i analiza danych
- sprawdzamy jak kształtują się zarobki w zleżności od wybranych zmiennych z wykorzystaniem miar częstości i tendencji centralnej:
- company_size
- exprence_level
- employment_type
- remote_ratio
```
WITH variables AS (SELECT company_size AS variable, salary_in_usd AS salary FROM data_science_salaries)
SELECT variable                                                                              AS variable_levels,
       COUNT(*)                                                                              AS observations,
       ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM variables), 2)                         AS percentage,
       ROUND(AVG(salary), 0)                                                                 AS avg_salary,
       ROUND(median(salary), 0)                                                              AS med_salary,
       ROUND(SQRT(SUM(POW(salary - (SELECT salary FROM variables), 2)) / (COUNT(*) - 1)), 0) AS stdev_salary
FROM variables
GROUP BY variable;
```
- sprawdzamy jak wyglądają zarobki wg krajów zatrudnienia
```
SELECT company_location             AS country,
       COUNT(*)                     AS count,
       ROUND(AVG(salary_in_usd), 0) AS avg_salary
FROM data_science_salaries
GROUP BY country
ORDER BY avg_salary DESC;
```
- sprawdzamy jeszcze liczbność wg typu pracy /zdalna, hybrydowa, stacjonarna/

```
SELECT company_location,
       COUNT(CASE WHEN remote_ratio = 0 THEN 1 END)   AS rem_0,
       COUNT(CASE WHEN remote_ratio = 50 THEN 1 END)  AS rem_50,
       COUNT(CASE WHEN remote_ratio = 100 THEN 1 END) AS rem_100,
       ROUND(AVG(salary_in_usd), 0)                   AS avg_salary
FROM data_science_salaries
GROUP BY company_location
ORDER BY avg_salary DESC;
```

- jeszcze sprawdźmy na jakim stanowisku zarabia się najwięcej, TOP35

```
SELECT job_title, COUNT(*) AS observations, ROUND(AVG(salary_in_usd), 0) AS avg_salary
FROM data_science_salaries
GROUP BY job_title
ORDER BY avg_salary DESC
LIMIT 35;
```

- sprawdzamy, w którym przedziale zarobków jset nawięcej obserwacji

```
SELECT salary_in_usd_range, COUNT(*) AS observations
FROM data_science_salaries
GROUP BY salary_in_usd_range;
```
## Wnioski
- najwyższe zarobki notujemy w 2022
- przy awansie można liczyć na około 36% wzrost wynagrodzenia
- patrząc na niepoprawną reprezentację danych jeżeli chodzi o typ zatrudnienia, wyniki są zakłamane i nie chcemy wyciągać wniosków z tej zmiennej
- usa są krajem w którym zarabia się najwięcej i wykonano największą liczbę obserwacji
- najwyższe zarobki mamy w pracy zdalnej i odpowiednio niższe w stacjonarnej i hybrydowej
- najwięcej zarabia się w dużych firmach
- najwięcej zarabie się na kierowniczych stanowiskach, ale najbardziej obsadzone są stanowiska data enegineer, data scientist, dana analyst i najwięcej zarabia data engineer
- najwięcej jest osób które zarabiają w przedziale 50k - 100k usd/y

## Wnioskowanie statystyczne
- zbadamy relacje pomiędzy zmiennymi jakościowymi
- następnie przejdziemy od badania zmiennych ilościowych

- zestawimy w pierwszej kolejności zmienne: work_year z remote_ratio, company_size, exprience_level, job_title
- przygotujemy tablę krzyżową, wyliczymy wartości oczekiwane, przeliczymy test niezależności chi2, dla porównania 3x3
- przeliczymy stopień swobody
- sprawdzimy istotność statystyczną testu chi2 z wartościami krtytycznymi

  ```
  WITH contingency_table AS (SELECT (r1c1 + r1c2 + r1c3)                                           AS r1t,
                                  (r2c1 + r2c2 + r2c3)                                           AS r2t,
                                  (r3c1 + r3c2 + r3c3)                                           AS r3t,
                                  (r1c1 + r2c1 + r3c1)                                           AS c1t,
                                  (r1c2 + r2c2 + r3c2)                                           AS c2t,
                                  (r1c3 + r2c3 + r3c3)                                           AS c3t,
                                  (r1c1 + r1c2 + r1c3 + r2c1 + r2c2 + r2c3 + r3c1 + r3c2 + r3c3) AS rct,
                                  r1c1,
                                  r1c2,
                                  r1c3,
                                  r2c1,
                                  r2c2,
                                  r2c3,
                                  r3c1,
                                  r3c2,
                                  r3c3
                           FROM (SELECT SUM(CASE WHEN work_year = 2020 AND job_title = 'Data Engineer' THEN 1.0 END)  AS r1c1,
                                        SUM(CASE WHEN work_year = 2020 AND job_title = 'Data Scientist' THEN 1.0 END) AS r1c2,
                                        SUM(CASE WHEN work_year = 2020 AND job_title = 'Data Analyst' THEN 1.0 END)   AS r1c3,
                                        SUM(CASE WHEN work_year = 2021 AND job_title = 'Data Engineer' THEN 1.0 END)  AS r2c1,
                                        SUM(CASE WHEN work_year = 2021 AND job_title = 'Data Scientist' THEN 1.0 END) AS r2c2,
                                        SUM(CASE WHEN work_year = 2021 AND job_title = 'Data Analyst' THEN 1.0 END)   AS r2c3,
                                        SUM(CASE WHEN work_year = 2022 AND job_title = 'Data Engineer' THEN 1.0 END)  AS r3c1,
                                        SUM(CASE WHEN work_year = 2022 AND job_title = 'Data Scientist' THEN 1.0 END) AS r3c2,
                                        SUM(CASE WHEN work_year = 2022 AND job_title = 'Data Analyst' THEN 1.0 END)   AS r3c3
                                 FROM data_science_salaries))
```

SELECT (POW(r1c1 - ef_r1c1, 2) / ef_r1c1) +
       (POW(r1c2 - ef_r1c2, 2) / ef_r1c2) +
       (POW(r1c3 - ef_r1c3, 2) / ef_r1c3) +
       (POW(r2c1 - ef_r2c1, 2) / ef_r2c1) +
       (POW(r2c2 - ef_r2c2, 2) / ef_r2c2) +
       (POW(r2c3 - ef_r2c3, 2) / ef_r2c3) +
       (POW(r3c1 - ef_r3c1, 2) / ef_r3c1) +
       (POW(r3c2 - ef_r3c2, 2) / ef_r3c2) +
       (POW(r3c3 - ef_r3c3, 2) / ef_r3c3) AS chi2,
       r1c1 - ef_r1c1,
       r1c2 - ef_r1c2,
       r1c3 - ef_r1c3,
       r2c1 - ef_r2c1,
       r2c2 - ef_r2c2,
       r2c3 - ef_r2c3,
       r3c1 - ef_r3c1,
       r3c2 - ef_r3c2,
       r3c3 - ef_r3c3
FROM (SELECT (r1t * c1t) / rct AS ef_r1c1,
             (r1t * c2t) / rct AS ef_r1c2,
             (r1t * c3t) / rct AS ef_r1c3,
             (r2t * c1t) / rct AS ef_r2c1,
             (r2t * c2t) / rct AS ef_r2c2,
             (r2t * c3t) / rct AS ef_r2c3,
             (r3t * c1t) / rct AS ef_r3c1,
             (r3t * c2t) / rct AS ef_r3c2,
             (r3t * c3t) / rct AS ef_r3c3
      FROM contingency_table),
     contingency_table;
```
- sprawdzamy poziom istotności dla alfa = 0,05 i mniejszych
- istnieje istotna zależność pomiędzy danym rokiem a typem pracy
- istnieje istotna zależność pomiędzy danym rokiem a wielkością firmy
- istnieje istotna zależność pomiędzy danym rokiem a poziomeme doświadczenia
- istnieje istotna zależność pomiędzy danym rokiem a stanowiskiem
## Sprawdzamy korelację pomiędzy zarobkami i danym rokiem
- pytanie czy w kolejnych latach zarobki się zmieniały, a jeżeli tak to jak
- obliczymmy statystykę Pearsona dla work_year i salary_in_usd

```
WITH statistics AS (SELECT avg_work_year,
                           avg_salary,
                           SQRT(SUM(POW(work_year - avg_work_year, 2)))  AS sqrt_sum_work_year,
                           SQRT(SUM(POW(salary_in_usd - avg_salary, 2))) AS sqrt_sum_salary
                    FROM (SELECT AVG(work_year) AS avg_work_year, AVG(salary_in_usd) AS avg_salary
                          FROM data_science_salaries),
                         data_science_salaries)
SELECT r_pearson * SQRT(COUNT(*) - 2) / SQRT(1 - POW(r_pearson, 2)) AS t_value,
       r_pearson
FROM (SELECT SUM((work_year - avg_work_year) * (salary_in_usd - avg_salary)) /
             (sqrt_sum_work_year * sqrt_sum_salary) AS r_pearson
      FROM statistics,
           data_science_salaries),
     data_science_salaries;
```
- istnieje słaba dodatnia zależność pomiędzy latami i zarobkami zatem badania zarabiali coraz więcej
- sprwadzamy czy korelacja jest istotna statystycznie
- wartość testu jest jest wyższa od poziomu alfa 0,05. Także dla pozostałych wartości alfa.
- zatem korelacja jest istotna statystycznie
- i jeszcze sprawadzimy jak zmieniały się zarobki pomiędzy osobami pracującymi zdalnie i stacjonarnie na przestrzeni 3 lat
- wykorzstamy test tWelcha
```
WITH statistics AS (SELECT SUM(POW((CASE WHEN remote_ratio = 100 THEN salary_in_usd END) - avg_salary_remote, 2)) /
                           (n_remote - 1)     AS variance_salary_remote,
                           SUM(POW((CASE WHEN remote_ratio = 0 THEN salary_in_usd END) - avg_salary_stationary, 2)) /
                           (n_stationary - 1) AS variance_salary_stationary,
                           avg_salary_remote,
                           avg_salary_stationary,
                           n_remote,
                           n_stationary
                    FROM (SELECT AVG(CASE WHEN remote_ratio = 100 THEN salary_in_usd END) AS avg_salary_remote,
                                 AVG(CASE WHEN remote_ratio = 0 THEN salary_in_usd END)   AS avg_salary_stationary,
                                 COUNT(CASE WHEN remote_ratio = 100 THEN 1.0 END)         AS n_remote,
                                 COUNT(CASE WHEN remote_ratio = 0 THEN 1.0 END)           AS n_stationary
                          FROM data_science_salaries
                          WHERE work_year == 2022),
                         data_science_salaries
                    WHERE work_year == 2022)
SELECT (avg_salary_remote - avg_salary_stationary) /
       SQRT((variance_salary_remote / n_remote) + (variance_salary_stationary / n_stationary)) AS welch_t_test,
       POW((variance_salary_remote / n_remote) + (variance_salary_stationary / n_stationary), 2) /
       ((POW(variance_salary_remote / n_remote, 2) / (n_remote - 1)) +
        (POW(variance_salary_stationary / n_stationary, 2) / (n_stationary - 1)))              AS df
FROM statistics;
```
- mamy stopnie swobody df
- sprawdzamy w tabeli
- nie możemy uznać testu t za istotny statystycznie, zatem nie możemy uznać że istnieją istotne różnice pomiędzy zarobkami dla pracy stancjonarnej i dla pracy zdalnej dla 2020, 2021, ale w 2022 istnieją statystycznie istotne różnice dla pracy zdalnej i pracy stacjonarnej
  
## Podsumowanie

- w brażny analiz danych zarabia się co raz więcej
- najbardziej popularnym stanowiskiem jest data scientist
- najwięcej zarobi data engineer
- najlepiej zarabiamy w dużych firmach
- istnieją istotne różnice w zarobkach pomiędzy pracą zdalną i stacjonarną tylko w 2022 roku
  





