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
# Wnioski
- najwyższe zarobki notujemy w 2022
- przy awansie można liczyć na około 36% wzrost wynagrodzenia
- patrząc na niepoprawną reprezentację danych jeżeli chodzi o typ zatrudnienia, wyniki są zakłamane i nie chcemy wyciągać wniosków z tej zmiennej
- usa są krajem w którym zarabia się najwięcej i wykonano największą liczbę obserwacji
- najwyższe zarobki mamy w pracy zdalnej i odpowiednio niższe w stacjonarnej i hybrydowej
- najwięcej zarabia się w dużych firmach
- najwięcej zarabie się na kierowniczych stanowiskach, ale najbardziej obsadzone są stanowiska data enegineer, data scientist, dana analyst i najwięcej zarabia data engineer
- najwięcej jest osób które zarabiają w przedziale 50k - 100k usd/y






